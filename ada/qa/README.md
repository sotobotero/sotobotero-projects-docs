# QA manual — cómo probar el sistema

Guía para quien prueba la funcionalidad sin conocer el código. Cada flujo es un documento aparte, con pasos
numerados, datos de ejemplo y el resultado esperado de cada paso.

> **Paso 0, siempre: [00-acceso.md](00-acceso.md).** Ahí están las direcciones, el login, qué cara se usa para qué y
> los tres casos de entrada reales. Este documento **no repite direcciones**: si cambia el dominio, se cambia solo allí.

## Flujos, en el orden que conviene probarlos

Primero lo básico (productos, ventas, caja), luego lo que depende de eso, y el ciclo completo al final como
validación humana de que todo cuadra junto.

| # | Flujo | Qué prueba | Documento |
|---|---|---|---|
| 1 | Inventario y facturación | Productos, entrada de stock, venta con 3 medios de pago, rectificativa, cierre de caja | [e2e-happy-path-inventario-facturacion.md](e2e-happy-path-inventario-facturacion.md) |
| 2 | Separados y cartera | Separado con anticipo, abonos, entrega, cancelación, venta fiada, límite de crédito, cobro de cartera | [e2e-cartera-separados.md](e2e-cartera-separados.md) |
| 3 | Compras y cuentas por pagar | Compra de contado, a crédito y mixta; abonos y reverso; devolución al proveedor; saldo a favor; cupo de crédito | [e2e-compras-cuentas-por-pagar.md](e2e-compras-cuentas-por-pagar.md) |
| 4 | Reporte de caja | Que la caja refleje lo que de verdad entró y salió, venga del flujo que venga | [e2e-reporte-de-caja.md](e2e-reporte-de-caja.md) |
| 5 | **Ciclo completo** | Compra → venta → separado → fiado → cartera → pago y devolución al proveedor, con conciliación de inventario, caja, cartera y cuentas por pagar | [e2e-ciclo-completo.md](e2e-ciclo-completo.md) |
| 6 | Conteo de inventario | Carga inicial de un almacén sin sistema, unidades sin soporte, conteo de auditoría y el límite con facturación | [e2e-conteo-de-inventario.md](e2e-conteo-de-inventario.md) |
| 7 | Gimnasio | Reservas y membresías de punta a punta | [e2e-gym-manual-tests.md](e2e-gym-manual-tests.md) |

### Guías para usar el producto (para quien lo usa, no para quien lo prueba)

En lenguaje llano, con el recorrido del día: qué se hace en el celular, qué en el computador y qué queda hecho solo.

| Guía | Para quién | Documento |
|---|---|---|
| **Restaurante — montarlo** | el dueño: mesas, qué se publica en la carta, platos con foto y precio, personas y caja | [restaurante-configuracion.md](restaurante-configuracion.md) |
| **Restaurante — atenderlo** | mesero, cocina y quien cobra: mesa, pedido, cocina, cuenta y lo que queda en inventario y caja | [guia-restaurante.md](guia-restaurante.md) |

### Guiones para presentar (no son guías de prueba)

Sirven para mostrar el producto a un cliente; el orden lo manda la historia, no la cobertura.

| Guion | Qué muestra | Documento |
|---|---|---|
| **Restaurante** (histórico) | Carta, mesa, pedido, cocina, cobro, kardex y lo financiero, escena por escena; se conserva como evidencia | [legacy/e2e-restaurante-demo.md](../legacy/e2e-restaurante-demo.md) |
| **Cargas de archivo** | Plantillas, productos con solo el nombre, rechazadas que se corrigen, proveedores y clientes | [e2e-cargas-archivo-demo.md](e2e-cargas-archivo-demo.md) |
| **Ciclo del negocio** | Guía en lenguaje llano para el dueño, con el guion de cuatro videos cortos | [guia-ciclo-del-negocio.md](guia-ciclo-del-negocio.md) |

### Documentos de apoyo (para preparar datos, no para probar)

| Documento | Para qué |
|---|---|
| [INVENTORY_ONBOARDING_STRATEGY.md](INVENTORY_ONBOARDING_STRATEGY.md) | Cargar el inventario de un negocio que parte de cero |
| [PLAN_SEED_LOTES_Y_PERFORMANCE.md](PLAN_SEED_LOTES_Y_PERFORMANCE.md) · [SEED_LOTES_RUNBOOK.md](SEED_LOTES_RUNBOOK.md) | Generar datos de prueba con lotes y vencimientos |

## Antes de empezar

1. **Un negocio (tenant) de pruebas**, nunca uno con datos reales: el que entrega el equipo.
2. **Un usuario con permisos** sobre lo que se va a probar. Si una opción no aparece en el menú, falta el permiso del
   rol: se pide, no se busca otra ruta.
3. **Datos base**: un cliente, un proveedor, tres productos con stock, un medio de pago en efectivo y uno electrónico,
   y una **caja** abierta para el usuario.
4. **Períodos contables abiertos** para hoy. Sin período la operación se guarda, pero su asiento queda en *Pendientes
   de contabilizar* (que también se prueba).
5. Anotar las cifras iniciales (stock, saldos, total de caja): los resultados esperados se expresan como cambios
   respecto a ellas.

## Cómo leer un flujo

- Cada paso tiene **Acción** y **Esperado**. Si lo esperado no ocurre, el flujo se detiene y se reporta.
- Los montos son redondos a propósito: usar los mismos para que los resultados coincidan con el documento.
- Ruta de menú entre comillas, por ejemplo *Inventario → Cuentas por pagar*.

## Cómo reportar un fallo

Flujo y número de paso, qué se hizo, qué se esperaba, qué pasó, captura, hora aproximada y el negocio y el usuario
usados. Con la hora se encuentra el error en el servidor.

## Mantener estos documentos al día

- Un flujo se actualiza en el mismo cambio que modifica la pantalla o la regla que prueba.
- Cada documento indica al final su **última revisión**; si tiene más de seis meses sin pasar por QA, tratarlo como
  dudoso.
- **Las direcciones viven solo en [00-acceso.md](00-acceso.md).** Al escribir un flujo, referirse a las caras por su
  nombre (Pulso Mobile, Backoffice, Sirius) y, si hace falta la dirección, enlazar a ese documento.
- **Cómo se escribe un documento de estos:** vive en `ai-workspace/DESIGN.md` (jerarquía, verbos del usuario,
  fuente única para lo que cambia y la prueba de los 5 segundos). Si un documento no lo pasa, se rehace antes de
  publicarlo.
- Esta carpeta se publica en el repositorio público de documentación: no poner nombres de clientes, direcciones,
  contraseñas ni datos reales de ningún negocio.
