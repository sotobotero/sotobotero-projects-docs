# Cómo se pone a vender el restaurante

> **Se monta, se atiende y el inventario se mueve solo.** Esta guía es el mapa completo: qué dejar listo antes de
> abrir, quién hace cada cosa durante el servicio y por qué camino el inventario se descuenta sin que nadie lo digite.
>
> Las direcciones y el acceso están en [00-acceso.md](00-acceso.md). Lo que hay que dejar montado antes de abrir, en
> [cómo dejar listo el restaurante](restaurante-configuracion.md).

## 🗺️ La foto completa

Cuatro bloques. El primero lo deja listo el dueño; los dos siguientes los hace el equipo cada día; el último no lo
hace nadie.

```mermaid
---
title: Los cuatro bloques del restaurante, y quién hace cada uno
---
flowchart LR
    M["🏗️ 1. Se monta<br/>carta, fotos, precios<br/>y recetas<br/>el dueño"]:::monta
    A["🪑 2. Se atiende<br/>mesa, pedido<br/>y cobro<br/>el mesero"]:::atiende
    C["👨‍🍳 3. La cocina<br/>prepara<br/>y marca<br/>el cocinero"]:::atiende
    Q["📦 4. Queda hecho<br/>kardex y caja<br/>sin digitar<br/>lo hace el sistema"]:::sistema

    M --> A --> C --> Q

    classDef monta fill:#f5f8ff,stroke:#c3d4f0,stroke-width:1px,color:#1e3a8a
    classDef atiende fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
    classDef sistema fill:#f7f5ff,stroke:#cfc3f2,stroke-width:1px,color:#4c1d95

    click M "restaurante-configuracion.md" "Ver cómo se monta"
    click A "restaurante-configuracion.md" "Quién hace cada cosa"
    click C "restaurante-configuracion.md" "Quién hace cada cosa"
```

Ver el paso a paso: [cómo dejar listo el restaurante](restaurante-configuracion.md).

## 🏗️ Antes de abrir: lo que deja listo el dueño

Tres cosas, y sin las tres no hay servicio:

- **Qué se publica.** Los grupos de la carta se encienden por canal, por sede o por mesa; lo que no se enciende no
  aparece en el celular del mesero. Detalle: [qué se publica](restaurante-configuracion.md).
- **Fotos y precios.** Cada plato es una tarjeta con foto, precio y grupo. Sin foto se vende peor y el mesero duda.
- **Recetas.** Qué gasta cada plato. Es lo que hace que el inventario baje solo: los tres caminos están abajo, y el
  detalle en cómo se arma un plato (interno: `docs/RECIPES_AND_PRODUCTION.md`).
- Y **una caja abierta** antes del primer cobro.

## 🪑 Durante el servicio: lo que hace el mesero

- Abre la mesa **sin escribir nada**: los platos más pedidos salen primero, en tarjetas.
- Cada ronda se envía a cocina y **lleva su número**: la segunda no se mezcla con la primera.
- Dos toques por plato, con una mano y sin teclado.

## 🗓️ Las reservas que llegan del marketplace

El cliente reserva desde el sitio del negocio y la reserva **entra como un pedido**, no como una nota aparte. Hasta que
alguien la sienta, espera en **Mesas → Reservas**.

```mermaid
---
title: De la reserva en el sitio a la mesa ocupada
---
flowchart LR
    W["🌐 El cliente reserva<br/>en el sitio del negocio"]:::web
    L["🗓️ Queda por sentar<br/>en Mesas → Reservas"]:::paso
    S["🪑 Se sienta en una<br/>mesa libre"]:::paso
    O["📱 El pedido abre<br/>como cualquier otro"]:::paso

    W --> L --> S --> O

    classDef web fill:#f7f5ff,stroke:#cfc3f2,stroke-width:1px,color:#4c1d95
    classDef paso fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
```

- La lista muestra **quién reservó, a qué hora y cuántos comensales**. Sin fecha se ven todas las de hoy en adelante; con
  fecha, solo ese día.
- **Sentar** pide una mesa libre y la ocupa. Desde ese momento el pedido es uno normal: se toma, va a cocina y se cobra.
- Una reserva no ocupa la mesa por sí sola: **la mesa se ocupa cuando se sienta**.

> [!IMPORTANT]
> Para que lleguen reservas tiene que estar cargado el servicio *Reserva* (un producto de tipo servicio con empleado y
> turno). Sin eso, el sitio del negocio no ofrece reservar.

## 👨‍🍳 La cocina

```mermaid
---
title: Los estados de un plato, de la comanda al cliente
---
flowchart LR
    P(["🕐 Pendiente"]):::encurso --> R(["🔥 Preparando"]):::encurso
    R --> L(["✅ Listo"]):::listo
    L --> S(["🍽️ Servido"]):::listo
    P -.->|"con motivo"| X(["❌ Cancelado"]):::malo
    R -.->|"con motivo"| X

    classDef encurso fill:#fff8f1,stroke:#f3c9a0,stroke-width:1px,color:#8a4b16
    classDef listo fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
    classDef malo fill:#fdf4f4,stroke:#e7b3b3,stroke-width:1px,color:#7f1d1d
```

La cocina ve solo lo que le toca, en el orden en que llegó. El salón se entera **sin gritar**: el mesero ve el plato
pasar a *listo* en su propio celular. Cancelar pide motivo, y queda quién y por qué.

## 🧾 El cobro

| Cliente | Qué sale |
|---|---|
| No pide factura | **ticket de venta**, con los medios de pago |
| Pide factura electrónica | el cliente se crea en la mesa y sale el documento DIAN; el documento es obligatorio |

- Con un plato **sin servir**, la cuenta no se puede pedir: el botón está bloqueado y dice por qué.
- La **propina** viene sugerida por la caja, la decide el cliente, se quita con un toque y va **aparte** de la factura.
- Al cobrar, la mesa queda libre.

## 🧮 Tres caminos para que el inventario baje solo

Se elige según cómo se cocine el plato. Los tres terminan igual: nadie digita el inventario al cierre.

```mermaid
---
title: De qué depende que el inventario baje, según cómo se prepare el plato
---
flowchart TB
    T1["🍲 Se prepara por tandas<br/>antes del servicio"]:::prep
    T2["🥗 Se arma al momento<br/>con ingredientes"]:::prep
    T3["🥤 Se vende tal cual<br/>la botella, el paquete"]:::prep
    R1["Orden de producción:<br/>baja al cerrarla"]:::camino
    R2["Receta (producto compuesto):<br/>baja al vender"]:::camino
    R3["Su propio stock:<br/>baja al vender"]:::camino

    T1 --> R1
    T2 --> R2
    T3 --> R3

    classDef prep fill:#fff8f1,stroke:#f3c9a0,stroke-width:1px,color:#8a4b16
    classDef camino fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b

    click R1 "../RECIPES_AND_PRODUCTION.md" "Cómo se arma por tandas"
    click R2 "../RECIPES_AND_PRODUCTION.md" "Cómo se arma la receta"
```

- **Camino 1 — por tandas.** Se cocina una vez para todo el servicio y se cierra la **orden de producción**: bajan
  los insumos y suben las porciones listas. Ejemplos: arroz con pollo, ajiaco, salsa de la casa.
- **Camino 2 — receta (producto compuesto).** El plato se arma al vender y descuenta sus ingredientes, hasta seis
  niveles (una ensalada dentro de un combinado). Ejemplos: ensalada, plato combinado.
- **Camino 3 — sin receta.** La bebida o el paquete que se vende tal cual, con su propio stock: baja una unidad.

Ver el paso a paso de los caminos 1 y 2: cómo se arma un plato y cuándo baja el inventario (interno: `docs/RECIPES_AND_PRODUCTION.md`).

Un plato con **stock propio** corta la receta ahí: si ya hay porciones hechas, se descuenta la porción, no los
ingredientes. El ejemplo completo, paso a paso, en de la receta al kardex (interno: `docs/RECIPES_AND_PRODUCTION.md`).

## 📦 Lo que queda hecho sin que nadie lo digite

| Dónde | Qué se ve |
|---|---|
| Backoffice → *Inventario → Kardex* | el plato vendido descontó lo que gasta |
| Backoffice → *Reporte de caja* | lo cobrado por cada medio de pago, y la propina aparte |
| Pulso Mobile → *Pedidos → Pagadas* | las propinas del día por mesero |

**Si el kardex no bajó:** la venta tiene que estar cerrada (no en borrador) y el plato marcado para descontar. Si
bajó el plato en vez de sus ingredientes, es porque el plato tiene existencias propias.

## 🛒 De dónde sale la materia prima

La compra se registra en el Backoffice y queda como cuenta por pagar; desde ahí el kardex tiene qué descontar.
Detalle: [compras y cuentas por pagar](e2e-compras-cuentas-por-pagar.md).

<details>
<summary><strong>Lo que todavía no se promete</strong></summary>

| Tema | Estado | Qué decir |
|---|---|---|
| Domicilios | no está | el módulo atiende **mesa**; el domicilio se evalúa |
| Reservas | funciona | el cliente reserva en el sitio del negocio y el pedido espera en *Mesas → Reservas*; el negocio necesita el servicio *Reserva* cargado |
| Comandas impresas | construido | sale en papel de 80 mm; requiere el agente de impresión instalado en el local |

</details>

La guía de montaje está aparte: [cómo dejar listo el restaurante](restaurante-configuracion.md).

**Última revisión:** 2026-11-19.
