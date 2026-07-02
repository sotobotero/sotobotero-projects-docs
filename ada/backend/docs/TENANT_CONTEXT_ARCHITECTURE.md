# Tenant Context Architecture (Booking-first)

## Objetivo

Definir una regla única, simple y consistente para resolver el tenant por request, alineada con:

- UX friendly (el usuario no introduce tenant manualmente).
- Arquitectura real actual: monolito ADA + frontend complementario en evolución.
- Keycloak como proveedor de identidad y autenticación global — solo hace lo que hace bien.
- Identidad federada: una persona, múltiples contextos (tenants).
- Resolución de tenant por dominio/subdominio cuando aplique.
- Modelo híbrido: marketplace compartido + tenant con DB propia según plan.

Este documento toma como **canónico** el diseño objetivo de la UH y separa explícitamente el **estado actual implementado** para evitar ambigüedad.

Referencia funcional principal:
- [`user-histories/active/uh-00-START-HERE-multitenant.md`](user-histories/active/uh-00-START-HERE-multitenant.md)
- Estado de todas las UHs: [`user-histories/INDEX.md`](user-histories/INDEX.md)

---

## Responsabilidades por capa

Cada capa tiene una responsabilidad única y no invade la de las demás.

| Capa | Responsabilidad | No hace |
|---|---|---|
| **Keycloak** | Identidad global, autenticación, sesiones, social login, OTP, emisión de JWT | Roles de negocio, membresías por tenant, permisos granulares |
| **ada_master** | Routing de tenants, membresías globales, configuración UI (bundles) | Lógica de negocio, permisos internos del tenant |
| **DB por tenant** | Roles granulares, permisos internos, datos de negocio | Identidad, autenticación |

---

## Modelo de actores — herencia RBAC unificada

Todos los actores que operan dentro de un tenant son subclases de `UserSystem`. `UserSystem` es el **sujeto RBAC universal** — tiene `Role` con sus permisos, pero sus credenciales internas (`login`/`password`) son **opcionales** para no forzar la capa de autenticación en actores que delegan a Keycloak.

```mermaid
flowchart TB
    classDef actor fill:#4A90D9,stroke:#2C5F8A,color:#fff
    classDef bizEntity fill:#E8F4E8,stroke:#5A8A5A,color:#2d4a2d
    classDef rbac fill:#FFF3CD,stroke:#B8860B,color:#5a4000
    classDef person fill:#F0F0F0,stroke:#888,color:#333

    subgraph IDENTIDAD ["🪪  Identidad (Person)"]
        direction TB
        P["Person
        ──────────────
        id
        external_id ← Keycloak sub
        mainEmail
        fullName
        documentNumber"]
    end

    subgraph ACTORES ["🔐  Actores autenticados (UserSystem)"]
        direction LR
        US["UserSystem
        ──────────────
        login  nullable
        password  nullable
        external_id heredado"]

        EMP["Employee
        ──────────
        jobPosition
        hourCost
        availability"]

        US -->|"extends JOINED"| EMP
    end


    subgraph RBAC_BOX ["🛡️  RBAC"]
        direction LR
        ROL["Role
        ────
        name"]
        PER["Permission
        ──────────
        resource
        action"]
        ROL -->|"tiene"| PER
    end

    subgraph NEGOCIO ["📋  Entidades de negocio"]
        direction LR
        CUS["Customer
        ──────────────
        billing.customer
        businessName
        tradeRegime
        accounts"]

        PRO["Provider
        ──────────────
        inventory.provaider
        businessName
        brand"]
    end

    P -->|"extends JOINED"| US
    US -->|"tiene 1 Role"| ROL
    US -. "vincula opcionalmente\ncuando factura" .-> CUS
    US -. "vincula opcionalmente\ncuando provee" .-> PRO

    class P person
    class US,EMP actor
    class ROL,PER rbac
    class CUS,PRO bizEntity
```

**Lectura del diagrama:**

- `UserSystem` es el **único sujeto de autenticación y RBAC**. Cualquier persona que tenga login en el sistema (interno u OAuth) pasa por aquí.
- `Employee` **hereda de `UserSystem`** (✅ implementado — migración 022 aplicada 2026-05-02).
- `Customer` y `Provider` **no heredan** de `UserSystem` — son entidades de negocio (contable / inventario) que pueden existir sin login. La asociación es **opcional** via `user_system_id nullable`.

**Ciclo de vida de una misma persona real:**

```
1. María se registra con Google
   → UserSystem creado (external_id = sub, login = NULL)

2. María reserva un turno
   → UserSystem usado directamente (RBAC role = CLIENT)

3. María recibe su primera factura
   → Customer creado (businessName, tradeRegime...)
   → Customer.user_system_id = maría.id

4. Don José provee insumos Y tiene acceso al portal
   → Provider creado (businessName = "Distribuidora José")
   → Provider.user_system_id = jose.id  ← activa acceso
   → Si es proveedor externo sin portal: user_system_id = NULL
```

**Migración aplicada (2026-05-02):**

```sql
-- 023-business-entities-link-to-usersystem.sql — ya aplicada
ALTER TABLE billing.customer
  ADD COLUMN IF NOT EXISTS user_system_id BIGINT REFERENCES public.user_system(id) ON DELETE SET NULL;

ALTER TABLE inventory.provaider
  ADD COLUMN IF NOT EXISTS user_system_id BIGINT REFERENCES public.user_system(id) ON DELETE SET NULL;
```

### Regla de identidad por tipo de actor

| Actor | AuthN | `login`/`password` | `external_id` | Obtiene RBAC fino |
|---|---|---|---|---|
| Admin/staff interno | Credentials en DB | ✅ required | NULL | ✅ vía `UserSystem.role` |
| Profesional OAuth | Keycloak / Google / OTP | NULL | ✅ `sub` Keycloak | ✅ vía `UserSystem.role` |
| Cliente OAuth | Keycloak / Google / OTP | NULL | ✅ `sub` Keycloak | ✅ vía `UserSystem.role` |

### Por qué `UserSystem` como base universal (no `Person` directo)

Si `Employee` o `Client` extendieran `Person` directamente (como estaba antes), necesitarían cada uno su propio FK a `public.role` para tener permisos. Al centralizar en `UserSystem`, el RBAC es una consecuencia de **ser un actor**, no de ser un tipo específico de actor.

### `Employee.role` — cargo laboral, no RBAC

`Employee.job_position` (hoy llamado `role VARCHAR(100)`) representa el **cargo organizacional** del empleado dentro del negocio: `"Gerente de operaciones"`, `"Colorista"`, `"Recepcionista"`. El valor por defecto en el código es `"Employee"`. Este campo es semánticamente correcto tal como está — **no es un intento de RBAC**.

El RBAC falta porque `Employee` aún no llega a `UserSystem`. Son dos conceptos distintos que coexisten en el mismo actor:

| Campo | Semántica | Dónde debe vivir |
|---|---|---|
| `job_position` | Cargo laboral visible en UI | `employee` (✅ renombrado de `role`) |
| `role` (FK) | Permisos del sistema | heredado de `UserSystem` (✅ implementado) |

**Correcciones aplicadas (2026-05-02):**

1. ✅ `Employee extends UserSystem` (era `extends Person`) — migración 022
2. ✅ `login` y `password` nullable en `public.user_system` — migración 021
3. ✅ `employee.role` renombrado a `employee.job_position` — migración 022

ADR: [`.ai/decisions/2026-05-02-employee-extends-usersystem-rbac.md`](../.ai/decisions/2026-05-02-employee-extends-usersystem-rbac.md) — incluye deuda LSP documentada.

### `Customer` (billing) — no requiere alineación a `UserSystem`

`Customer` existe en `org.sotobotero.ada.module.billing.entities`. Es una **entidad contable**, no un actor de autenticación. Representa a la persona o empresa que aparece en facturas, contratos y cuentas — puede ser una empresa jurídica que nunca se loguea en la app.

`Customer` y el actor autenticado son **dos objetos distintos** que a veces coinciden en la misma persona real:

```
María se loguea en la app   →  UserSystem  (autenticación + RBAC)
María paga una factura      →  Customer    (entidad contable, no necesita login)
```

Fusionarlos (`Customer extends UserSystem`) implicaría:
- **Migración destructiva de datos:** cada `billing.customer` existente necesitaría una fila nueva en `public.user_system` — todos los servicios romperían en arranque sin esa migración.
- **Violación de SRP:** mezclaría responsabilidad contable con autenticación.
- `Patient` tiene un FK directo a `Customer` — se vería arrastrado en la migración.

**Decisión:** `Customer` permanece como `extends Person`. El actor OAuth que reserva turnos se maneja como `UserSystem` directamente (o una subclase futura `AppUser extends UserSystem`), con una referencia opcional hacia `Customer` cuando esa persona también tenga historial de facturación.

---

## Estado actual vs objetivo

### Estado actual (implementado hoy)

- El monolito ADA resuelve contexto de tenant por `Host` y por información de request (`JWT` y compatibilidad con `X-TenantID`) en el filter.
- `ada_master` sigue siendo la fuente de verdad para registro de tenants y bundles.
- Los permisos granulares de negocio viven en la base de datos de cada tenant.
- En booking, el aislamiento inicial en datos compartidos se está resolviendo por `professionalId` (estrategia pragmática de arranque).
- No existe todavía una estrategia transversal de aislamiento por `tenant_id` en todas las tablas del dominio.

### Objetivo incremental

- Mantener `booking-first`: primero estabilidad operativa del flujo de reservas.
- Endurecer aislamiento por fases, evitando sobrediseño temprano.
- Evolucionar desde reglas por `professionalId` hacia políticas más robustas cuando el modelo de datos lo soporte de forma transversal.

---

## Keycloak — identidad y autenticación global

Keycloak gestiona exclusivamente:

- Quién eres (identidad única global)
- Cómo te autenticas (Google, Facebook, OTP, user+password)
- Roles generales: `ROLE_USER`, `ROLE_ADMIN`, `ROLE_PROFESSIONAL`
- Claims estándar: `sub`, `email`, `email_verified`
- Scopes: `openid`, `profile`, `email`
- Gestión de sesiones y refresh tokens

Para alinear completamente con la UH, se distinguen dos artefactos:

- **Token de identidad (Keycloak)**: mínimo, sin datos de negocio.
- **Token de aplicación (ADA/Spring)**: enriquecido con contexto de tenant después de resolver `Host` + membresía en `ada_master`.

Ejemplo de token de identidad (Keycloak):

```json
{
  "sub": "uuid-keycloak",
  "email": "maria@gmail.com",
  "email_verified": true,
    "roles": ["ROLE_USER"],
  "scope": "openid profile email"
}
```

Ejemplo de token de aplicación (objetivo UH):

```json
{
    "sub": "uuid-keycloak",
    "user_id": "uuid-global-maria",
    "tenant": "glamour",
    "tenant_mode": "business",
    "role": "staff",
    "exp": 1234567890
}
```

### Por qué `sub` y no email como identificador

El `sub` es el identificador único e inmutable que Keycloak asigna a cada usuario. No cambia aunque el usuario cambie su email o conecte otro social login.

```
María se registra con Google  → sub: "uuid-123", email: maria@gmail.com
María cambia su email          → sub: "uuid-123"  ← no cambia
María conecta Facebook         → sub: "uuid-123"  ← sigue siendo el mismo
```

Si se usara email como FK en ada_master, un cambio de email rompería todas las relaciones. Con `sub` eso nunca ocurre.

### Identity Providers — estado de soporte

| IDP | Soporte en Keycloak | Prioridad | Notas |
|---|---|---|---|
| Google OAuth2 | Nativo | P0 — lanzamiento | Mayor adopción en el segmento |
| OTP por email | Nativo | P0 — lanzamiento | Magic link o código 6 dígitos |
| Usuario + contraseña | Nativo | P0 — lanzamiento | Opción de último recurso |
| Facebook | Nativo | P1 — iteración 2 | Social login |
| Instagram | IDP custom OAuth2 | P2 — iteración 3 | Configuración manual adicional |
| TikTok | IDP custom OAuth2 | P2 — iteración 3 | Configuración manual adicional |

Lanzar con Google + OTP + user/password cubre el 80% de los casos de uso y reduce la complejidad inicial.

---

## ada_master — routing y membresías globales

`ada_master` es la base de datos existente extendida. Responde preguntas que deben resolverse **antes** de conectar contexto de negocio:

```
tenant_registry  → ¿este slug existe? ¿a qué DB apunta? ¿está activo?
users/memberships → ¿este usuario global tiene acceso a este tenant y con qué rol?
bundles          → ¿qué configuración UI aplica a este tenant? (existente)
```

### Estructura requerida (objetivo UH)

```sql
-- Extender tenant_registry existente
ALTER TABLE tenant_registry ADD COLUMN IF NOT EXISTS mode    VARCHAR DEFAULT 'business';
-- 'marketplace' | 'business'
ALTER TABLE tenant_registry ADD COLUMN IF NOT EXISTS db_name VARCHAR;
-- null = base de datos compartida (marketplace), valor = DB propia (business)

-- Identidad global
CREATE TABLE IF NOT EXISTS users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email        VARCHAR UNIQUE NOT NULL,
    keycloak_id  VARCHAR UNIQUE NOT NULL,
    verified_at  TIMESTAMPTZ,
  created_at   TIMESTAMPTZ DEFAULT now(),
);

-- Membresías globales — una persona, N tenants
CREATE TABLE IF NOT EXISTS memberships (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL REFERENCES users(id),
    tenant_slug  VARCHAR NOT NULL REFERENCES tenant_registry(slug),
    role         VARCHAR NOT NULL,
    status       VARCHAR DEFAULT 'active',
    created_at   TIMESTAMPTZ DEFAULT now(),
    UNIQUE (user_id, tenant_slug)
);
```

Relación con la estructura existente:
- `tenant_registry` ya existe → agregar columnas `mode` y `db_name`
- `bundles` ya existe → sin cambios
- `users` y `memberships` → tablas nuevas para identidad global + acceso multi-tenant
- `user_tenants` puede mantenerse temporalmente como compatibilidad técnica durante migración.

### Por qué membresías en ada_master y no en Keycloak

Keycloak Groups/Roles no escala para miles de tenants:

| Aspecto | Groups en Keycloak | users/memberships en ada_master |
|---|---|---|
| Escala a miles de tenants | Se degrada | Es solo una tabla |
| Onboarding nuevo tenant | Requiere Keycloak Admin API | INSERT en ada_master |
| Keycloak caído | Sin login Y sin membresías | Sin login, membresías disponibles |
| Queries complejas | Imposible | SQL normal |
| Roles granulares de negocio | No aplica | Viven en DB del tenant |

### Routing actual — cómo se construye la conexión hoy

`ConnectionProviderImpl.buildJdbcUrl()` construye la URL JDBC directamente del `tenantId` en contexto:

```java
// ConnectionProviderImpl.java
private String buildJdbcUrl(String baseUrl, String tenantId) {
    int lastSlashIndex = baseUrl.lastIndexOf('/');
    return baseUrl.substring(0, lastSlashIndex + 1) + tenantId;
    // resultado: jdbc:postgresql://host:5432/<tenantId>
}
```

El `tenantId` es siempre el valor de `TenantContext.getCurrentTenant()` → que proviene del header `X-TenantID` → que es el `slug` de `tenant_registry`.

**Consecuencias directas del diseño actual:**

| Campo | Uso real hoy | Observación |
|---|---|---|
| `slug` | Identificador de runtime en **todo**: header, TenantContext, JDBC URL, FK de memberships | Es la única fuente de verdad de routing |
| `db_name` | **No leído por ningún código de routing** — se escribe pero no se consulta | Campo de metadata / preparación futura |

**Invariante obligatoria:** mientras el routing use `slug` directamente como nombre de DB, `slug` DEBE ser idéntico al nombre de la base de datos en PostgreSQL. Si difieren, el app intenta conectar a una DB que no existe y falla en runtime.

```
slug = "gym_nueva"  →  jdbc:postgresql://host:5432/gym_nueva
                                                    ^^^^^^^^
                                          debe existir esta DB
```

---

### TODO — Routing para modo marketplace

Cuando se implemente el modo `marketplace` (tenants que comparten una sola DB), el routing deberá:

1. **Consultar `tenant_registry`** para obtener `db_name` y `mode` antes de crear la conexión.
2. **Bifurcar según `mode`:**
   - `mode = 'business'` → conectar a `db_name` (una DB dedicada por tenant)
   - `mode = 'marketplace'` → conectar a la DB compartida del marketplace (ignorar `slug` como DB name)
3. **Aplicar aislamiento lógico** dentro de la DB compartida (row-level security por `tenant_id`).

Pseudocódigo del routing futuro:

```java
// TODO: implementar cuando se active el modo marketplace
TenantRegistryEntry entry = tenantRegistryRepo.findBySlug(tenantId);
if (entry.getMode().equals("marketplace")) {
    return buildJdbcUrl(baseUrl, MARKETPLACE_DB_NAME); // DB compartida
} else {
    return buildJdbcUrl(baseUrl, entry.getDbName());   // DB dedicada
}
```

**Prerequisitos antes de activar marketplace:**
- [ ] `ConnectionProviderImpl` debe hacer el lookup a `tenant_registry` (hoy no lo hace)
- [ ] Todas las tablas compartidas deben tener columna `tenant_id` con políticas RLS
- [ ] El `slug` deja de ser equivalente al nombre de la DB para tenants marketplace
- [ ] `MultiTenantConnectionProviderImpl` necesita caché del registro para no golpear ada_master en cada request

---

## DB por tenant — roles y permisos granulares

Cada tenant con plan Business o superior tiene su propia DB con la siguiente estructura de permisos interna:

```
DB: db_glamour
├── schema billing
├── schema accounting
└── schema operations
    ├── staff
    ├── roles          → dueño, recepcionista, estilista, colorista
    ├── permissions    → puede_ver_caja, puede_editar_agenda, puede_ver_reportes
    ├── staff_roles    → qué rol tiene cada empleado (referenciando user_id global)
    ├── appointments
    ├── clients
    └── services
```

Los permisos granulares los define cada negocio dentro de su propio espacio — no están en Keycloak ni en ada_master.

---

## Modelo híbrido — marketplace + tenant propio

### Planes y modelo de aislamiento

| Plan | Perfil | Modo | Almacenamiento | Aislamiento |
|---|---|---|---|---|
| Free | Profesional individual | marketplace | Base de datos compartida | Aislamiento lógico (objetivo: row-level) |
| Pro | Negocio pequeño 1–5 personas | business | Base de datos compartida | Objetivo UH: row-level + RLS fuerte |
| Business | Salón mediano 5–50 personas | business | Base de datos dedicada por tenant | Aislamiento alto |
| Enterprise | Cadena / franquicia | business | Base de datos dedicada + servidor opcional | Aislamiento alto + infraestructura dedicada |

### Nota de precisión sobre aislamiento

- En PostgreSQL, hoy existen estructuras por schemas dentro de una misma base, pero para conversación de arquitectura y producto se estandariza el lenguaje a **base de datos** (compartida vs dedicada).
- El endurecimiento con RLS global queda como fase posterior; actualmente debe manejarse con cautela porque el modelo no incorpora `tenant_id` de forma transversal en todas las tablas.
- Por alineación con la UH, RLS se mantiene como **objetivo de diseño**, no como estado actual ya desplegado.

### Estructura de datos por modo

```
DB: marketplace (compartida — planes Free y Pro)
└── schema public
    ├── profiles     → profesionales independientes (keycloak_id como FK)
    ├── services     → servicios ofrecidos
    ├── bookings     → reservas
    └── reviews      → reseñas públicas

DB: db_glamour (tenant propio — plan Business+)
├── schema billing
├── schema accounting
└── schema operations

DB: ada_master (global — siempre)
├── tenant_registry
├── users
├── memberships
└── bundles
```

### Aislamiento en booking (realidad actual)

- En la iteración actual, booking aplica aislamiento operativo principalmente por `professionalId` y relaciones de servicio/profesional/disponibilidad.
- Esta estrategia es válida para arrancar MVP, pero no reemplaza un modelo transversal con `tenant_id` y políticas RLS globales.
- Regla práctica: cualquier nueva consulta de booking debe mantener filtro explícito por contexto de profesional/tenant para evitar fugas.

---

## Flujos de onboarding

### Caso 1 — Profesional independiente (marketplace compartido)

```mermaid
sequenceDiagram
    participant U as Profesional
    participant APP as Spring Boot
    participant KC as Keycloak
    participant ADA as ada_master
    participant MKT as DB marketplace

    U->>APP: Click "Continuar con Google"
    APP->>KC: OAuth2 flow Google
    KC-->>APP: JWT { sub: "uuid-123", email: "carlos@gmail.com" }
    APP->>ADA: ¿Existe membership(user="uuid-123", slug="marketplace")?
    ADA-->>APP: No existe → primer login
    APP->>ADA: UPSERT users + INSERT memberships(user_id, tenant_slug="marketplace", role="professional")
    APP->>MKT: INSERT profiles(keycloak_id="uuid-123", nombre, especialidad)
    APP-->>U: Bienvenido al marketplace — completa tu perfil
```

No se crea ninguna DB dedicada. El profesional vive en base de datos compartida. Sin fricción — el usuario no elige ni ve nada relacionado con tenants.

### Caso 2 — Negocio con suscripción (tenant propio)

```mermaid
sequenceDiagram
    participant U as Dueño
    participant APP as Spring Boot
    participant KC as Keycloak
    participant ADA as ada_master
    participant PG as PostgreSQL

    U->>APP: Registro negocio (nombre, slug="glamour", plan=Business)
    APP->>KC: Crear usuario Keycloak via Admin API
    KC-->>APP: { sub: "uuid-456" }
    APP->>ADA: INSERT tenant_registry(slug="glamour", db_name="db_glamour", mode="business", status="provisioning")
    APP->>ADA: UPSERT users + INSERT memberships(user_id, tenant_slug="glamour", role="owner")
    APP->>PG: CREATE DATABASE db_glamour
    APP->>PG: CREATE SCHEMA billing, accounting, operations
    APP->>PG: Ejecutar migraciones iniciales
    APP->>ADA: UPDATE tenant_registry SET status="active"
    APP->>KC: Enviar email verificación al dueño
    APP-->>U: Tu espacio está listo en glamour.tuplatforma.com
```

### Caso 3 — Cliente final (llega por link del negocio)

```mermaid
sequenceDiagram
    participant U as Cliente
    participant APP as Spring Boot
    participant KC as Keycloak
    participant ADA as ada_master
    participant TDB as DB glamour

    U->>APP: glamour.tuplatforma.com/reservar
    APP->>APP: Host header → slug="glamour"
    APP->>ADA: tenant_registry(slug="glamour") → db_name, status=active
    U->>KC: Login con Instagram
    KC-->>APP: JWT { sub: "uuid-789", email: "lucia@gmail.com" }
    APP->>ADA: ¿Existe membership(user="uuid-789", slug="glamour")?
    ADA-->>APP: No existe → primer acceso a este tenant
    APP->>ADA: UPSERT users + INSERT memberships(user_id, tenant_slug="glamour", role="client")
    APP->>TDB: INSERT clients(keycloak_id="uuid-789", nombre)
    APP-->>U: Reserva disponible — sin ver nada de tenants
```

### Caso 4 — Profesional que escala a negocio propio

```mermaid
sequenceDiagram
    participant U as Profesional→Dueño
    participant APP as Spring Boot
    participant ADA as ada_master
    participant MKT as DB marketplace
    participant PG as PostgreSQL

    U->>APP: Activa plan Business, slug="carlos-studio"
    APP->>ADA: INSERT tenant_registry(slug="carlos-studio", mode="business")
    APP->>PG: CREATE DATABASE db_carlos_studio + schemas
    APP->>ADA: INSERT memberships(user_id="uuid-123", tenant_slug="carlos-studio", role="owner")
    Note over ADA: memberships ahora refleja DOS contextos para uuid-123:\n- marketplace\n- carlos-studio
    APP->>MKT: Leer datos existentes del profesional
    APP->>PG: Migrar clientes, reseñas, servicios
    APP-->>U: Tu espacio está listo — tus datos fueron migrados
    Note over U: Sigue apareciendo en marketplace\nY tiene su propio espacio
```

### Caso 5 — Staff invitado por el dueño

```mermaid
sequenceDiagram
    participant D as Dueño
    participant APP as Spring Boot
    participant ADA as ada_master
    participant KC as Keycloak
    participant S as Staff
    participant TDB as DB glamour

    D->>APP: Invitar empleado (email: ana@gmail.com)
    APP->>ADA: INSERT invitations(email, tenant_slug="glamour", role="staff")
    APP->>KC: Enviar email con link de invitación
    S->>APP: Click link → glamour.tuplatforma.com/invitacion?token=xxx
    APP->>APP: Host header → slug="glamour" (tenant resuelto)
    S->>KC: Elige IDP (Google / OTP / user+pass)
    KC-->>APP: JWT { sub: "uuid-staff-001" }
    APP->>ADA: UPSERT users + INSERT memberships(user_id, tenant_slug="glamour", role="staff")
    APP->>TDB: INSERT staff_roles(keycloak_id="uuid-staff-001", role="staff")
    APP-->>S: Bienvenida a Glamour Peluquería
```

---

## Resolución de tenant en runtime — Spring Boot

### Estado real actual (implementado hoy)

- La resolución de tenant en el monolito vigente ocurre en `FilterAllRequestDispatcher`.
- Prioridad operativa actual:
    1. claim `tenant` en JWT (si llega),
    2. fallback por `Host/subdominio` en rutas públicas,
    3. `X-TenantID` solo como compatibilidad,
    4. validación de mismatch si llegan JWT + `X-TenantID` y difieren.
- En `prod/staging`, si no hay tenant resoluble, debe fallar request.

### Objetivo evolutivo

Los diagramas siguientes representan el diseño objetivo de la UH: `Host/subdominio` como contexto primario y validación de membresía en `ada_master` para rutas protegidas.

Una vez el usuario está autenticado, cada request pasa por el filter que resuelve el tenant en este orden:

```mermaid
flowchart TD
    A[Request HTTP] --> B{¿Trae JWT válido?}
    B -->|Sí| C[Extraer sub del JWT]
    C --> D[Resolver tenant por Host header\nconsultar ada_master.tenant_registry]
    D --> E{¿memberships user + slug existe?}
    E -->|Sí| F[Conectar a DB del tenant\nset search_path si aplica]
    E -->|No y ruta pública| G[Auto-enroll si corresponde\neo Error 403]
    F --> H[Consultar permisos granulares\ndentro del tenant]
    H --> I[SecurityContext listo\nContinuar request]
    B -->|No| J{¿Ruta pública booking/onboarding?}
    J -->|Sí| K{¿Host/subdominio resoluble?}
    K -->|Sí| L[tenant = subdominio\nContinuar sin auth]
    K -->|No en prod| M[Error 400: tenant no resoluble]
    K -->|No en dev| N[tenant = default local — solo dev]
    J -->|No| O[Error 401: autenticación requerida]
```

### Diagrama 2 — Secuencia request booking público

```mermaid
sequenceDiagram
    participant Client as Cliente/Browser
    participant API as ADA Filter
    participant ADA as ada_master
    participant TDB as DB Tenant

    Client->>API: POST /restapi/v1/diary/booking/by-service/{id}
    alt Trae JWT válido
        API->>API: Extraer sub del JWT
        API->>ADA: tenant_registry por Host → db_name
        API->>ADA: memberships(user, slug) → acceso confirmado
        ADA-->>API: tenant resuelto
    else No JWT — ruta pública
        API->>ADA: Resolver por Host (glamour.tuplatforma.com)
        ADA-->>API: tenant=glamour
    end
    API->>TDB: Conectar → ejecutar operación
    API-->>Client: 200/4xx según reglas de negocio
```

### Regla operativa por tipo de ruta

**Rutas públicas de booking/onboarding:**
- Resolver tenant por Host/subdominio como regla principal.
- JWT es opcional en entrada pública; si existe, se usa para enriquecer contexto de usuario.
- Si ambos fallan: `prod/staging` → Error 400 / `local/dev` → tenant default.

**Rutas no públicas / internas:**
- JWT obligatorio.
- Tenant resuelto por Host + membership (`users/memberships`) según diseño UH.
- Aceptar `X-TenantID` solo por compatibilidad transicional cuando aplique.
- Si JWT y `X-TenantID` no coinciden → Error 400 mismatch.
- Sin tenant resoluble → Error 401/403.

---

## SSO entre subdominios — cookie de dominio raíz

Keycloak setea una cookie en el dominio raíz al autenticar:

```
Set-Cookie: KEYCLOAK_SESSION=xxx; Domain=.tuplatforma.com; HttpOnly; Secure; SameSite=Lax
```

Cualquier subdominio la recibe automáticamente:

```
app.tuplatforma.com        ✓ SSO automático
glamour.tuplatforma.com    ✓ SSO automático
donjose.tuplatforma.com    ✓ SSO automático
```

El usuario se loguea **una sola vez** y el sistema resuelve el tenant silenciosamente al cambiar de subdominio. Si la cookie expiró → Keycloak redirige a login.

---

## Switcher de contexto — usuario con múltiples tenants

Un usuario puede pertenecer a más de un tenant. La app expone un switcher:

```
┌─────────────────────────────┐
│  Cambiar de espacio          │
│                              │
│  🌐 Marketplace              │
│  ✂️  Glamour Peluquería  ←   │
│  💈 Barbería Don José        │
└─────────────────────────────┘
```

La lista se obtiene de:

```sql
SELECT tr.slug, tr.mode, tr.plan
FROM memberships m
JOIN users u ON u.id = m.user_id
JOIN tenant_registry tr ON tr.slug = m.tenant_slug
WHERE u.keycloak_id = :sub
    AND m.status = 'active'
  AND tr.status = 'active'
```

Al seleccionar un contexto → redirige al subdominio correspondiente → Spring Boot resuelve nuevo tenant desde Host → sin re-login gracias a la cookie de dominio raíz.

---

## Notas de implementación

- Filter principal: `src/main/java/org/sotobotero/ada/core/beans/util/FilterAllRequestDispatcher.java`
- Pruebas: `src/test/java/org/sotobotero/ada/core/beans/util/FilterAllRequestDispatcherTest.java`

### Pseudocódigo del filter actualizado

```java
@Component
public class FilterAllRequestDispatcher {

    public void doFilter(HttpServletRequest request, ...) {

        String tenant = null;

        // 1. Intentar desde JWT
        String jwt = extractJwt(request);
        if (jwt != null) {
            String sub = extractSub(jwt);
            String slug = resolveSlugFromHost(request); // Host header
            if (adaMaster.userTenantExists(sub, slug)) {
                tenant = slug;
            }
        }

        // 2. Fallback rutas públicas — Host header
        if (tenant == null && isPublicRoute(request)) {
            tenant = resolveSlugFromHost(request);
        }

        // 3. Sin tenant resoluble
        if (tenant == null) {
            if (isLocalEnvironment()) {
                tenant = defaultLocalTenant; // solo dev
            } else {
                throw new TenantNotResolvableException();
            }
        }

        // 4. Conectar al datasource correcto
        TenantContext.set(tenant);
        chain.doFilter(request, response);
    }
}
```

---

## Sistema de autorización — permisos granulares por tenant

Cada tenant tiene su propio catálogo de permisos. Este capítulo describe cómo se asignan, cómo se almacenan y cómo se resuelven en tiempo de login.

### Entidades implicadas

```
Role ──────────────── role_permision ──────── Permission
  │                  (qué puede el rol)          │
  │ bypass (flag)                                │
  │                                              │
UserSystem ──── user_permission_grant ───────── Permission
               (qué se le ha concedido          (permite sumar permisos
                explícitamente)                  fuera del rol)

UserSystem ──── user_permission_revoke ──────── Permission
               (qué se le ha revocado
                explícitamente)
```

### Cómo se guardan los permisos

Cuando se asigna o cambia el rol de un usuario, el sistema escribe automáticamente en `user_permission_grant` una fila por cada permiso del rol (con `source = role-sync`). Esto convierte la tabla de grants en la **única fuente de verdad** sobre qué tiene asignado cada usuario, independientemente de cómo se llegó a esa asignación.

También es posible añadir grants manuales (`source = manual-grant`) para dar permisos adicionales puntuales sin cambiar el rol, y revokes para retirar permisos concretos sin cambiar el rol.

### Cómo se resuelven los permisos en el login

En el momento del login, el `PermissionResolver` calcula los permisos efectivos del usuario y los carga en la sesión HTTP antes de que la UI los lea:

```
¿El rol tiene bypass activado?
    Sí → el usuario recibe todos los permisos del rol directamente
          (sin pasar por el sistema de grants/revokes)
    No → permisos efectivos = grants activos − revokes activos
```

```mermaid
flowchart TD
    L[Login] --> R{role.bypass?}
    R -->|Sí| RP[Devuelve todos los permisos del rol]
    R -->|No| G[Lee user_permission_grant activos]
    G --> V[Resta user_permission_revoke activos]
    V --> EF[Permisos efectivos]
    RP --> S[Sesión HTTP]
    EF --> S
    S --> UI[UI lee getCurrentUser.getPermissions]
```

### El flag bypass

`role.bypass = true` es el mecanismo para roles de sistema (como `root`) que deben tener acceso total sin gestión de grants individuales. Con bypass activado el resolver devuelve todos los permisos del rol sin consultar las tablas de grants y revokes.

Por defecto todos los roles tienen `bypass = false`. Solo se activa manualmente para roles de superadministración.

### Roles por defecto — contrato de seed obligatorio

Toda base de datos de tenant **debe entregarse poblada** con estos tres roles en los IDs exactos indicados. El código fuente contiene constantes que referencian estos IDs por valor; si los IDs no existen en la DB el sistema falla en runtime.

| id | name | bypass | Propósito |
|---|---|---|---|
| `1` | `root` | `true` | Superadministrador técnico. `bypass = true` — el resolver le devuelve todos los permisos del rol sin pasar por grants/revokes. Uso exclusivo de soporte e infraestructura. |
| `2` | `administrador del sistema` | `false` | Usuario del cliente con funciones de administración. Se entrega con los permisos preconfigurados correspondientes al plan contratado. Es quien gestiona empleados, configuración del negocio y catálogos. |
| `3` | `usuario basico` | `false` | Rol mínimo asignado automáticamente a usuarios creados por el sistema (empleados creados desde UI, signups OAuth). Acceso mínimo: únicamente cambiar su propia contraseña (`/pages/core/common/ChangePasword.xhtml`). El primer login de estos usuarios los lleva directamente a esa pantalla. |

**Constantes del código que dependen de estos IDs** (`Common.java`):

```java
// Vista a la que se redirige un usuario creado automáticamente en su primer login
public static final long DEFAULT_AUTO_USER_VIEW_ID = 30L;  // /pages/core/common/ChangePasword.xhtml

// Rol asignado a usuarios creados automáticamente por el sistema
public static final long DEFAULT_AUTO_USER_ROLE_ID = 3L;   // usuario basico
```

Estas constantes se usan en `EmployeeController.prepareCreate()` para asignar vista y rol por defecto al crear un empleado. Si el rol `3` o la vista `30` no existen en la DB del tenant, la creación de empleados falla silenciosamente.

**Migración de referencia:** `033-configure-default-auto-user-role.sql` — configura el rol `3` y le asigna el permiso mínimo de cambio de contraseña. Debe ejecutarse en cada tenant al provisionar la DB.

### Qué no hace este sistema

- No gestiona autenticación — eso es Keycloak.
- No reemplaza la resolución de tenant — el tenant se resuelve antes, en el filter de request.
- Los permisos son internos al tenant: cada DB de negocio tiene su propio catálogo de `Role` y `Permission`.

---

## Pendiente posterior (no bloqueante para booking-first)

- Job de migración marketplace → business cuando un profesional escala de plan (Caso 4).
- Endurecer política RLS para planes Pro en base de datos compartida.
- Implementar IDPs P1/P2: Facebook, Instagram, TikTok.
- Cache Redis para consultas frecuentes a `ada_master` (tenant_registry + users/memberships) en alta concurrencia.