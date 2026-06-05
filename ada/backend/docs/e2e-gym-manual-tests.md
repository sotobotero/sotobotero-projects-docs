# E2E Manual Tests — Caso de uso: Gimnasio

> Ejecutar en orden. Cada sección asume que la anterior se completó.  
> URL base local: `http://localhost:8080/ADA_ENTERPISE_CORE`

---

## 0. Pre-requisitos

| Requisito | Verificar |
|---|---|
| Backend corriendo | `mvn compile -Pdev` → servidor en `:8080` |
| Tenant activo en DB | `SELECT * FROM tenant_registry WHERE slug = '<tu-tenant>'` |
| Usuario administrador creado | cuenta con rol `admin` en el tenant |

---

## 1. Login

1. Ir a `http://localhost:8080/ADA_ENTERPISE_CORE`
2. Ingresar usuario y contraseña del admin del tenant gimnasio.
3. **Verificar:** llega al dashboard sin error 401/403.

---

## 2. Configurar el producto (mensualidad)

> Solo la primera vez. Si ya existe el producto, saltar a la sección 3.

**Ruta:** Inventario → Productos → Nuevo

| Campo | Valor |
|---|---|
| Nombre | `Mensualidad Gym` |
| Tipo | Servicio |
| Precio | `120.000` (o el que aplique) |
| **Cíclico** | ✅ activado |
| Estado | `SERVICE_AVAILABLE` |

**Guardar.** Anotar el ID del producto (visible en la lista).

**Verificar:** producto aparece en lista con estado `SERVICE_AVAILABLE`.

---

## 3. Crear cliente

> Si ya existe el cliente, saltar a la sección 4.

**Ruta:** Clientes → Nuevo

| Campo | Valor |
|---|---|
| Nombre | `Juan Pérez` (o cualquier nombre de prueba) |
| Correo | `juan@test.com` |
| Identificación | `123456789` |

**Guardar.** Anotar el ID del cliente.

---

## 4. Crear factura de inscripción + contrato

**Ruta:** Facturación → Nueva Factura

### 4.1 Cabecera de la factura

| Campo | Valor |
|---|---|
| Cliente | `Juan Pérez` |
| Fecha | Hoy |
| **Crear contrato** | ✅ activado (`makeContract = true`) |
| **Meses de permanencia** | `6` (o `0` para mes a mes sin permanencia) |

### 4.2 Línea de detalle

| Campo | Valor |
|---|---|
| Producto | `Mensualidad Gym` |
| Cantidad | `1` |
| Precio unitario | `120.000` |

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

El scheduler corre diario automáticamente. Para forzarlo manualmente basta con avanzar la fecha del sistema un mes **o** invocar el endpoint de debug (si está habilitado).

> En entorno local, la fecha se puede ajustar en el `billingDate` del servicio si hay un override de `Clock`, o esperando al día equivalente del mes siguiente.

**Verificar tras el ciclo:**
- Aparece una nueva factura generada automáticamente para `Juan Pérez`.
- `lastInvoice` del contrato apunta a la nueva factura.
- `pendingMonths` decrementó en 1 (pasa de `6` a `5`).

---

## 6. Finalizar contrato

**Ruta:** Clientes → `Juan Pérez` → Contratos → Seleccionar contrato → Finalizar

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
