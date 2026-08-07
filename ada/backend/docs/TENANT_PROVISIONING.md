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
cd infraestructure-ascode/docker-compose/postgres/db/dbfiles/otherfiles

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
               ├─ Clona desde default_tenant (skip si ya existe)
               ├─ Concede privilegios a appuser en todos los schemas
               ├─ Registra en ada_master.tenant_registry
               ├─ Upsert admin en ada_master.users (keycloak_id, login=NIT, SHA1)
               ├─ Memberships: NIT + root → tenant (role='owner')
               ├─ (opcional --clean-data) Limpia datos de negocio
               ├─ INSERT person + user_system (NIT admin, copiado de root id=1)
               ├─ UPDATE company_profile (name, NIT, email; limpia campos del clone)
               ├─ (opcional --country=XX) Trunca nomenclature, resembra geografía del país
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
  db:default_tenant public \
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
| `$1` | `source` | `db:default_tenant` | DB fuente a clonar, o ruta a un dump |
| `$2` | `schemas` | `public` | Legacy; el script concede sobre todos los schemas igualmente |
| `$3` | `nit` | `900987654-3` | NIT de la empresa; normalizado automáticamente |
| `$4` | `company_name` | `"Academia Fitness S.A.S"` | Nombre legal (quoted si tiene espacios) |
| `$5` | `email` | `info@empresa.co` | *(opcional)* Email admin; default `admin@<slug>.local` |
| `--clean-data` | *(flag)* | `--clean-data` | Limpia datos de negocio del clone; mantiene solo catálogos e id=1 |
| `--country=XX` | *(flag)* | `--country=CO` | Trunca `public.nomenclature` y la resembra con el seed del país indicado (código ISO). Actualiza además `company_profile.country`. Sin este flag se conserva la nomenclatura clonada. Disponible: `CO` (Colombia), `ES` (España). |
| `--seed-consumidor-final` | *(flag)* | `--seed-consumidor-final` | Inserta el cliente "Consumidor Final" (NIT DIAN 222222222222) requerido para facturación electrónica colombiana (Resolución 000042/2020). La geografía (Colombia/Bogotá D.C.) se resuelve automáticamente si `--country=CO` fue especificado. Idempotente — no falla si ya existe. |

### Idempotencia

El script es re-ejecutable sin DROP previo. Si la DB ya existe, salta el CREATE y el clone y solo actualiza registry, users, memberships, user_system y company_profile.

---

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

## Iniciar sesión

| Campo | Valor |
|---|---|
| Tenant | `<tenant_slug>` |
| Usuario | `<NIT>` |
| Contraseña | `<NIT>` (inicial — **cambiar tras primer acceso**) |

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
| **Frontend / carga de inventario** | https://arcadia.sotobotero.com |
| **Reportes (Grafana)** | https://arcadia.sotobotero.com/grafana |
| **Guía de usuario** | https://github.com/sotobotero/sotobotero-projects-docs/blob/master/ada/qa/docs/qa/e2e-happy-path-inventario-facturacion.md |

---

## Ejemplo real ejecutado

```bash
bash provision-tenant.sh \
  db:default_tenant public \
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
