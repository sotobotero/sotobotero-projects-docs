# QA manual — cómo probar ADA

Guía para quien prueba la funcionalidad sin conocer el código. Cada flujo es un documento aparte, con pasos numerados,
datos de ejemplo y el resultado esperado de cada paso.

## Flujos disponibles

Orden sugerido: primero lo básico (productos, ventas, caja), luego lo que depende de eso, y el ciclo completo al final como
validación humana de que todo cuadra junto.

| # | Flujo | Qué prueba | Documento |
|---|---|---|---|
| 1 | Inventario y facturación | Productos, entrada de stock, venta con 3 medios de pago, rectificativa, cierre de caja | [e2e-happy-path-inventario-facturacion.md](e2e-happy-path-inventario-facturacion.md) |
| 2 | Separados y cartera | Separado con anticipo, abonos, entrega, cancelación, venta fiada, límite de crédito, cobro de cartera | [e2e-cartera-separados.md](e2e-cartera-separados.md) |
| 3 | Compras y cuentas por pagar | Compra de contado, a crédito y mixta; abonos y reverso; devolución al proveedor; saldo a favor (reembolso y cruce); cupo de crédito | [e2e-compras-cuentas-por-pagar.md](e2e-compras-cuentas-por-pagar.md) |
| 4 | Reporte de caja | Que la caja refleje lo que de verdad entró y salió, venga del flujo que venga | [e2e-reporte-de-caja.md](e2e-reporte-de-caja.md) |
| 5 | **Ciclo completo** | Compra → venta → separado → fiado → cartera → pago y devolución al proveedor, y conciliación de inventario, caja, cartera y cuentas por pagar | [e2e-ciclo-completo.md](e2e-ciclo-completo.md) |
| 6 | Gimnasio (reservas y membresías) | Caso de uso de un gimnasio de punta a punta | [e2e-gym-manual-tests.md](e2e-gym-manual-tests.md) |

Documentos de apoyo (para quien prepara datos, no para probar):

| Documento | Para qué |
|---|---|
| [INVENTORY_ONBOARDING_STRATEGY.md](INVENTORY_ONBOARDING_STRATEGY.md) | Cómo cargar el inventario de un negocio que parte de cero |
| [PLAN_SEED_LOTES_Y_PERFORMANCE.md](PLAN_SEED_LOTES_Y_PERFORMANCE.md) y [SEED_LOTES_RUNBOOK.md](SEED_LOTES_RUNBOOK.md) | Generar datos de prueba con lotes y vencimientos |

## Dónde se prueba (las tres aplicaciones)

Todo vive bajo un mismo dominio, `https://ada.sotobotero.com`:

| Aplicación | Dirección | Para qué |
|---|---|---|
| **Arcadia** (aplicación nueva, Angular) | <https://ada.sotobotero.com/> | Inventario y reservas en la interfaz nueva |
| **ADA** (aplicación completa, JSF) | <https://ada.sotobotero.com/ADA_ENTERPISE_CORE/> | Facturación, caja, separados, cartera, compras, cuentas por pagar, contabilidad y reportes |
| **Grafana** (analítica) | <https://ada.sotobotero.com/grafana/> | Tableros de ventas, inventario y vencimientos |

- **Inicio de sesión:** con el **usuario y la contraseña que te entregó el equipo**, para el negocio (tenant) que te asignaron.
  Es la misma cuenta en las tres; Arcadia y Grafana te llevan a la pantalla de acceso del sistema (`sso.sotobotero.com`) y
  vuelven solos. Si te pide elegir negocio, elige el de pruebas.
- **Nunca** usar la cuenta de un negocio real, ni compartir ni escribir las credenciales en reportes o capturas.
- **Qué probar en cuál:** los flujos de esta carpeta se hacen en **ADA**, salvo que el documento diga otra cosa. Las
  pantallas de compras, cuentas por pagar, separados y cartera **solo existen en ADA por ahora**. En Arcadia se prueba
  inventario y reservas; en Grafana se comprueba que los tableros muestren los datos que se acaban de generar (puede tardar
  unos minutos y se ve solo el negocio al que se tiene permiso).
- Si el menú no muestra una opción, falta un permiso del rol: pedirlo al equipo, no buscar otra ruta.

## Antes de empezar

1. **Un negocio (tenant) de pruebas**, nunca uno con datos reales (el que te asignaron en el equipo). Si el despliegue acaba de cambiar, reiniciar ADA: los
   textos de las pantallas se cargan al arrancar.
2. **Un usuario con permisos** sobre Facturación (Separados, Cobro de cartera, Caja) e Inventario (Compras, Cuentas por
   pagar). Si una opción no aparece en el menú, falta el permiso del rol.
3. **Datos base**: un cliente, un proveedor, tres productos con stock, un medio de pago en efectivo y uno electrónico
   (tarjeta o transferencia), y al menos una **caja** abierta para el usuario.
4. **Períodos contables abiertos** para la fecha de hoy. Sin período, las operaciones se guardan igual, pero su asiento
   queda en *Pendientes de contabilizar* (que también se prueba, ver los flujos).
5. Anotar las cifras iniciales (stock, saldo del proveedor, saldo del cliente, total de la caja) antes de cada flujo: todos
   los resultados esperados se expresan como cambios respecto a ellas.

## Cómo leer un flujo

- Cada paso tiene **Acción** y **Esperado**. Si lo esperado no ocurre, el flujo se detiene y se reporta.
- Los montos de los ejemplos son redondos a propósito; usar los mismos para que los resultados coincidan con los del documento.
- Ruta de menú entre comillas, por ejemplo *Inventario → Cuentas por pagar*.

## Cómo reportar un fallo

Un reporte útil lleva: flujo y número de paso, qué se hizo, qué se esperaba, qué pasó, captura de pantalla, hora aproximada
y el negocio (tenant) y usuario usados. Con la hora se puede encontrar el error en el servidor.

## Mantener estos documentos al día

- Un flujo se actualiza en el mismo cambio que modifica la pantalla o la regla que prueba.
- Cada documento indica al final su **última revisión**; si tiene más de seis meses sin pasar por QA, tratarlo como dudoso.
- Esta carpeta se publica en el repositorio público de documentación: no poner nombres de clientes, direcciones, contraseñas ni
  datos reales de ningún negocio.
