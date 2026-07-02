# Provisioning de un nuevo tenant

## Arquitectura

| Componente | Rol |
|---|---|
| `ada_master` | DB del framework — `tenant_registry`, `users`, `memberships` |
| `<tenant_db>` | DB de negocio — `billing`, `inventory`, `public.user_system`, etc. |
| `provision-tenant.sh` | **Script HOST** — orquesta Keycloak + DB (nuevo, punto de entrada) |
| `provision-tenant-db.sh` | **Script CONTAINER** — solo operaciones psql (llamado por el anterior) |

---

## Flujo completo de provisioning

```
[Host — provision-tenant.sh]
  │
  ├─ 1. Keycloak API (curl)
  │     └─ upsert usuario 'admin' → obtiene KEYCLOAK_UUID real
  │
  └─ 2. docker exec provision-tenant-db.sh (con KEYCLOAK_UUID + ADMIN_PASS_HASH)
         │
         ├─ Crea la DB del tenant
         ├─ Clona desde default_tenant (o dump)
         ├─ Concede privilegios a appuser en todos los schemas
         ├─ Registra en ada_master.tenant_registry (slug, db_name, company_name)
         ├─ Upsert admin en ada_master.users (keycloak_id real, login=NIT, pass=SHA1(NIT))
         ├─ Memberships: NIT + root → tenant (role='owner')
         ├─ INSERT person + user_system en la DB de negocio (NIT + nombre empresa)
         ├─ UPDATE company_profile: name + document_number; limpia campos de contacto del clone
         └─ (opcional) limpia datos de negocio
```

---

## Prerequisitos

- SSH al servidor Ionos: `ssh ionos`
- `KEYCLOAK_ADMIN_PASS` disponible (jamás hardcodearlo — ver `.vscode/.env.debug.local`)
- `APP_PASSWORD_DB` disponible (password de `appuser`)

---

## Ejecutar el provisioning

```bash
# Desde el servidor Ionos
KEYCLOAK_ADMIN_PASS=<pass_keycloak_admin> \
APP_PASSWORD_DB=<pass_appuser> \
/home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles/provision-tenant.sh \
  <tenant_slug> db:default_tenant public <NIT> "Nombre Legal de la Empresa S.A.S" info@empresa.co
```

### Con limpieza de datos de negocio (tenant nuevo / QA)

```bash
KEYCLOAK_ADMIN_PASS=<pass> APP_PASSWORD_DB=<pass> \
./provision-tenant.sh <tenant_slug> db:default_tenant public <NIT> "Nombre Legal" info@empresa.co --clean-data
```

### Parámetros

| Posición | Nombre | Ejemplo | Descripción |
|---|---|---|---|
| `$1` | `tenant_slug` | `gym_nueva` | Nombre de DB y runtime identifier del tenant |
| `$2` | `source` | `db:default_tenant` | DB fuente a clonar o ruta a dump |
| `$3` | `schemas` | `public` | Legacy; el script concede sobre todos los schemas |
| `$4` | `nit` | `900123456` | NIT / número de identificación de la empresa |
| `$5` | `company_name` | `"GymX S.A.S"` | Nombre legal de la empresa (quoted si tiene espacios) |
| `$6` | `email` | `info@gymx.co` | *(opcional)* Email de contacto — Keycloak, ada_master y company_profile; default `admin@<slug>.local` |
| `$7` | *(opcional)* | `--clean-data` | Limpia datos de negocio; mantiene catálogos y users id 1-2 |

### Variables de entorno requeridas

| Variable | Fuente | Descripción |
|---|---|---|
| `KEYCLOAK_ADMIN_PASS` | Secret / env manual | Password del admin master de Keycloak |
| `APP_PASSWORD_DB` | Secret / env manual | Password del `appuser` en PostgreSQL |

---

## Qué crea el script

### Keycloak (realm `arcadia`)

| Campo | Valor |
|---|---|
| `username` | `<NIT>` — un usuario por empresa, único en el realm |
| `email` | `admin@<tenant_slug>.local` |
| `password` | `<NIT>` (plain text en Keycloak, SHA1 en ada_master) |
| `temporary` | `false` (cambiar manualmente tras primer login) |

Si ya existe un usuario Keycloak con `username=<NIT>` (re-provisioning), solo actualiza el password.

### ada_master.tenant_registry

```
slug         = <tenant_slug>
db_name      = <tenant_slug>
company_name = <company_name>     ← nombre legal para queries en ada_master
mode         = 'business'
status       = 'active'
```

### ada_master.users

```
login         = '<NIT>'               ← identificador único por empresa
email         = 'admin@<tenant>.local'
keycloak_id   = <UUID real de Keycloak>
password_hash = SHA1(<NIT>)
```

### ada_master.memberships

| user (login) | tenant_slug | role | status |
|---|---|---|---|
| `<NIT>` | `<tenant_slug>` | `owner` | `active` |
| `root` | `<tenant_slug>` | `owner` | `active` |

### company_profile en la DB de negocio

El clone de `default_tenant` trae `company_profile` con datos de demo. El script actualiza solo los campos que conoce en el momento del provisioning:

| Campo | Valor | Razón |
|---|---|---|
| `name` | `<company_name>` | Nombre legal de la empresa |
| `document_number` | `<NIT>` | Identificación fiscal |
| `email` | `<email>` | Email del parámetro de entrada (o `admin@<slug>.local` si omitido) |
| `addres`, `phone` | `''` (vacío) | Desconocidos — el admin los rellena tras primer login |
| `url_website` | `NULL` | Desconocido |
| `bank_account_number`, `internation_bank_acount` | `''` | Desconocidos |
| `creditor_id`, `autorization_code` | `''` | Desconocidos |
| Resto (locale, timezone, impuestos, impresoras…) | **sin tocar** | Heredan del clone y son configurables por el admin |

### user_system en la DB de negocio

> **Regla de mapping crítica:** `LoginBean` resuelve la sesión con:
> ```java
> String resolvedLogin = memberships.get(0).get("login");   // de ada_master
> findByPropertiesConjunctionEq(UserSystem.class, Map.of("login", resolvedLogin));
> ```
> `ada_master.users.login = NIT` debe coincidir con `user_system.login = NIT`.
> Por eso el script inserta un usuario nuevo en `user_system` con `login = NIT`.

| id | login | role | status | origen |
|---|---|---|---|---|
| 1 | `root` | 1 (super admin) | 1 | clone de `default_tenant` |
| 2 | `admin` | 2 (System Administrator) | 1 | clone de `default_tenant` |
| 3+ | `<NIT>` | **2 (System Administrator)** | 1 | **insertado por este script** |

El INSERT usa herencia JOINED (`person` → `user_system`):
- `person.document_number = NIT`, `firts_name = company_name`, `full_name = company_name`
- `user_system.login = NIT`, `role/status/company/cash/view` copiados de id=2

---

## Iniciar sesión

| Campo | Valor |
|---|---|
| Tenant | `<tenant_slug>` |
| Usuario | `<NIT>` (número de identificación de la empresa) |
| Contraseña | `<NIT>` (contraseña inicial — **cambiar tras primer acceso**) |

---

## Restart del servicio ADA

```bash
docker service update --force ada-staging_ada
```

---

## Verificar el registro

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)

# Tenant en registry
docker exec $C psql -U postgres -d ada_master \
  -c "SELECT slug, db_name, mode, status FROM tenant_registry;"

# Memberships
docker exec $C psql -U postgres -d ada_master \
  -c "SELECT u.login, m.tenant_slug, m.role, m.status
      FROM memberships m
      JOIN users u ON u.id = m.user_id
      ORDER BY m.tenant_slug, u.login;"

# Usuarios base en la DB de negocio
docker exec $C psql -U postgres -d <tenant_slug> \
  -c "SELECT id, login, role, status FROM public.user_system WHERE id IN (1,2) ORDER BY id;"
```

---

## Limpiar datos de un tenant existente

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)
docker cp /home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles/clean-tenant-data.sql \
  $C:/tmp/clean-tenant-data.sql
docker exec $C psql -U postgres -d <tenant_slug> -f /tmp/clean-tenant-data.sql
```

**Qué limpia:** clientes, facturas, productos, empleados, contabilidad, diario, kardex, audit log. Resetea secuencias.
**Mantiene:** catálogos, `company_profile`, usuarios id 1 y 2.

---

## Seguridad

| Ítem | Estado |
|---|---|
| Keycloak UUID real (no placeholder) en ada_master | ✅ |
| Password único por tenant (SHA1 del NIT) en ada_master | ✅ |
| Credenciales nunca hardcodeadas en scripts | ✅ (env vars) |
| `KEYCLOAK_ADMIN_PASS` solo en `.vscode/.env.debug.local` (git-ignored) | ✅ |
| Forzar cambio de password en primer login | ⚠️ Pendiente |
| Entrega de credenciales por canal seguro (no chat/email) | ⚠️ Pendiente |
| TTL para credenciales bootstrap | ⚠️ Pendiente |
| Auditoría de onboarding (quién, cuándo, tenant) | ⚠️ Pendiente |
