# Probar lo que hace la cocina

> **Prepara, avisa y responde lo que nadie más atiende.** Pantalla **Cocina**, con permiso `KITCHEN` y **sin** permiso
> de mesas. Las direcciones, en [00-acceso.md](00-acceso.md). Antes: lo que deja listo [el dueño](e2e-restaurante-dueno.md).

```mermaid
---
title: Qué le llega a la cocina de una mesa
---
flowchart LR
    M{"🟣 ¿La mesa<br/>tiene mesero?"}:::sistema
    SI["Solo los platos<br/>ya confirmados"]:::contexto
    NO["Platos, llamadas<br/>y por confirmar"]:::haz

    M -->|"sí"| SI
    M -->|"no"| NO

    classDef sistema fill:#f7f5ff,stroke:#cfc3f2,stroke-width:1px,color:#4c1d95
    classDef contexto fill:#f4f5f7,stroke:#cfd4dc,stroke-width:1px,color:#374151
    classDef haz fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
```

## 🧰 Antes de empezar

| Necesitas | Para qué escena |
|---|---|
| Mesa A con mesero y mesa B sin mesero | K-05 a K-10 |
| Un teléfono (o ventana privada) con el enlace de cada una | K-05 a K-10 |
| Dos sucursales con una mesa del mismo nombre | K-11 |

## 🔥 La cola

| # | Haz | Debe pasar | ✔ |
|---|---|---|---|
| K-01 | Un mesero envía una ronda | Aparece una tarjeta **por pedido**, con sonido, en máximo 5 segundos | ☐ |
| K-02 | **Empezar** y luego **Listo** en cada plato | La tarjeta pasa a *Listos para servir* | ☐ |
| K-03 | Pliega una tarjeta y espera la siguiente lectura | Sigue plegada, con el resumen de platos | ☐ |
| K-04 | Corta la red un momento | *Sin conexión*; se conserva lo último y se recupera solo | ☐ |

## 🔔 Mesas sin mesero

| # | Haz | Debe pasar | ✔ |
|---|---|---|---|
| K-05 | Llama al mesero desde **B** | *Mesa B llama* en Cocina y en Mesas | ☐ |
| K-06 | Llama desde **A** | **No** aparece ni suena en Cocina | ☐ |
| K-07 | **Atendido** en la llamada de B | Se quita de Cocina y de Mesas | ☐ |
| K-08 | El comensal pide desde B y la cocina **Confirma** | Entra a la cola y la mesa queda **sin mesero** | ☐ |
| K-09 | Vuelve a llamar desde B | Sigue llegando a Cocina | ☐ |
| K-10 | El comensal pide desde A | **No** aparece en Cocina; sí en Mesas | ☐ |

## 🏬 Sucursales y agotados

| # | Haz | Debe pasar | ✔ |
|---|---|---|---|
| K-11 | Elige **Todas las sucursales** | Cada tarjeta y llamada dice su sucursal | ☐ |
| K-12 | Cambia a una sola | La cola se vuelve a leer sin sonar como pedido nuevo | ☐ |
| K-13 | **Marcar agotado** un plato, con hora y motivo | En la carta del mesero, el aviso cabe en la tarjeta en un renglón; en la de la mesa sale agotado y no se puede pedir | ☐ |
| K-14 | **Disponible ya** | Se vuelve a pedir | ☐ |

## 🩺 Si algo falla

```mermaid
---
title: Qué revisar según el síntoma
---
flowchart TB
    S1["🟠 No llegan<br/>llamadas"]:::mal
    S2["🟠 Faltan platos<br/>en la cola"]:::mal
    A1["La mesa tiene mesero:<br/>revisa en Configurar"]:::haz
    A2["Mira la sucursal<br/>elegida arriba"]:::haz

    S1 --> A1
    S2 --> A2

    classDef mal fill:#fff8f1,stroke:#f3c9a0,stroke-width:1px,color:#8a4b16
    classDef haz fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b
```

El agotado en el catálogo público, con prueba automática, en
[aprobación y plato agotado](e2e-restaurante-aprobacion-y-agotado.md). Las escenas de esta página son manuales.

**Dueño del documento:** equipo de restaurante. **Última revisión:** 2026-11-25.
