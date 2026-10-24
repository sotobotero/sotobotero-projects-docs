# E2E manual — Reporte de caja

La caja es el movimiento real de dinero del día: lo que entra y lo que sale, sin importar de qué módulo venga. El
reporte se saca por caja y por rango de fechas (*Caja → Reporte de caja*). Antes: [README de QA](README.md).

## Qué debe aparecer

| Origen | Cómo se prueba | Signo | Etiqueta en el reporte |
|---|---|---|---|
| Venta | [Inventario y facturación](e2e-happy-path-inventario-facturacion.md), pasos 3 y 4 | Entrada | Ventas |
| Rectificativa (devolución de venta) | mismo flujo, paso 6 | Salida | Devoluciones |
| Anticipo / abono de separado, abono de cartera | [Separados y cartera](e2e-cartera-separados.md), 1.1, 1.4, 3.3 | Entrada | Recibos |
| Devolución de dinero al cancelar un separado | Separados y cartera, 2.3 | Salida | Devoluciones |
| Compra pagada de contado | [Compras y cuentas por pagar](e2e-compras-cuentas-por-pagar.md), 2.1 | Salida | Compras de contado |
| Abono a proveedor | Compras, 3.1 | Salida | Pago a proveedores |
| Reverso de un abono a proveedor | Compras, 3.3 | Entrada | Pago a proveedores |
| Devolución de compra | Compras, 4.3 | Entrada, en la **caja elegida al devolver** | Devoluciones de compra |
| Reembolso del proveedor | Compras, 4.4 | Entrada | Reembolsos de proveedores |
| Cruce de saldo a favor | Compras, 4.6 | **No aparece** (no mueve dinero) | — |

## Reglas que se comprueban

1. **La caja es del día en que se mueve el dinero**, no la del documento original. Una devolución de compra de hace un mes
   entra hoy, en la caja elegida hoy.
2. **Solo lo pagado de contado** entra por la devolución de una compra; lo que estaba a crédito reduce la deuda o deja saldo a
   favor, no mueve la caja.
3. **Medio de pago distinto de efectivo** (transferencia, tarjeta, bono) también es movimiento de caja y sale en el reporte;
   cambia lo que el cajero entrega físicamente (el comprobante en lugar del efectivo), no si se registra.
4. **Cada movimiento sale una sola vez.** Una compra mixta no duplica su parte de contado ni su parte a crédito.

## Pasos

### 1. Antes
- **Acción:** generar el reporte de la Caja A para hoy y anotar los totales (si hay movimientos previos).

### 2. Ejecutar los flujos
- **Acción:** correr los bloques indicados de los flujos de la tabla, anotando el monto de cada operación.

### 3. Comparar
- **Acción:** generar de nuevo el reporte y revisar fila por fila contra la tabla anterior.
- **Esperado:**
  - Cada operación aparece con su etiqueta y signo; el total por medio de pago cuadra con la suma de las operaciones.
  - El cruce de saldo a favor y la parte a crédito de una compra mixta no aparecen.
  - Un rango de fechas sin movimientos muestra el mensaje "No hay información para el reporte con los filtros elegidos".

### 4. Cierre
- **Acción:** *Caja → Cierre* de la Caja A.
- **Esperado:** el cierre contempla todos los movimientos del día, los totales por medio de pago cuadran y se genera el
  comprobante sin error.

*Última revisión: 2026-10-24.*
