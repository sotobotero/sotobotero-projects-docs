# Provisioning de un nuevo tenant

## Arquitectura rápida

| Componente | Rol |
|---|---|
| `ada_master` | DB del framework — contiene la tabla `public.datasources` que registra los tenants |
| `<tenant_db>` | DB de negocio del tenant — esquemas: `billing`, `inventory`, `public`, etc. |
| `provision-tenant-db.sh` | Script en `infraestructure-as-code` que hace todo el proceso |

El app ADA lee `ada_master.public.datasources` al arrancar para saber a qué DBs conectarse. Sin ese registro el tenant no existe para el app.

---

## Prerequisitos

- Acceso SSH al servidor Ionos: `ssh ionos`
- Nombre del contenedor Patroni activo (verificar con `docker ps | grep patroni`)
- Credenciales del usuario `appuser` (en Docker secret `ada-db-password`)

---

## Crear un nuevo tenant

### 1. Copiar el script al contenedor Patroni

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)

docker cp \
  /home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles/provision-tenant-db.sh \
  $C:/tmp/provision-tenant-db.sh

docker exec $C chmod +x /tmp/provision-tenant-db.sh
```

### 2. Ejecutar el script

#### Modo básico — clonar y registrar
```bash
docker exec \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -e APP_USER_DB=appuser \
  -e APP_PASSWORD_DB=<password_de_appuser> \
  -e MASTER_DB=ada_master \
  -e DB_INTERNAL_HOST=192.168.1.20 \
  $C /tmp/provision-tenant-db.sh <nuevo_tenant> db:default_tenant public
```

#### Modo todo-en-uno — clonar, registrar y limpiar datos de negocio
Útil para crear un tenant de QA o cliente nuevo sin datos previos:
```bash
docker exec \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -e APP_USER_DB=appuser \
  -e APP_PASSWORD_DB=<password_de_appuser> \
  -e MASTER_DB=ada_master \
  -e DB_INTERNAL_HOST=192.168.1.20 \
  $C /tmp/provision-tenant-db.sh <nuevo_tenant> db:default_tenant public --clean-data
```

**El script hace automáticamente:**
1. Crea la DB `<nuevo_tenant>`
2. Clona esquema + datos desde `default_tenant` (o cualquier DB fuente)
3. Concede permisos a `appuser` en todos los schemas
4. Registra el tenant en `ada_master.public.datasources`
5. *(con `--clean-data`)* Borra todos los datos de negocio — deja solo catálogos y usuarios admin id 1 y 2

> Para usar un dump en lugar de clonar una DB en vivo:
> ```bash
> ... $C /tmp/provision-tenant-db.sh <nuevo_tenant> /path/al/dump.dump public
> ```

### 3. Restart del servicio ADA

```bash
docker service update --force ada-staging_ada
```

### 4. Configurar el perfil de la empresa (primera vez)

Conectarse a la DB del tenant y actualizar según el tipo de negocio:

```sql
-- Ejemplo para un gimnasio
UPDATE public.company_profile
SET
  name            = 'Gym QA Test',
  manage_contracts = true,
  month_stay      = 6
WHERE id = 1;
```

---

## Iniciar sesión en el nuevo tenant

En la pantalla de login de ADA:

| Campo | Valor |
|---|---|
| Tenant | `<nombre_registrado_en_datasources>` |
| Usuario | admin del tenant (heredado de la DB fuente) |

### Verificación mínima de usuarios para E2E

Antes de ejecutar pruebas E2E, validar que quedó al menos un usuario base activo (`id` 1 o 2):

```bash
docker exec $C psql -U postgres -d <nombre_tenant> -c \
  "SELECT id, login, role, status FROM public.user_system WHERE id IN (1,2) ORDER BY id;"
```

Esperado:
- al menos uno de esos usuarios con `status = 1`
- normalmente existen `root` (id=1) y `admin` (id=2)

Si no aparecen, recrear el tenant desde `db:default_tenant` o restaurar usuarios base antes del login.

---

## Verificar el registro

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)
docker exec $C psql -U postgres -d ada_master -c "SELECT id, name, url FROM public.datasources;"
```

---

## Limpiar datos de un tenant ya existente

Si el tenant ya existe y solo necesitas borrar los datos de negocio (ej: re-preparar QA):

```bash
C=$(docker ps --filter name=patroni --format "{{.Names}}" | head -1)
SCRIPT_PATH=/home/ionosadmin/infraestructure-as-code/docker-compose/postgres/db/dbfiles/otherfiles/clean-tenant-data.sql

docker cp $SCRIPT_PATH $C:/tmp/clean-tenant-data.sql
docker exec $C psql -U postgres -d <nombre_tenant> -f /tmp/clean-tenant-data.sql
```

**Qué limpia:**
- Borra: clientes, facturas, productos, proveedores, empleados, contabilidad, diario, kardex, audit log
- Resetea todas las secuencias a 1
- Mantiene: catálogos (nomenclaturas, tipos, impuestos), configuración de caja, `company_profile`, usuarios admin (id 1 y 2)

**Verificar resultado esperado:**
```bash
docker exec $C psql -U postgres -d <nombre_tenant> -c \
  "SELECT count(*) as customers FROM billing.customer;
   SELECT count(*) as invoices  FROM billing.invoice;
   SELECT count(*) as products  FROM inventory.product;
   SELECT count(*) as users     FROM public.user_system;"
# Esperado: customers=0, invoices=0, products=0, users=2
```

---

## TODO de seguridad — aprovisionamiento de usuarios iniciales

Estado actual observado en onboarding:
- Se crean usuarios base (`root` y `demo`).
- Existe una contraseña inicial genérica para `demo` (con hash conocido), lo cual no es aceptable para producción.

Objetivo:
- Eliminar credenciales iniciales predecibles/reutilizadas entre tenants.
- Definir un flujo seguro de creación y entrega de credenciales iniciales por tenant.

### Cambios requeridos en aprovisionamiento automático

- [ ] **No usar hash fijo ni contraseña por defecto compartida** (`demo/demo` o equivalentes).
- [ ] **Generar contraseña aleatoria única por tenant** (alta entropía) para el usuario inicial.
- [ ] **Marcar credencial como temporal** y forzar cambio en primer login.
- [ ] **Deshabilitar/eliminar `demo` en entornos productivos** (mantenerlo solo en QA/local si aplica).
- [ ] **Evitar exponer contraseñas en logs** del script/CI/CD.
- [ ] **Registrar auditoría mínima** del onboarding: tenant, fecha, actor, usuarios creados (sin secretos).

### Entrega segura de credenciales iniciales

- [ ] **No enviar credenciales por chat/correo plano**.
- [ ] **Entregar por canal seguro** (secret manager, vault, enlace de un solo uso o ticket cifrado).
- [ ] **Definir TTL para credenciales bootstrap** (expiran si no se usan).
- [ ] **Rotar inmediatamente** después de la primera autenticación exitosa.

### Criterio de aceptación (Definition of Done)

Un onboarding de tenant se considera correcto cuando:
- no existe ninguna contraseña inicial genérica reutilizable entre tenants,
- el usuario inicial obliga cambio de contraseña al primer acceso,
- la entrega de credenciales queda trazada por canal seguro,
- y no se imprimen secretos en logs de aprovisionamiento.
