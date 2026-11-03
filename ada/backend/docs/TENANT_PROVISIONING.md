# Tenant Provisioning & Destroy

## Scripts

| Script | Dónde corre | Rol |
|---|---|---|
| `provision-tenant.sh` | Host (Ionos) | Orquesta Keycloak + DB — punto de entrada para producción/staging |
| `provision-tenant-db.sh` | Contenedor Patroni | Operaciones psql (llamado por los anteriores) |
| `provision-tenant-local.sh` | Máquina local del dev | Provisiona tenant en postgres local + Ionos Keycloak vía HTTP (sin Docker ni SSH) |
| `destroy-tenant.sh` | Host (Ionos) | Destruye un tenant completo — irreversible |
| `destroy-tenant-db.sh` | Contenedor Patroni | DROP DB + limpieza ada_master (llamado por el anterior) |

Todos en: `infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles/`

> El repositorio se llama `infraestructure-as-code` (así está en GitHub y así está en Ionos). En la máquina local el
> clon vive en una carpeta llamada `infraestructure-ascode`: es el mismo repo, solo cambia el nombre de la carpeta.

**Credenciales requeridas del operador: ninguna.** La password de Keycloak se lee directamente del Docker secret montado en el contenedor Keycloak (`/run/secrets/keycloak-admin-password-staging`) mediante `docker exec cat`. Solo se necesita acceso a Docker.

---

## Provisioning local (dev)

Usar cuando se necesita un tenant en la máquina local del dev para probar el flujo de autenticación sin desplegar a Ionos. No requiere Docker Swarm ni acceso SSH.

**Qué hace:**
- Llama a Ionos Keycloak (`sso.sotobotero.com`) por HTTP para crear/actualizar el usuario con `UPDATE_PASSWORD` — el mismo Keycloak al que ya apunta ADA local en `.env.debug.local`, sin ningún contenedor local
- Clona `default_tenant` → nuevo DB en postgres local (`localhost:5432`) vía `CREATE DATABASE … TEMPLATE` (sin pg_dump, evita incompatibilidades de versión)
- Registra el tenant en `ada_master` local: `tenant_registry`, `users` (con `keycloak_id`), `memberships`, `user_system`, `company_profile`

**Prerequisito** — asegurarse de que `.vscode/.env.debug.local` tenga:
```
ADA_KEYCLOAK_LOGIN_ENABLED=true   # ADA autentica vía Keycloak ROPC
```

**Uso:**
```bash
cd infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles

bash provision-tenant-local.sh \
  db:default_tenant public \
  <NIT> "<Nombre Legal>" [email] \
  --clean-data
```

**Ejemplo:**
```bash
bash provision-tenant-local.sh db:default_tenant public 999888001 "Astral Studio" --clean-data
# Login: 999888001 / 999888001 → muestra panel de cambio forzado
```

---

## Flujo de provisioning (Ionos)

```
provision-tenant.sh (host)
  │
  ├─ [1/3] docker exec keycloak → cat secret → obtiene KEYCLOAK_ADMIN_PASS
  │
  ├─ [2/3] curl → Keycloak API (upsert usuario NIT) → obtiene KEYCLOAK_UUID
  │
  └─ [3/3] docker exec patroni → provision-tenant-db.sh
               ├─ Crea DB (skip si ya existe — idempotente)
               ├─ Clona desde la base indicada en $1 (skip si ya existe)
               ├─ Concede privilegios a appuser en todos los schemas
               ├─ Registra en ada_master.tenant_registry
               ├─ Upsert admin en ada_master.users (keycloak_id, login=NIT, SHA1)
               ├─ Memberships: NIT + root → tenant (role='owner')
               ├─ (opcional --clean-data) Limpia datos de negocio
               ├─ INSERT person + user_system (NIT admin, copiado de root id=1)
               ├─ UPDATE company_profile (name, NIT, email; limpia campos del clone)
               ├─ (con --clean-data) borra los períodos contables del origen y crea los del año en curso
               ├─ (opcional --country=XX) Trunca nomenclature, resembra geografía del país
               │    y reemplaza plan de cuentas e impuestos por los del país (accounting/XX.sql)
               │    y actualiza company_profile.country al país indicado
               └─ (opcional --seed-consumidor-final) Inserta Consumidor Final DIAN 222222222222
                    con geografía resuelta automáticamente si --country=CO fue especificado
```

## Flujo de destroy

```
destroy-tenant.sh (host)
  │
  ├─ Busca NIT admin en ada_master (sin parámetro extra)
  ├─ Muestra resumen + pide confirmación (escribir slug exacto)
  │
  ├─ [1/3] curl → Keycloak API → DELETE usuario NIT
  │
  └─ [2/3] docker exec patroni → destroy-tenant-db.sh
               ├─ DELETE memberships del tenant
               ├─ DELETE usuario de ada_master.users (si no tiene otros tenants)
               ├─ DELETE de tenant_registry
               ├─ pg_terminate_backend (cierra conexiones activas del pool)
               └─ DROP DATABASE
```

---

## Prerequisitos

- Acceso SSH al servidor Ionos: `ssh ionos`
- Acceso a Docker en el servidor (para `docker exec` / `docker ps`)
- Contenedores `staging-infra_keycloak.*` y `staging-infra_patroni.*` corriendo
- **Email de bienvenida (opcional):** crear `docker-compose/config/.env.smtp` a partir de `.env.smtp.example` con las credenciales SMTP reales. Si el archivo no existe el provisioning igual completa — solo omite el email.

---

## Provisioning

```bash
# Desde el servidor Ionos — sin variables de entorno
cd /home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles

bash provision-tenant.sh \
  db:qa public \
  <NIT> \
  "Nombre Legal de la Empresa S.A.S" \
  info@empresa.co \
  --clean-data \
  --country=CO \
  --seed-consumidor-final
```

El `tenant_slug` se deriva **automáticamente**:
- Primeras 2 letras del nombre de empresa (minúsculas, solo alfa) + NIT normalizado (sin espacios, puntos ni dígito de verificación)
- Ejemplo: `"Academia Fitness Demo S.A.S"` + `"900987654-3"` → slug `ac900987654`

El NIT se puede ingresar normalizado o en formato colombiano estándar (`123.456.789-0`) — el script lo limpia de todas formas.

### Parámetros

| Posición | Nombre | Ejemplo | Descripción |
|---|---|---|---|
| `$1` | `source` | `db:qa` | DB fuente a clonar (un tenant vivo con el esquema al día; en Ionos no hay `default_tenant`, se usa `qa`), o ruta a un dump. La clona con `pg_dump \| pg_restore` |
| `$2` | `schemas` | `public` | Legacy; el script concede sobre todos los schemas igualmente |
| `$3` | `nit` | `900987654-3` | NIT de la empresa; normalizado automáticamente |
| `$4` | `company_name` | `"Academia Fitness S.A.S"` | Nombre legal (quoted si tiene espacios) |
| `$5` | `email` | `info@empresa.co` | *(opcional)* Email admin; default `admin@<slug>.local`. **Keycloak no admite emails repetidos** (`duplicateEmailsAllowed=false`): si ya lo usa otro usuario falla; un alias `usuario+algo@gmail.com` llega al mismo buzón y vale |
| `--clean-data` | *(flag)* | `--clean-data` | Limpia datos de negocio del clone; mantiene solo catálogos e id=1 |
| `--country=XX` | *(flag)* | `--country=CO` | Trunca `public.nomenclature` y la resembra con el seed del país indicado (código ISO). Actualiza además `company_profile.country`. Aplica además el plan de cuentas y los impuestos del primer país (`dbfiles/accounting/<ISO>-<PLAN>.sql`; ver más abajo). Sin este flag se conserva todo lo clonado. Disponible: `CO` (Colombia: nomenclatura DIVIPOLA, PUC e IVA colombiano) y `ES` (España: nomenclatura, PGC e IVA español). Un país sin archivo conserva lo del tenant de origen. |
| `--public-booking=<host>` | *(flag)* | `--public-booking=reservas.ejemplo.com` | Abre la reserva anónima del tenant en ese host: inserta `ada_master.tenant_domains` y pone `company_profile.public_booking_enabled = true`. Sin el flag el tenant no expone nada público. El host debe llegar a ADA por un `server` de nginx. Ver [API pública](PUBLIC_API_AND_ACCESS_CONTROL.md). |
| `--max-users=N` | *(flag)* | `--max-users=5` | Licencia: usuarios nominales permitidos (no cuentan `root` ni el rol `client`). Por defecto 3. |
| `--access-months=N` | *(flag)* | `--access-months=12` | Licencia: meses de acceso desde hoy (`access_until`). Por defecto 12. |
| `--seed-consumidor-final` | *(flag)* | `--seed-consumidor-final` | Inserta el cliente "Consumidor Final" (NIT DIAN 222222222222) requerido para facturación electrónica colombiana (Resolución 000042/2020). La geografía (Colombia/Bogotá D.C.) se resuelve automáticamente si `--country=CO` fue especificado. Idempotente — no falla si ya existe. |

### Activar el marketplace a un tenant que ya existe

Los seeds están en `db/seed/provisioning/marketplace/` y son idempotentes. En orden:

```bash
# 1. roles y permisos que abren el portal de proveedora y la plataforma
psql -d <bd del tenant> -v ON_ERROR_STOP=1 -f db/seed/provisioning/modules/marketplace.sql
# 2. sectores y categorías del catálogo público
psql -d <bd del tenant> -v ON_ERROR_STOP=1 -f db/seed/provisioning/marketplace/sectors.sql
# 3. planes de proveedora (solo si el tenant no nació de una solicitud)
psql -d ada_master -v tenant=<slug> -f db/seed/provisioning/provider-plans-default.sql
# 4. opcional: campaña de bienvenida de referidos
psql -d <bd del tenant> -v ON_ERROR_STOP=1 -f db/seed/provisioning/referrals/campaign.sql
```

Qué abre cada permiso y qué se siembra en cada tipo de tenant: [`db/seed/provisioning/README.md`](../db/seed/provisioning/README.md).

### Idempotencia

El script es re-ejecutable sin DROP previo. Si la DB ya existe, salta el CREATE y el clone y solo actualiza registry, users, memberships, user_system y company_profile.

---

## Operador público compartido (ejemplo real, 2026-10-21)

Tenant de pruebas que atiende la reserva anónima del host `stg-toki.sotobotero.com` (marketplace compartido en staging; `toki.sotobotero.com` queda reservado para producción; hasta el 2026-10-27 fue `arcadia.sotobotero.com`):

```bash
bash provision-tenant.sh db:qa public 9850777 "Sotobotero Marketplace" \
  sotobotero+marketplace@gmail.com --clean-data --country=CO --public-booking=stg-toki.sotobotero.com
```

Slug `so9850777`. `tenant_domains.host` es clave primaria: antes de asociar un host a otro tenant hay que borrar su fila
(`DELETE FROM ada_master.tenant_domains WHERE host = '...'`), si no el provisioning falla.

## Destroy

```bash
# Desde el servidor Ionos
cd /home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles

bash destroy-tenant.sh <tenant_slug>
```

El script busca el NIT del admin en `ada_master`, muestra el resumen completo y exige escribir el slug exacto para confirmar antes de ejecutar cualquier acción destructiva.

```
╔══════════════════════════════════════════════════════════════╗
║  !! DESTROY TENANT — THIS IS IRREVERSIBLE !!
║  Tenant slug : ac900987654
║  Company     : Academia Fitness Demo S.A.S
║  Admin login : 900987654
╚══════════════════════════════════════════════════════════════╝

Type the tenant slug to confirm destruction: _
```

---

## Siembra del provisioning: qué hay, de qué país y quién lo usa

El provisioning no siembra "datos de negocio": limpia el clon y le pone lo que el tenant necesita para operar en su país.
Estos archivos **no son migraciones** y viven todos aquí, en `db/seed/provisioning/`, porque siembran el modelo de datos que
este repo define. El script de provisioning (que corre en el host, repo de infraestructura) **no los tiene al lado**: hay que
decirle dónde están con `--seed-dir=<ruta>` o `ADA_SEED_DIR`; si no los encuentra, se detiene antes de tocar ninguna base.

```bash
bash provision-tenant.sh db:qa public 900987654 "Empresa S.A.S" --clean-data --country=CO \
  --seed-dir=/home/ionosadmin/ADA_ENTERPISE_CORE/db/seed/provisioning
```

En Ionos: clonar ADA junto al repo de infraestructura (o subir `db/seed/provisioning/` en el despliegue).

| Pieza | Qué es | Depende del país | Quién la usa | Dónde vive |
|---|---|---|---|---|
| `db/seed/provisioning/clean-tenant-data.sql` | Borra los datos de negocio del clon y deja catálogos e `id=1` | No | provision, con `--clean-data` | este repo |
| `db/seed/provisioning/nomenclature/generic.sql` | Tipos de vía y países: la base sobre la que se apoya cualquier geografía | No | provision, con `--country` | este repo |
| `db/seed/provisioning/nomenclature/CO.sql` | Geografía DIVIPOLA (Colombia) | Sí, `CO` | provision, `--country=CO` | este repo |
| `db/seed/provisioning/nomenclature/ES.sql` | Geografía de España | Sí, `ES` | provision, `--country=ES` | este repo |
| `db/seed/provisioning/accounting/CO-PUC.sql` | Plan Único de Cuentas colombiano (2.520 cuentas, 66 activas), impuestos y métodos de pago; reemplaza el plan clonado en una transacción y **se niega si el tenant ya tiene movimientos contables** | Sí, `CO` | provision, con `--country` (primer país) | este repo |
| `db/seed/provisioning/accounting/ES-PGC.sql` | Plan General Contable español (919 cuentas, 20 activas), impuestos y métodos de pago; misma regla | Sí, `ES` | provision, idem | este repo |
| Consumidor Final DIAN (NIT 222222222222) | Cliente obligatorio para facturar electrónicamente en Colombia; la geografía se resuelve sola desde la nomenclatura | Sí, `CO` | provision, con `--seed-consumidor-final`; es **SQL dentro de `provision-tenant-db.sh`** (infra), idempotente por documento | infra (script) |
| `db/seed/provisioning/hosts.sql` | Declara un host y su zona (staff o public); es el registro completo desde la 192: sin fila el host no existe para la app | No | a mano por entorno, y provisioning al crear el host | este repo |
| `db/seed/provisioning/tenant-domain.sql` | Registra el host del tenant en `ada_master.tenant_domains` (sin él no hay zona pública: catálogo, reservas, alta de proveedora) | No | a mano (`psql -d ada_master -v tenant=<slug> -v host=<host>`); el provisioning lo hace con `--public-booking=<host>` | este repo |
| `db/seed/provisioning/platform-operator.sql` | Marca en `ada_master.users` quién es operador de plataforma (segunda llave del admin de plataforma, junto al permiso del tenant) | No | a mano (`psql -d ada_master -v login=<login>`) | este repo |
| `db/seed/provisioning/provider-plans-default.sql` | Planes de proveedora para un marketplace que no nació de una solicitud | No, pero solo aplica a un marketplace | a mano (`psql -d ada_master -v tenant=<slug>`) | este repo |
| `db/seed/demo/*.sql` | Escenarios de prueba (lotes, métodos de pago) | No | a mano en una base de pruebas | este repo |

Reglas que se desprenden de ahí:

- **País = datos, no esquema.** Añadir un país es añadir dos archivos (`nomenclature/<ISO>.sql`,
  `accounting/<ISO>-<PLAN>.sql`) y un flag, sin tocar el esquema de la base.
- **El plan de cuentas por país vive solo en `dbfiles/accounting/<ISO>-<PLAN>.sql`.** No hay copia en este repo: la versión
  manual antigua que estaba en `db/seed/` quedó fuera porque no reemplazaba impuestos ni métodos de pago, y tener dos
  versiones del mismo catálogo solo invita a correr la incompleta. Se aplica con `--country` sobre un tenant **sin asientos**;
  si el tenant ya facturó, el archivo se niega y hay que decidir a mano (el script no lo hace).
- **`--country` acepta varios (`CO,ES`) para la geografía, pero el plan de cuentas es solo del primero.** Un tenant tiene un
  plan de cuentas, no dos.
- Los seeds de este repo (`db/seed/`) son manuales y no los corre ningún script: `provisioning/` para lo que necesita un
  negocio al habilitarse y `demo/` para escenarios de prueba. Todo lo que depende del país (nomenclatura y plan de cuentas)
  está en el repo de infraestructura, junto al script que lo aplica.

## Qué crea el provisioning

### Keycloak (realm `arcadia`)

| Campo | Valor |
|---|---|
| `username` | `<NIT>` — un usuario por empresa, único en el realm |
| `email` | email del parámetro (o `admin@<slug>.local`) |
| `password` | `<NIT>` en texto plano — cambiar tras primer login |
| `temporary` | `false` |

Re-provisioning (DB ya existe): solo actualiza el password de Keycloak.

### ada_master.tenant_registry

```
slug         = <prefix2letras><nit_normalizado>
db_name      = <slug>
company_name = <company_name>
mode         = 'business'
status       = 'active'
max_users    = 3 (o --max-users)
access_until = now() + 12 meses (o --access-months)
```

### ada_master.users + memberships

```
login         = '<NIT>'
email         = <email>
keycloak_id   = <UUID real de Keycloak>
password_hash = SHA1(<NIT>)
```

| login | tenant_slug | role | status |
|---|---|---|---|
| `<NIT>` | `<slug>` | `owner` | `active` |
| `root` | `<slug>` | `owner` | `active` |

### user_system en la DB de negocio

Con `--clean-data`, el orden es: clone → clean → insert NIT admin → update company_profile.

| id | login | origen |
|---|---|---|
| 1 | `root` | clone de `default_tenant` (siempre presente) |
| 2 | `<NIT>` | insertado por el script — role/status copiados de root (id=1) |

> **Regla crítica:** `ada_master.users.login = NIT` debe coincidir con `user_system.login = NIT` para que `LoginBean` resuelva la sesión correctamente.

### Períodos contables, plan de cuentas e impuestos en la DB de negocio

| Qué | Cuándo | Resultado |
|---|---|---|
| Períodos | `--clean-data` | Borra los del tenant de origen y crea los del **año en que se provisiona** (`AAAA00` inicial, `AAAA01`-`AAAA12`, `AAAA13` cierre anual), abiertos. Los del año siguiente hay que crearlos antes de enero: sin un período abierto ADA deja las ventas y compras como pendientes de contabilizar |
| Plan de cuentas | `--country=CO` / `ES` | `accounting/CO-PUC.sql`: PUC colombiano (Decreto 2650 de 1993), 2.520 cuentas, 66 activas (base de comercio). `accounting/ES-PGC.sql`: el PGC español tal como lo tiene la plantilla QA (919 cuentas). Ambos reemplazan el plan del origen y repuntan las columnas que referencian el catálogo (medios de pago, bancos, tipos de inventario, impuestos, separados). Una transacción; se niegan si el tenant ya tiene asientos |
| Impuestos | `--country=CO` / `ES` | CO: IVA generado 19% y 5% (ventas, `240805`) e IVA descontable 19% y 5% (compras, `240810`). ES: IVA repercutido 10/21, soportado 10/21 y recargo de equivalencia |
| Geografía | `--country=CO` / `ES` | `nomenclature/CO.sql` (33 departamentos, 1.122 municipios) o `ES.sql` (52 provincias, 140 comarcas, 8.198 municipios; sacada de la base de QA, donde vivía) |

Cada país es un archivo `accounting/<ISO>-<PLAN>.sql`; sin archivo se conserva el plan clonado. Por validar con el contador de cada cliente: subcuentas de IVA, compras como `620505` (inventario periódico) y la línea de ventas `413524`.

### company_profile en la DB de negocio

| Campo | Valor |
|---|---|
| `name` | `<company_name>` |
| `document_number` | `<NIT>` |
| `email` | `<email>` |
| `country` | Nomenclature del país (`--country=CO` → Colombia) — solo si se pasa el flag |
| `addres`, `phone`, `bank_account_number`… | `''` — el admin los rellena tras primer login |
| Resto (locale, timezone, impuestos…) | Sin tocar — heredan del clone |

#### Agente de impresión (`printer_agent_url` / `printer_agent_secret`)

El agente de impresión es un proceso local que corre en el PC donde están conectadas las impresoras USB. El tenant lo configura desde **Perfil de empresa → Impresoras**.

| Campo | Qué poner |
|---|---|
| `printer_agent_url` | URL del agente: `http://<IP-del-PC-con-impresora>:5000` |
| `printer_agent_secret` | Token configurado en `print-agent.env` (`AGENT_SECRET`) |

> **No usar `localhost`** — ADA corre en el servidor y `localhost` apuntaría al propio servidor, no al PC del cajero. Usar la IP local del PC donde está conectada la impresora (ver `hostname -I` en Linux o `ipconfig` en Windows).

---

## Roles: quién puede qué

Cuatro capas, de fuera hacia adentro. Ninguna sustituye a otra: cada decisión se toma en la capa de su alcance.

> **Estado (2026-11-03):** `hosts.zona`, `memberships.zona` y `user_auth_identity` existen en local y staging (migración 192) y
> `auth-options` ya resuelve el realm por zona en staging. `myWorkspace` (las dos autoridades de abajo) está probado en local y
> **pendiente de despliegue**; hasta que entre, el menú se decide con el permiso solo.

| Capa | Dónde vive | Qué decide |
|---|---|---|
| **Realm** (Keycloak) | los dos realms: **staff** y **abierto** del marketplace | quién puede autenticarse en cada puerta. Sale de la **zona del host** (`ada_master.hosts.zona`), nunca de un valor por host |
| **Plataforma** | `ada_master.users.platform_operator` | aprobar el **alta de un tenant completo** (onboarding) |
| **Membresía** | `ada_master.memberships (tenant_slug, role, zona)` | a qué tenants entra una persona, **por qué puerta** y con qué rol del tenant |
| **Rol + permisos del tenant** | `<base del tenant>.role` (con `bypass`), `permission`, `role_permision`, `user_permission_grant` | qué pantallas y operaciones dentro de ese tenant |

⚠️ El `role` de la **membresía** no autoriza por sí mismo: describe la puerta y el origen. La autorización dentro del tenant es
el rol de **su** base de datos.

### Roles de membresía (`ada_master.memberships.role`)

| `role` | `zona` | Quién es | Qué le permite |
|---|---|---|---|
| `owner` | `staff` | la dueña del negocio | entrar al tenant como su administrador (con el rol que tenga en la base del tenant) |
| `staff` | `staff` | equipo de plataforma / soporte | operar; aprobar altas solo si además tiene `platform_operator` |
| `provider` | `public` | proveedora del marketplace | entrar por el host público con su cuenta del realm abierto |
| `client` | `public` | cliente del marketplace | entrar por el host público con su cuenta del realm abierto |

La fila de `platform_operator` **no** es una membresía: vive en `users` y solo aplica al onboarding de tenants completos.

### Roles del tenant (semilla de la base de negocio)

| Rol | `bypass` | Permisos | Zona | Qué abre |
|---|---|---|---|---|
| `root` | **sí** | 67 | staff | todo: el permiso se salta las comprobaciones (plataforma/desarrollo) |
| `Administrador del sistema` | no | 67 | staff | todo el catálogo del tenant: es el administrador del negocio |
| `CONTABILIDAD` | no | 45 | staff | contabilidad y documentos contables |
| `PRODUCCIÓN` | no | 45 | staff | operación de inventario y productos |
| `VENTAS` | no | 5 | staff | facturación y ventas |
| `Cajero` | no | 4 | staff | caja |
| `COMERCIAL` | no | 3 | staff | clientes y contratos |
| `usuario basico` | no | 1 | staff | mínimo (cambio de contraseña) |
| `marketplace_provider` | no | 2 | public | portal de proveedora: sus servicios, su agenda, sus fotos |
| `marketplace_client` | no | 1 | public | cambio de contraseña; el resto de vistas son públicas |

Los permisos por módulo (inventario, facturación, contabilidad, agenda, verticales) y a qué rol se conceden están en
[`db/seed/provisioning/modules/`](../db/seed/provisioning/README.md): encender un módulo es conceder sus permisos al rol.

### Autoridades: las tres formas de mandar

Son **datos**, no código. Se conceden y se quitan con filas, y ninguna concede las otras dos.

| Autoridad | Dónde vive | Alcance | Qué permite | Qué **no** permite |
|---|---|---|---|---|
| **Operador de plataforma** | `ada_master.users.platform_operator = true` | **todos** los tenants | aprobar/rechazar el **alta de un tenant completo** (onboarding) y marcarla aprovisionada; es la única que puede, porque el tenant aún no existe cuando se aprueba | **no** da administración de marketplace de ningún tenant: eso exige membresía y permiso **en cada uno** |
| **Administrador de tenant** | membresía **staff activa** en ese tenant + `PLATFORM_MEMBERSHIPS_MANAGE` en **su** base | **un** tenant | ver y gestionar las **solicitudes de marketplace** de ese tenant: activar, suspender, cambiar plan, autorizar perfil, decidir el dueño de la comisión | no toca altas de tenants completos, ni tenants donde no tenga membresía |
| **Operador sobre varios tenants** (mover prestadoras) | la misma que la anterior, **en origen y destino** | **dos** tenants | mover una membresía de proveedora entre dos tenants donde es administrador; `moveTargets` solo lista esos | no mueve hacia tenants donde no es administrador |

`myWorkspace` expone las dos primeras como **`platformOperator`** y **`tenantAdmin`**, así que el menú y las pestañas de la
administración se deciden con estos datos y no con el permiso suelto. La tercera no se expone: la decide el backend en cada
operación (`canOperateIn`).

**Quién ve qué, sin nombrar a nadie** (es el flag, no una persona):

| Quien tenga… | "Administración" | "Solicitudes de tenant" | "Miembros de marketplace" |
|---|---|---|---|
| `platform_operator = true` | sí, **en cualquier tenant** | sí, **siempre** (el alta no pertenece a ningún tenant) | solo en los tenants donde además tenga **membresía staff** (y el tenant tenga marketplace) |
| membresía staff + permiso en el tenant (sin el flag) | sí, en **ese** tenant | no | sí, en **ese** tenant, si tiene marketplace |
| las dos cosas | sí | sí | sí, en los tenants donde tenga la membresía |

Un tenant sin marketplace no ofrece "Miembros de marketplace" a nadie: no hay nada que administrar ahí.

**Propiedades que se desprenden de la tabla** (el backend ya las aplica):

- Dar `platform_operator` **no** convierte a nadie en administrador de todos los negocios: para marketplace sigue necesitando
  membresía + permiso en cada tenant.
- Un administrador de tenant **no** aprueba altas de tenants, aunque tenga el permiso en su base.
- Todo cambio queda en `platform_audit` con actor (onboarding y membresías), así que el uso del flag ultra-elevado tiene rastro.

### Cambiar de una autoridad a otra (siempre datos, sin tocar código)

| De → a | Qué se hace | Riesgo |
|---|---|---|
| Operador de plataforma → administrador de **algunos** tenants | `users.platform_operator = false` y dejar/crear la **membresía staff** + permiso solo en esos tenants | bajo; deja de poder aprobar altas |
| Administrador de algunos → operador de plataforma | `users.platform_operator = true` (las membresías se quedan como están) | alto: pasa a poder aprobar altas de cualquier tenant |
| Todo → nada | quitar `platform_operator` y la membresía staff del tenant (o bajarle el permiso al rol) | bajo |
| Añadir un tenant a su alcance | crear la fila de `memberships` con `zona='staff'` en ese tenant (el permiso lo lleva el rol de su base) | bajo |

> El provisioning ya crea la membresía staff de `root` en cada tenant nuevo, así que root administra el marketplace de todos
> sin que nadie tenga que acordarse.

### Cómo se conceden### Cómo se conceden

| Qué | Cómo |
|---|---|
| `platform_operator` | `psql -d ada_master -v login=<login> -f db/seed/provisioning/platform-operator.sql` |
| Host y zona de un tenant | `psql -d ada_master -v host=<host> -v zona=<staff|public> [-v tenant=<slug>] -f db/seed/provisioning/hosts.sql`; el provisioning lo hace con `--public-booking=<host>` (zona public) |
| Membresía (tenant + puerta + rol) | la crea el provisioning (owner/staff); el alta pública crea `provider`/`client` con zona `public` |
| Rol del tenant y sus permisos | provisioning inserta el `user_system` con su rol; los permisos por módulo, con los seeds de `modules/` |

## Iniciar sesión

| Campo | Valor |
|---|---|
| Tenant | `<tenant_slug>` |
| Usuario | `<NIT>` |
| Contraseña | `<NIT>` (inicial — **cambiar tras primer acceso**) |

---

## Acceso a Grafana

No hay que registrar nada en Grafana al crear un tenant: el plugin `tenant-postgres` lista los tenants desde las
membresías de `ada_master` del usuario y consulta la base del tenant (slug = nombre de la base) con el usuario
único `appuser`. Basta con que el provisioning cree la membresía activa y que las migraciones `tenant` (incluidas
las de permisos de dashboards, 128 y 129) estén aplicadas en la base nueva.

Arquitectura, permisos por dashboard y diagnóstico: [GRAFANA_ANALYTICS_ACCESS.md](GRAFANA_ANALYTICS_ACCESS.md).

---

## Entregable al cliente

Tabla lista para entregar tras cada provisioning. Reemplazar los campos entre `< >`.

| Campo | Valor |
|---|---|
| **Empresa** | `<Nombre Legal>` |
| **Tenant** | `<tenant_slug>` |
| **Usuario** | `<NIT>` |
| **Contraseña inicial** | `<NIT>` ⚠️ cambiar en primer acceso |
| **Email** | `<email>` |
| **Backend (ADA)** | https://ada.sotobotero.com/ADA_ENTERPISE_CORE/ |
| **Frontend / carga de inventario** | https://ada.sotobotero.com |
| **Reportes (Grafana)** | https://ada.sotobotero.com/grafana |
| **Guía de usuario** | https://github.com/sotobotero/sotobotero-projects-docs/blob/master/ada/qa/docs/qa/e2e-happy-path-inventario-facturacion.md |

---

## Ejemplo real ejecutado

```bash
bash provision-tenant.sh \
  db:qa public \
  '900987654-3' \
  'Academia Fitness Demo S.A.S' \
  'admin@academiafitness.co' \
  --clean-data \
  --country=CO \
  --seed-consumidor-final
```

| Campo | Valor |
|---|---|
| `tenant_slug` (auto) | `ac900987654` |
| NIT normalizado | `900987654` |
| Login / password inicial | `900987654` |
| Email | `admin@academiafitness.co` |

```bash
bash destroy-tenant.sh ac900987654
# → escribe "ac900987654" para confirmar
```

---

## Verificar estado

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)

# Todos los tenants registrados
docker exec $C psql -U postgres -d ada_master \
  -c "SELECT slug, company_name, mode, status FROM tenant_registry ORDER BY slug;"

# Memberships
docker exec $C psql -U postgres -d ada_master \
  -c "SELECT u.login, m.tenant_slug, m.role, m.status
      FROM memberships m JOIN users u ON u.id = m.user_id
      ORDER BY m.tenant_slug, u.login;"

# Usuarios en la DB de negocio
docker exec $C psql -U postgres -d <tenant_slug> \
  -c "SELECT id, login, role, status FROM public.user_system ORDER BY id;"
```

---

## Seguridad

| Ítem | Estado |
|---|---|
| Keycloak UUID real en ada_master | ✅ |
| Password único por tenant (SHA1 del NIT) | ✅ |
| Credenciales leídas del Docker secret del contenedor — no del operador | ✅ |
| Ningún secret en código, logs ni git | ✅ |
| Confirmación explícita (escribir slug) antes de destroy | ✅ |
| Forzar cambio de password en primer login | ⚠️ Pendiente |
| Entrega de credenciales al cliente | ✅ Email de bienvenida automático (requiere `.env.smtp`) |
