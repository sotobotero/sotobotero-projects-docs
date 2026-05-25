# Booking Availability Pattern (Service-first)

## Objetivo

Reservar por **servicio** sin depender de `diary` legacy, usando disponibilidad semántica calculada en tiempo real desde los calendarios operativos de cada empleado.

---

## Regla funcional principal

- La **compañía** define un calendario global en templates (blueprint).
- Al crear un **empleado**, la UI precarga esos templates por defecto.
- El usuario ajusta el calendario del empleado antes de guardar.
- **Sin `employee_weekly_availability`, el empleado nunca aparece como candidato en ningún slot.**  
  Estar asignado a un servicio (`employeesByService`) no es suficiente: se requiere disponibilidad horaria configurada.

---

## Modelo de datos

### Configuración del servicio
- `inventory.product`
  - `slot_duration_minutes`
  - `requires_online_reservation_payment`
  - `online_reservation_deposit_percent`
- Estos campos aplican cuando `type = SERVICE`.

### Disponibilidad de empleado (fuente operativa)

| Tabla | Propósito | Recurrente | Fecha específica |
|---|---|---|---|
| `employee_weekly_availability` | Horario laboral semanal | ✅ | ❌ |
| `employee_break_schedule` | Pausas recurrentes intrajornada | ✅ | ❌ |
| `employee_availability_exception` | Bloqueos puntuales (festivos, vacaciones, etc.) | ❌ | ✅ |

> **Regla crítica:** `employee_weekly_availability` es la fuente primaria.  
> Si está vacía, `expandAndAccumulateFromWeeklyTemplates` no genera ningún slot para ese empleado y nunca es candidato.

### Templates de compañía (blueprints)

| Tabla | Propósito |
|---|---|
| `company_availability_template` | Horario base por día para nuevos empleados |
| `company_break_template` | Pausas base por día para nuevos empleados |
| `company_availability_exception_template` | Excepciones base (festivos/bloqueos) para propagar a empleados |

---

## Flujo de creación de empleado

1. Abrir `Nuevo empleado`.
2. Precargar template de compañía en la UI.
3. Usuario ajusta calendario del empleado.
4. Guardar empleado.
5. Persistir disponibilidad final del empleado en tablas `employee_*`.

---

## Algoritmo de disponibilidad (backend, runtime)

Clase: `DiaryBookingDomainService.expandAndAccumulateFromWeeklyTemplates`

```
Para cada empleado habilitado para el servicio:
  1. Leer employee_weekly_availability  → ventanas horarias
  2. Restar employee_break_schedule     → quitar pausas recurrentes
  3. Restar employee_availability_exception → quitar bloqueos puntuales
  4. Restar reservas existentes         → quitar slots ya ocupados
  5. Particionar por slot_duration_minutes
  6. Acumular en SlotAccumulator:
       key = "startTime|endTime"
       value.employeeIds.add(employeeId)  ← LinkedHashSet<Long>
7. Devolver slots ordenados, cada uno con:
     - startTime / endTime
     - capacity (empleados disponibles en ese slot)
     - status ("AVAILABLE" | "FULL")
     - candidateEmployeeIds: [id1, id2, ...]  ← SOLO empleados con disponibilidad real
```

> Los campos `candidateEmployeeIds` son la **fuente de verdad** para saber qué profesionales
> pueden ser asignados a un slot concreto. No dependen de GraphQL.

---

## Dos fuentes de datos en el frontend (diferencia clave)

| Fuente | Endpoint | Qué devuelve |
|---|---|---|
| `employeesByService(serviceId)` | GraphQL | Todos los empleados **asignados** al servicio, sin importar si tienen horario |
| `availability/by-service/{id}` | REST | Slots con `candidateEmployeeIds` = empleados que tienen horario y están disponibles ese momento |

**Solo los empleados presentes en `candidateEmployeeIds` de un slot son seleccionables para ese slot.**  
Un empleado puede aparecer en `employeesByService` pero no en `candidateEmployeeIds` si no tiene `employee_weekly_availability` configurada.

---

## Lógica del frontend — SlotPickerComponent

### Fuentes de datos cargadas en `ngOnInit`

```typescript
loadAvailabilityByService(serviceId)  // REST → BookingSlot[] con candidateEmployeeIds
loadEmployeesByService(serviceId)     // GraphQL → Employee[] (catálogo visual)
```

### Estado local

```typescript
selectedSlot: BookingSlot | null
selectedEmployee: Employee | null
employeeSelectionMode: 'none' | 'auto' | 'manual'
```

| Modo | Significado |
|---|---|
| `'none'` | Sin empleado seleccionado; auto-selección activa al cambiar slot |
| `'auto'` | Empleado seleccionado automáticamente; puede ser reemplazado al cambiar slot |
| `'manual'` | Usuario eligió explícitamente; no se sobreescribe al cambiar slot |

### `visibleEmployees` (getter)

Filtra el catálogo `employees` (GraphQL) para mostrar **solo los candidatos del slot activo**:

```typescript
get visibleEmployees(): Employee[] {
  if (!this.selectedSlot?.candidateEmployeeIds?.length) return this.employees;
  const candidates = new Set(
    this.selectedSlot.candidateEmployeeIds.map(id => this.normalizeEmployeeId(id))
  );
  return this.employees.filter(emp => candidates.has(this.normalizeEmployeeId(emp.id)));
}
```

> Los chips de profesionales en la UI solo muestran empleados de `visibleEmployees`.
> Si un empleado no aparece en `candidateEmployeeIds` del slot, simplemente no se muestra.

### `selectSlot(slot)` — flujo completo

```
1. Guardar slot seleccionado en estado local y en BookingStateService.
2. Si el empleado seleccionado manualmente ya no es candidato en el nuevo slot:
     - Deseleccionarlo.
     - Si era 'manual', mantener modo 'manual' (para no auto-seleccionar sin permiso del usuario).
3. Si no hay empleado y el modo no es 'manual':
     - Auto-seleccionar el primer candidato del slot (resolveDefaultEmployeeForSlot).
     - Modo → 'auto'.
```

### `selectEmployee(emp)` — flujo

```
1. Si emp no está en visibleEmployees → ignorar (guard).
2. Si es el mismo empleado → deseleccionar (toggle), modo → 'none'.
3. Si es otro empleado → seleccionar, modo → 'manual'.
```

### Auto-selección inicial al cargar disponibilidad

Al recibir los slots por primera vez, si el store no tiene ningún slot guardado:

```typescript
const firstAvailable = normalized.find(slot => slot.available);
if (firstAvailable) this.selectSlot(firstAvailable);
```

Esto dispara también la auto-selección del primer profesional candidato del slot.

### `normalizeEmployeeId(id)`

Los IDs de empleado vienen como `number` desde la REST API y como `string` desde GraphQL.
Para evitar fallos de comparación (`39217 !== "39217"`):

```typescript
private normalizeEmployeeId(id: string | number | null | undefined): string {
  return String(id ?? '').trim();
}
```

Se aplica en `visibleEmployees`, `selectSlot` y `resolveDefaultEmployeeForSlot`.

---

## Flujo de selección completo (paso a paso UX)

```
Usuario abre paso 2 (SlotPicker)
  → loadAvailabilityByService() + loadEmployeesByService()
  → Al recibir slots: auto-seleccionar primer slot disponible
  → Auto-seleccionar primer candidato del slot (modo = 'auto')

Usuario cambia de día
  → selectDate() filtra slots del día y llama selectSlot(primerSlotDisponible)
  → Si el empleado actual no es candidato del nuevo slot: deseleccionarlo
  → Auto-seleccionar primer candidato del nuevo slot (si modo ≠ 'manual')

Usuario selecciona manualmente un profesional
  → employeeSelectionMode = 'manual'
  → Al cambiar de slot: NO se auto-sobreescribe su selección
  → Si ese profesional no es candidato en el nuevo slot: sí se limpia

Usuario confirma (Continuar)
  → confirmSlot(): si no hay empleado, intentar auto-asignar uno final
  → Emitir slotSelected → BookingShellComponent avanza el stepper
```

---

## Decisiones de arquitectura

- Sin fallback a tablas legacy de free slots.
- Modelo service-first: `candidateEmployeeIds` por slot es la fuente de verdad operativa.
- `employeesByService` (GraphQL) es solo el catálogo visual; su presencia no garantiza disponibilidad.
- `employeeSelectionMode` protege la intención del usuario frente a auto-selecciones silenciosas.
- IDs siempre normalizados a `string` antes de comparar para evitar problemas `number vs string`.
- Mantener implementación simple, testeable y con zona horaria explícita por tenant/sede.

---

## Diagnóstico: "El profesional X nunca aparece como candidato"

Si un empleado está en `employeesByService` pero nunca aparece en chips ni en `candidateEmployeeIds`:

1. Verificar que tenga registros en `employee_weekly_availability`:
   ```graphql
   query {
     employeeWeeklyAvailability(employeeId: <ID>) {
       dayOfWeek startTime endTime active
     }
   }
   ```
2. Si el array está vacío → crear horario para el empleado.
3. Si tiene horario → verificar que no tenga `employee_availability_exception` bloqueando todo el rango consultado.
4. Confirmar que el empleado esté asignado al servicio correcto en `employee_service`.

---

## Regeneración de clientes frontend (REST + GraphQL)

La guía operativa completa de regeneración (comandos, configuración y referencias) se mantiene en el README principal:

- [README.md](../README.md)
