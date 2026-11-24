# Cómo probar el sistema

> Se prueba con el negocio que se tiene delante. Si vienes a ver **un negocio concreto**, la primera tabla te dice qué
> correr y en qué orden; si no, las guías vigentes van en el orden en que se aprenden. Cada documento es un flujo aparte,
> con pasos numerados, datos de ejemplo y lo que tiene que pasar en cada paso.

> [!IMPORTANT]
> **Paso 0, siempre: [cómo entrar y qué cara es cada una](00-acceso.md).** Ahí están la dirección, el login y qué pantalla
> se usa para qué. Este índice no repite direcciones: si cambia el dominio, se cambia solo allí.

## Si vienes a probar un negocio concreto

| El negocio es… | Corre esto, en este orden |
|---|---|
| **Restaurante, cafetería o bar** | [guia-restaurante.md](guia-restaurante.md) · [montarlo](restaurante-configuracion.md) · [aprobar el pedido del catálogo y el plato agotado](e2e-restaurante-aprobacion-y-agotado.md) |
| **Manufactura, confección o algo que pase por estaciones** | [guia-produccion-etapas.md](guia-produccion-etapas.md) → [conteo de inventario](e2e-conteo-de-inventario.md) *(heredado)* |
| **Joyería, ropa, ferretería o tienda** | [producto y venta](e2e-happy-path-inventario-facturacion.md) *(heredado)* → [separados y cartera](e2e-cartera-separados.md) *(heredado)* → [conteo de inventario](e2e-conteo-de-inventario.md) *(heredado)* |
| **Clínica u odontología** | [historia clínica y RIPS](e2e-historia-clinica-rips.md) → [separados y cartera](e2e-cartera-separados.md) *(heredado)* → [la caja cuadra](e2e-reporte-de-caja.md) *(heredado)*. El módulo clínico solo vive en el Backoffice |
| **Gimnasio o centro de servicios** | [reservas y membresías](e2e-gym-manual-tests.md) *(heredado)* → [separados y cartera](e2e-cartera-separados.md) *(heredado)* |
| **Proveedor que vende en un marketplace** | [producto y venta](e2e-happy-path-inventario-facturacion.md) *(heredado)* → [cargas de archivo](e2e-cargas-archivo-demo.md) *(heredado)* |

Si el negocio no encaja en ninguna fila, corre **[el ciclo completo](e2e-ciclo-completo.md)**: es el flujo que más vale,
porque si ese cuadra, los módulos están hablando entre ellos.

## Qué probar, si no hay un negocio concreto

Estas son las guías **vigentes**: ya están escritas con el estándar de hoy (guía corta, un diagrama por idea, pasos con
*acción* y *esperado*, y salida para cada fallo).

| # | Qué vas a probar | Documento |
|---|---|---|
| 1 | **Entrar y saber qué cara es cada una** | [00-acceso.md](00-acceso.md) |
| 2 | **Montar y atender un restaurante**: carta, mesas, cocina, cobro | [guia-restaurante.md](guia-restaurante.md) · [montarlo](restaurante-configuracion.md) |
| 3 | **Seguir una prenda por la planta**: ruta, paquetes, etapas y WIP | [guia-produccion-etapas.md](guia-produccion-etapas.md) |
| 4 | **El día completo del negocio**, en lenguaje llano | [guia-ciclo-del-negocio.md](guia-ciclo-del-negocio.md) |
| 5 | **Historia clínica y RIPS**: la cita abre la atención, las notas no se reescriben (corrección y anulación firmadas), consentimiento y anexos, y el corte de RIPS con y sin factura | [e2e-historia-clinica-rips.md](e2e-historia-clinica-rips.md) |
| 6 | **Aprobación del pedido público y plato agotado**: el pedido del catálogo no entra a cocina hasta que alguien lo aprueba (aprobar, rechazar con motivo, aprobar y cobrar), y el plato agotado se sigue viendo en el catálogo sin poder pedirse | [e2e-restaurante-aprobacion-y-agotado.md](e2e-restaurante-aprobacion-y-agotado.md) |

## Flujos heredados (regresión)

Los ocho flujos de la primera ola: siguen sirviendo de **checklist de regresión** y se corren tal cual, anotando lo que
falle, pero su forma —pasos largos, datos de ejemplo atados a una base concreta— ya no es la que se usa para lo nuevo. Al
reescribir uno, se sube a la tabla de arriba y se borra de esta; [guia-produccion-etapas.md](guia-produccion-etapas.md) es
el molde.

```mermaid
---
title: El orden en que se probaban
---
flowchart LR
    A["🧾 1 · Producto y venta<br/>inventario y facturación"]:::paso
    B["💳 2 · Separados y compras<br/>cartera y cuentas por pagar"]:::paso
    C["🧮 3 · La caja cuadra<br/>el reporte del día"]:::paso
    D(["🏁 4 · El ciclo completo<br/>todo junto, de punta a punta"]):::meta

    A --> B --> C --> D

    classDef paso fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
    classDef meta fill:#f7f5ff,stroke:#cfc3f2,stroke-width:1px,color:#4c1d95
```

| # | Qué queda probado | Documento |
|---|---|---|
| 1 | **Producto y venta**: crear el producto, entrada de stock, venta con tres medios de pago, rectificativa y cierre de caja | [e2e-happy-path-inventario-facturacion.md](e2e-happy-path-inventario-facturacion.md) |
| 2 | **Separados y cartera**: separado con anticipo, abonos, entrega, cancelación, venta fiada, límite de crédito y cobro | [e2e-cartera-separados.md](e2e-cartera-separados.md) |
| 3 | **Compras y cuentas por pagar**: compra de contado, a crédito y mixta; abono y reverso; devolución al proveedor; saldo a favor; cupo | [e2e-compras-cuentas-por-pagar.md](e2e-compras-cuentas-por-pagar.md) |
| 4 | **La caja cuadra**: que el reporte refleje lo que de verdad entró y salió, venga del flujo que venga | [e2e-reporte-de-caja.md](e2e-reporte-de-caja.md) |
| 5 | **El ciclo completo**: compra → venta → separado → fiado → cartera → pago y devolución, con inventario, caja y saldos conciliados | [e2e-ciclo-completo.md](e2e-ciclo-completo.md) |
| 6 | **Conteo de inventario**: cargar un almacén que no estaba en el sistema, unidades sin soporte y conteo de auditoría | [e2e-conteo-de-inventario.md](e2e-conteo-de-inventario.md) |
| 7 | **Gimnasio**: reservas y membresías de punta a punta | [e2e-gym-manual-tests.md](e2e-gym-manual-tests.md) |
| 8 | **Cargas de archivo**: plantillas, filas rechazadas que se corrigen | [e2e-cargas-archivo-demo.md](e2e-cargas-archivo-demo.md) |

El 5 es el que más vale, y el 1 a 4 se pueden correr sueltos: el 5, solo después.

> [!NOTE]
> **Heredado no es «no sirve».** Es un flujo que se corre igual y se anota lo que falle; lo que cambia es la forma en que
> está escrito, no la validez de lo que prueba.

## Cómo se lee un flujo

Un flujo son pasos con **Acción** y **Esperado**: se hace lo que dice la acción y se compara con lo esperado. Si no
coincide, el flujo **se detiene ahí** y se reporta; no se sigue para ver si el resto cuadra, porque todo lo de después
arrastra el error.

Los montos son redondos a propósito. Ruta de menú entre comillas, por ejemplo *Inventario → Cuentas por pagar*.

## Si un paso no da lo esperado

Se reporta con estos seis datos y con eso el fallo se encuentra:

| Dato | Por qué |
|---|---|
| flujo y número de paso | ubica la parte del sistema |
| qué hiciste | reproduce el camino |
| qué esperabas | compara contra la regla escrita |
| qué pasó de verdad | el síntoma, sin interpretarlo |
| captura y hora aproximada | la hora lleva al error en el servidor |
| negocio y usuario usados | sin ellos el error no se reproduce |

## Preparar los datos antes de empezar

<details>
<summary><strong>Cinco cosas que tienen que existir</strong></summary>

1. **Un negocio de pruebas**, nunca uno con datos reales: lo entrega el equipo.
2. **Un usuario con permisos** sobre lo que se va a probar. Si una opción no aparece en el menú, falta el permiso: se
   pide, no se busca otra ruta.
3. **Datos base**: un cliente, un proveedor, tres productos con stock, un medio de pago en efectivo y uno electrónico, y
   una **caja abierta** para el usuario.
4. **Períodos contables abiertos** para hoy; sin período la operación se guarda pero su asiento queda en *Pendientes de
   contabilizar* (que también se prueba).
5. **Las cifras iniciales anotadas** (stock, saldos, total de caja): los resultados esperados se expresan como cambios
   respecto a ellas.

</details>

## Presentar el producto a un cliente

<details>
<summary><strong>Tres guiones que no son pruebas</strong></summary>

| Guion | Qué muestra | Documento |
|---|---|---|
| **Ciclo del negocio** | un día completo en lenguaje llano, con cuatro videos cortos | [guia-ciclo-del-negocio.md](guia-ciclo-del-negocio.md) |
| **Cargas de archivo** | plantillas, productos con solo el nombre, filas rechazadas que se corrigen | [e2e-cargas-archivo-demo.md](e2e-cargas-archivo-demo.md) |
| **Restaurante** (histórico) | la historia escena por escena; se conserva como evidencia | legacy/e2e-restaurante-demo.md (interno: `docs/legacy/e2e-restaurante-demo.md`) |

</details>

## Para preparar un negocio desde cero

<details>
<summary><strong>Guías de arranque</strong></summary>

| Guía | Para qué |
|---|---|
| [INVENTORY_ONBOARDING_STRATEGY.md](INVENTORY_ONBOARDING_STRATEGY.md) | cargar el inventario de un negocio que abre sin sistema |
| [guia-ciclo-del-negocio.md](guia-ciclo-del-negocio.md) | el día del negocio completo, para el dueño |

</details>

## Mantener esto al día

- Un flujo se actualiza en el mismo cambio que modifica la pantalla o la regla que prueba.
- Un flujo heredado que se reescribe sube a la tabla de vigentes; la tabla de heredados solo se acorta.
- Cada documento cierra con su **última revisión**; con más de seis meses, se trata como dudoso.
- **Las direcciones viven solo en [00-acceso.md](00-acceso.md).** Al escribir, las caras se nombran por su nombre: Pulso
  Mobile, Pulso Backoffice, Sirius.
- Cómo se escribe uno de estos documentos: `ai-workspace/DESIGN.md` (jerarquía, verbos del usuario, fuente única para lo
  que cambia y la prueba de los 5 segundos). Un documento que no lo pase se rehace antes de publicarlo.
- Esta carpeta se publica en el repositorio público de documentación: no lleva nombres de clientes, direcciones,
  contraseñas ni datos de ningún negocio.
