# E2E — Aprobación del pedido público, plato agotado y carrito del catálogo

> **Automático.** Las dos escenas las corre Playwright sobre el arnés del restaurante
> (`tests/playwright/restaurant/`): ADA de este checkout, el tenant de pruebas,
> Keycloak local, el shell con `mf-restaurant` y el catálogo público con `mf-booking`. La tabla de escenas del final se marca a
> mano cuando alguien las corre; cada fila dice qué spec la prueba.
>
> Piezas del plan `work/restaurante-pendientes`: **aprobador del pedido público** (pieza 1) y **plato agotado** (pieza 2).

| Spec | Qué prueba |
|---|---|
| `restaurant-order-approval.spec.ts` | Un cliente **anónimo** pide desde el catálogo público; el pedido nace esperando aprobación y **no entra a la cocina**; la bandeja `/restaurante/por-aprobar` muestra la fila con su total y «Solicitud anónima del catálogo»; al abrirla se ven cliente, líneas y total. **Aprobar** la deja aprobada y el pedido entra a la cola de cocina. Un segundo pedido se **rechaza**: sin motivo no deja confirmar y con motivo queda cancelado. Un tercero se aprueba con **«Aprobar y cobrar»** (el bloque de pago compartido) y queda pagado, con su factura. |
| `restaurant-dish-soldout.spec.ts` | Desde la cocina se marca el plato agotado **con hora**: la tarjeta queda con `dish-soldout-badge-<productId>` y la hora. En el catálogo público el plato **se sigue viendo** con `catalog-soldout-<id>`, «Pedir» deshabilitado y `order-soldout-blocked` al intentarlo. «Disponible ya» lo devuelve y el plato se vuelve a pedir. **Sin hora** (campo vacío) queda agotado igual y solo el botón lo devuelve. El pedido de mesa que usa se marca y se cancela al final. |

Un caso feliz y un caso de error por pieza: el error de la pieza 1 es el **rechazo sin motivo** (no confirma, no envía nada) y el
de la pieza 2 es **pedir un plato agotado** (el diálogo lo dice y el botón de pedir queda apagado).

## Datos que necesita

Todo es del tenant de pruebas `e2e_restaurant`; nunca de staging ni de producción. El arnés y el stack completo están en
`tests/playwright/restaurant/README.md`.

| Dato | Valor por defecto | Cómo se prepara |
|---|---|---|
| Tenant de pruebas | `e2e_restaurant` | `bash tests/playwright/restaurant/support/recreate-db.sh` |
| Producto **publicado en el catálogo, con precio** y acción «Pedir» | título que contiene `PW_CATALOG_DISH` (`Arroz con`) | se publica desde el negocio (catálogo/ADA); el spec lo busca por ese texto |
| Plato para el switch de cocina | `PW_SOLDOUT_DISH_ID` (`31`), `PW_SOLDOUT_DISH` (`Arroz con`) | un plato que el negocio venda y que tenga stock para una ronda |
| El plato del catálogo del agotado | `PW_CATALOG_DISH_TITLE` (cae a `PW_SOLDOUT_DISH`) | el mismo plato publicado, con «Pedir» |
| Usuario con `ORDER_APPROVE` | `root` (la migración 326 lo da al rol `root`) | el login del arnés; sin el permiso la bandeja no lista ni decide |
| Usuario **sin** `ORDER_APPROVE` | uno del equipo, sin el permiso | escena manual: un usuario se crea por el provisioning de empleados (Keycloak + `ada_master` + roles), no insertando filas |
| Host público del tenant | `e2e.localhost` (`PW_PUBLIC_HOST`) | una fila en el `ada_master.hosts` local (zona `public`) que lo mapee a `e2e_restaurant`; el catálogo anónimo toma el tenant del host |
| `mf-booking` | `4302` (`PW_MF_BOOKING_PORT`) | `cd arcadia/frontend/mf-booking && npm start -- --port 4302 --live-reload=false` |
| Migraciones | 326 (estado y permiso de aprobación), 285–288 (catálogo y hosts) | `recreate-db.sh` reconstruye el tenant desde la plantilla |

**Datos que crean los specs, marcados para no ensuciar el tenant:** los tres pedidos del aprobador llevan
`QA e2e <stamp> N` en el cliente y `e2e <stamp> N` en la nota para la cocina; el aprobado queda abierto en la cocina, el
rechazado cancelado y el cobrado pagado. El pedido del agotado se cancela al terminar y el plato queda disponible.

## Comandos

```bash
cd tests/playwright

# 1. Que las specs parsean y se listan (no necesita ningún servidor)
npm run test:restaurant:list

# 2. Las dos piezas, con el stack local del arnés levantado
npm run test:restaurant:approval
npm run test:restaurant:soldout
```

Sin el stack levantado no se corren: el primer comando (`--list`) es el que se puede verificar siempre.

## Escenas

| # | Escena | Esperado | Spec | ✔ |
|---|---|---|---|---|
| 1 | El cliente pide desde el catálogo público | El pedido queda esperando aprobación y la **cola de cocina no lo muestra** | `restaurant-order-approval` | ☐ |
| 2 | El aprobador abre `/restaurante/por-aprobar` | La fila del pedido aparece con su **total** y la etiqueta «Solicitud anónima del catálogo» | `restaurant-order-approval` | ☐ |
| 3 | Abre la fila | Se ven **cliente, líneas y total** del pedido | `restaurant-order-approval` | ☐ |
| 4 | «Aprobar» | Queda aprobada y el pedido **entra a la cola de cocina** (base: `ORDER_OPEN`) | `restaurant-order-approval` | ☐ |
| 5 | «Rechazar» **sin motivo** (caso de error) | No deja confirmar: avisa que el motivo es obligatorio y no se envía nada | `restaurant-order-approval` | ☐ |
| 6 | «Rechazar» **con motivo** | Queda rechazada y el pedido **cancelado** (base: `ORDER_CANCELLED`) | `restaurant-order-approval` | ☐ |
| 7 | «Aprobar y cobrar» eligiendo **un medio de pago** (la pantalla no escribe el importe) | Queda **pagado**, con su factura confirmada por el total de la factura, y el inventario ya salió (base: `ORDER_PAID`, factura existente) | `restaurant-order-approval` | ☐ |
| 8 | La cocina marca el plato agotado **con hora** | La tarjeta queda con `dish-soldout-badge-<productId>` y la **hora** | `restaurant-dish-soldout` | ☐ |
| 9 | El catálogo público, con el plato agotado | El plato **se sigue viendo**, con `catalog-soldout-<id>` y «Pedir» deshabilitado | `restaurant-dish-soldout` | ☐ |
| 10 | Intenta pedir el plato agotado (caso de error) | El diálogo muestra `order-soldout-blocked` y el botón de pedir está apagado | `restaurant-dish-soldout` | ☐ |
| 11 | «Disponible ya» | La tarjeta pierde el aviso y **el plato se vuelve a pedir** desde el catálogo | `restaurant-dish-soldout` | ☐ |
| 12 | Se marca agotado **sin hora** (campo vacío) | Queda agotado igual (aviso sin hora) y **solo el botón** lo devuelve | `restaurant-dish-soldout` | ☐ |
| 13 | Un usuario **sin** `ORDER_APPROVE` abre la bandeja (manual) | No la lista ni puede decidir: la pantalla no le da la opción y el backend responde 403 | — (manual) | ☐ |
| 14 | Arma un carrito con **dos platos distintos** desde el catálogo público | «Carrito» cuenta 2, se ven las dos líneas con sus adicionales y el **domicilio aparece una sola vez** en el total | manual (carrito) | ☐ |
| 15 | Quita un plato del carrito | La línea desaparece y el total baja en el acto | manual (carrito) | ☐ |
| 16 | Quita **la última** línea | El panel **se cierra solo** (no queda una pantalla vacía abierta) | manual (carrito) | ☐ |
| 17 | Cierra el pedido desde el carrito con sus datos | Sale **un** pedido con las dos líneas, aparece el enlace de seguimiento y el carrito queda vacío | manual (carrito) | ☐ |
| 18 | Recarga la página con el carrito armado | El carrito sigue ahí (se guarda en el navegador) y el contador cuadra | manual (carrito) | ☐ |
| 19 | Un plato **agotado** en el catálogo | No se puede agregar al carrito: el aviso manda y el botón está apagado | manual (carrito) | ☐ |
| 20 | Cambia de «A domicilio» a «Recoger» con el pedido armado | El pedido **no se vacía**: siguen los platos y solo se ajusta el domicilio, con un aviso de qué se cambió | manual (carrito) | ☐ |
| 21 | Con sesión iniciada, abre el carrito | Los datos de contacto salen **precargados y plegados**; «editar datos» los abre para ese pedido | manual (carrito) | ☐ |
| 22 | Elige la dirección en el mapa | El punto queda en el mapa y la dirección elegida es la del pedido | manual (carrito) | ☐ |

## Límites conocidos

- Solo `root` inicia sesión en el arnés, así que la escena 13 (aprobador **sin** `ORDER_APPROVE`) es manual: ese usuario sale del
  provisioning de empleados, no de insertar filas. Su caso negativo de backend está cubierto por las pruebas de ADA.
- La escena 8 compara la hora que el spec envía (`18:30` del día siguiente) con la que muestra la tarjeta. Si la sucursal del
  tenant vive en otra zona, es la primera aserción que hay que mirar.
- Las escenas 1–7 necesitan un plato publicado con «Pedir»; si el tenant de pruebas no lo tiene, el spec falla en la búsqueda del
  catálogo con un mensaje claro y **no** inventa datos.

**Última revisión:** 2026-11-24.
