# E2E Manual Tests — Caso de uso: Gimnasio

> Ejecutar en orden. Cada sección asume que la anterior se completó.  
> URL staging: `https://ada.sotobotero.com/ADA_ENTERPISE_CORE`

## Alcance de esta guía

Esta guía usa un **caso hipotético de gimnasio** para facilitar las pruebas E2E.

El sistema no está limitado a gimnasios: soporta casos de negocio con:
- solo **servicios**,
- solo **productos**,
- **productos compuestos** (producción),
- o combinaciones mixtas (productos físicos + servicios).

Ejemplo mixto para este caso:
- servicio mensual facturable mes a mes (contrato),
- producto comodín sin costo (por ejemplo, tarjeta de acceso),
- producto físico opcional con inventario (por ejemplo, suplemento vitamínico).

---

## Interfaz del Sistema

El sistema es **muy intuitivo** y sigue un patrón consistente en todas sus funcionalidades:

### Estructura general

- **Menú superior:** Clasificado por módulos funcionales (Seguridad, Parámetros comunes, Facturación, Inventario, Contabilidad, Agenda, Inteligencia de negocio)
- **Submenús:** Al hacer clic en cada módulo se despliegan las diferentes opciones disponibles
- **Área de trabajo:** Siempre muestra un listado de entidades/elementos de esa funcionalidad
- **Tabla:** Con cabecera filtrable (búsqueda, ordenamiento por columnas)
- **Botonera:** Botones para las operaciones principales: **Create** (crear), **Edit** (editar), **Remove** (eliminar), **More Options** (acciones adicionales según el caso)

![Área de Trabajo - Listado de entidades](images/AreaDetrabajoListas.png)

### Flujo de formularios (Modales)

Cuando pulsas cualquier botón de la botonera (**Create**, **Edit**, **Rectify**, etc.):

1. Se abre un **modal (ventana) con el formulario** respectivo para procesar esa acción
2. Sigue los pasos indicados en el formulario
3. **Campos obligatorios:** Marcados con `*` — deben completarse antes de guardar
4. **Botones de acción:** En cada modal encontrarás:
   - **Save/Guardar:** Para guardar los cambios
   - **Add** (si aplica): Para agregar líneas de detalle (ej: líneas en una factura)
   - **Remove** (si aplica): Para eliminar líneas
   - Otros botones según la funcionalidad

![Ventana Modal - Formularios con pasos](images/VenanadModales.png)

La intuitividad está en que **este mismo patrón se repite en todas las funcionalidades** del sistema: desde Facturación hasta Inventario, desde Contabilidad hasta Agenda.

---

## 0. Pre-requisitos

| Requisito | Verificar |
|---|---|
| Backend disponible | `https://ada.sotobotero.com/ADA_ENTERPISE_CORE` |
| Tenant activo en DB | `SELECT id, name, url FROM public.datasources WHERE name = '<tu-tenant>';` (DB `ada_master`) |
| Usuario administrador/base activo | `SELECT id, login, role, status FROM public.user_system WHERE id IN (1,2);` (DB del tenant) y al menos uno con `status=1` |
| Credenciales iniciales | usar las credenciales entregadas durante el alta/onboarding del tenant |

---

## 1. Login

1. Ir a `https://ada.sotobotero.com/ADA_ENTERPISE_CORE`

![Pantalla de Login](images/login.png)

2. Completar los campos:
   - **Tenant ID:** Ingresar el ID del tenant que fue entregado en tu proceso de onboarding
   - **Usuario:** Usar las credenciales entregadas en el onboarding (ej: usuario administrador/base activo)
   - **Contraseña:** La contraseña asociada a ese usuario

3. **Verificar:** Llega al dashboard sin error 401/403.

---

## 2. Configurar producto o servicio (según el caso)

> Solo la primera vez. Si ya existe el ítem, saltar a la sección 3.

Para este ejemplo (Gym) se usa un **servicio** de mensualidad.

> **📋 Características predefinidas en nuevos tenants**
> 
> Cuando se provisiona un nuevo tenant, el sistema genera automáticamente:
> - Líneas de producto
> - Grupos
> - Cajas
> - Ubicaciones
> - Listas de precios
> - *Etc.*
>
> **Objetivo:** agilizar la puesta en marcha.
>
> **Flexibilidad:** Todo es **completamente parametrizable**. Cada cliente puede:
> - Crear características propias
> - Modificar las existentes según su estructura organizativa
> - Coordinar migraciones de datos
> - Importar/exportar vía plantillas de Excel
>
> Esto puede hacerse antes, durante o después de iniciar operaciones.

**Ruta:** Parametros comunes → Productos → Nuevo

| Campo | Valor |
|---|---|
| Nombre | `Mensualidad Gym` |
| Tipo | Servicio |
| Precio | `120.000` (o el que aplique) |
| **Cíclico** | ✅ activado |
| Estado | `SERVICE_AVAILABLE` |

**Guardar.** Anotar el ID del producto (visible en la lista).

**Verificar:** el ítem aparece en lista con estado `SERVICE_AVAILABLE`.

---

## 3. Crear cliente

> Si ya existe el cliente, saltar a la sección 4.

**Ruta:** Facturación → Clientes → Nuevo

| Campo | Valor |
|---|---|
| Nombre | `Juan Pérez` (o cualquier nombre de prueba) |
| Correo | `juan@test.com` |
| Identificación | `123456789` |

**Guardar.** Anotar el ID del cliente.

---

## 4. Crear factura de inscripción + contrato

**Ruta:** Facturación → Facturas → botón crear

### 4.1 Cabecera de la factura

| Campo | Valor |
|---|---|
| Cliente | `Juan Pérez` |
| Fecha | Hoy |
| **Agregar producto o servicio** | Pulse el boton agregasr de la tabla, busque y selecciones el producto o servicio `Mensualidad Gym` |
| **Crear contrato** | ✅ activado (`makeContract = true`) |
| **Meses de permanencia** | `6` (o `0` para mes a mes sin permanencia) |

### 4.2 Línea de detalle

| Campo | Valor |
|---|---|
| Producto | `Mensualidad Gym` |
| Cantidad | `1` |
| Precio unitario | `120.000` |

> Nota: en otros casos de uso se puede facturar un producto físico, un servicio, o una combinación de ambos.

**Guardar factura.**

**Verificar:**
- Factura queda con estado activo/pagada.
- En Contratos (Clientes → Ver Cliente → Contratos) aparece un nuevo registro:
  - Estado: `CONTRACT_ACTIVE`
  - `monthsStay`: `6` (o lo que se ingresó)
  - `pendingMonths`: `6`
  - Producto: `Mensualidad Gym`
  - Producto cambia a `SERVICE_RENTED`

---

## 5. Simular facturación cíclica (ciclo mensual)

El scheduler corre automáticamente **todos los días a las 3:00 AM UTC**. Para forzarlo manualmente sin esperar hay dos opciones:

---

### Opción A — Endpoint REST (disparo inmediato)

**curl:**
```bash
curl -X POST http://localhost:8080/ADA_ENTERPISE_CORE/restapi/v1/billing/trigger-cycle
```

**Swagger UI:**
1. Ir a `http://localhost:8080/ADA_ENTERPISE_CORE/swagger-ui.html`
2. Buscar la sección **billing-trigger-api**
3. Expandir `POST /restapi/v1/billing/trigger-cycle`
4. Pulsar **Try it out** → **Execute**

El endpoint procesa el ciclo para todos los tenants activos en ese momento.

---

### Opción B — Ajustar BD para que el job de las 3 AM lo agarre

El scheduler compara un único campo: `billing_cicle.start_day_billing` (número entero = día del mes) con el día del mes actual. **No hay cálculo de fechas en el ciclo** — es simplemente "el día N de cada mes se factura este ciclo".

Además tiene un filtro anti-duplicado en dos niveles:
- **Nivel 1 (DAO):** excluye contratos cuya `last_invoice.invoice_date` sea de hoy.
- **Nivel 2 (servicio):** valida que hayan transcurrido al menos `billing_cicle.end_day_billing` días desde la última factura — **este valor se toma dinámicamente del ciclo de cada cliente**, no es un número fijo. Ejemplo: ciclo "Primero de cada mes" tiene `end_day_billing=30` → requiere 30 días; ciclo quincenal tiene `end_day_billing=14` → requiere 14 días.

**Condiciones que deben cumplirse simultáneamente:**

| # | Campo | Condición |
|---|---|---|
| 1 | `billing_cicle.start_day_billing` | = día del mes de hoy (ej: `7` si hoy es 7 de junio) |
| 2 | `contract_items.last_invoice → invoice.invoice_date` | NOT entre las 00:00 y 23:59 de hoy (anti-duplicado) |
| 3 | `contract_items.pending_month` | > 0 |
| 4 | `contract_items.rental` | > 0 |

**1. Ver el estado actual de los contratos activos:**
```sql
SELECT
  ci.id              AS contract_id,
  ci.pending_month,
  ci.rental,
  bc.id              AS billing_cicle_id,
  bc.name            AS cicle_name,
  bc.start_day_billing,
  EXTRACT(DAY FROM NOW())::integer AS today_day,
  i.invoice_date     AS last_invoice_date
FROM billing.contract_items ci
JOIN billing.customer cu ON cu.id = ci.customer
JOIN billing.account a ON a.customer = cu.id AND a.type = 1
JOIN billing.billing_cicle bc ON bc.id = a.billing_cycle
JOIN billing.invoice i ON i.id = ci.last_invoice
WHERE ci.pending_month > 0;
```

**2. Actualizar `start_day_billing` al día de hoy** (condición 1):
```sql
-- Reemplaza <ID_BILLING_CICLE> con el id del query anterior
UPDATE billing.billing_cicle
SET start_day_billing = EXTRACT(DAY FROM NOW())::integer
WHERE id = <ID_BILLING_CICLE>;
```

**3. Retrofechar la `invoice_date` de la última factura al mes pasado** (condición 2 — anti-duplicado):
```sql
-- Reemplaza <ID_LAST_INVOICE> con el last_invoice del query anterior
UPDATE billing.invoice
SET invoice_date = invoice_date - INTERVAL '1 month'
WHERE id = <ID_LAST_INVOICE>;
```

> **Nota:** Solo en entornos de prueba. Revertir ambos cambios después si es necesario.

**Verificar tras el ciclo:**
- Aparece una nueva factura generada automáticamente para `Juan Pérez`.
- `lastInvoice` del contrato apunta a la nueva factura.
- `pendingMonths` decrementó en 1 (pasa de `6` a `5`).

---

## 6. Finalizar contrato

**Ruta:** Facturación → Clientes → `Juan Pérez` → Contratos → Seleccionar contrato → Finalizar

**Verificar:**
- Estado del contrato: `CONTRACT_FINISHED`
- `pendingMonths`: `0`
- Producto `Mensualidad Gym` vuelve a `SERVICE_AVAILABLE`

---

## 7. Caso borde: contrato sin permanencia (monthsStay = 0)

Repetir sección 4 pero con **Meses de permanencia = 0**.

**Verificar:**
- Contrato creado normalmente.
- El ciclo automático **no** genera nuevas facturas (sin meses pendientes).
- Útil para servicio de día, VoD, clase suelta.

---

## 8. Caso borde: intento de duplicado

Crear una segunda factura para `Juan Pérez` con **el mismo producto** en la misma fecha.

**Verificar:**
- Si el producto ya tiene un contrato activo (`SERVICE_RENTED`), el sistema debe advertir o rechazar la segunda creación de contrato (guard de idempotencia).
- No se crea un contrato duplicado.

---

## Resumen de estados esperados

| Momento | Estado contrato | Estado producto |
|---|---|---|
| Antes de factura | — | `SERVICE_AVAILABLE` |
| Factura creada con `makeContract=true` | `CONTRACT_ACTIVE` | `SERVICE_RENTED` |
| Ciclo mensual ejecutado | `CONTRACT_ACTIVE` | `SERVICE_RENTED` |
| Contrato finalizado | `CONTRACT_FINISHED` | `SERVICE_AVAILABLE` |
