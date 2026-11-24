# Cómo entrar

> **Una dirección. Una contraseña. El resto lo eliges tú.**
>
> Esta página es para cualquiera: no hace falta saber de sistemas. Si vienes a probar el producto, entra primero y
> sigue con el [índice de QA](INDEX.md).

## 📍 La llave

### `ada.sotobotero.com`

**Es la única dirección que hay que recordar.** Escríbela en el navegador y ya estás en la puerta. Funciona igual en
computador, tableta y celular.

> [!TIP]
> **¿Te llegó un correo del equipo?** El botón del correo te deja en esa misma puerta. No hay que escribir nada.

<!-- Mantenimiento: si cambia el dominio, además de la tabla de "Las direcciones" hay que
     cambiar las líneas `click` de los dos diagramas de este archivo (mermaid no lee variables). -->

```mermaid
---
title: ¿Cómo llego desde mi correo (o mi navegador) hasta lo que necesito hacer?
config:
  securityLevel: loose
---
flowchart LR
    A["📬 Abre el correo<br/>de bienvenida"]:::camino
    B["🧭 O escribe<br/>ada.sotobotero.com"]:::camino
    L["🔐 Escribe tu usuario<br/>y contraseña"]:::acceso
    Q{"¿Qué vas a hacer?"}:::acceso
    M["📱 Pulso Mobile<br/>atender y cargar"]:::app
    K["🖥️ Backoffice<br/>facturar y cuadrar"]:::app
    S["📊 Sirius<br/>mirar tableros"]:::app

    A -->|pulsa el enlace| L
    B -->|entra| L
    L --> Q
    Q -->|"atiendo al público"| M
    Q -->|"cuadro las cuentas"| K
    Q -->|"miro los números"| S

    classDef camino fill:#f5f8ff,stroke:#c3d4f0,stroke-width:1px,color:#1e3a8a
    classDef acceso fill:#f7f5ff,stroke:#cfc3f2,stroke-width:1px,color:#4c1d95
    classDef app fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b

    click B "https://ada.sotobotero.com/"
    click M "https://ada.sotobotero.com/" "Abrir Pulso Mobile"
    click K "https://ada.sotobotero.com/ADA_ENTERPISE_CORE/" "Abrir el Backoffice"
    click S "https://ada.sotobotero.com/grafana/" "Abrir Sirius"
```

**El correo es del primer día; la dirección, de todos los días.** Y una sola cuenta para todo: si ya entraste, cambiar
de pantalla no te vuelve a pedir la contraseña. Si tienes varios negocios, eliges al entrar.

## 🆘 Si algo no funciona

Busca tu síntoma arriba; la salida está debajo. Arriba, en pequeño lo que te pasa; abajo, en grande lo que haces.

```mermaid
---
title: Si algo no funciona, encuentra tu síntoma y su salida
config:
  securityLevel: loose
---
flowchart TB
    A1["🚪 La contraseña<br/>no me acepta"]:::fallo
    A2["🚪 No llega<br/>el correo"]:::fallo
    B1["👀 Falta una opción<br/>en el menú"]:::fallo
    B2["👀 No aparecen<br/>mis negocios"]:::fallo
    C1["🖥️ La pantalla<br/>queda en blanco"]:::fallo
    R1["Restablécela<br/>desde la pantalla<br/>de acceso"]:::arreglo
    R2["Revisa spam y pide<br/>que lo reenvíen.<br/>No crees otra cuenta."]:::arreglo
    R3["Tu rol no la tiene:<br/>pídela al<br/>administrador"]:::arreglo
    R4["Tu cuenta quedó<br/>en otro negocio:<br/>avisa al equipo"]:::arreglo
    R5["Recarga; si sigue,<br/>repórtalo con<br/>la hora"]:::arreglo

    A1 --> R1
    A2 --> R2
    B1 --> R3
    B2 --> R4
    C1 --> R5

    classDef fallo fill:#fff8f1,stroke:#f3c9a0,stroke-width:1px,color:#8a4b16
    classDef arreglo fill:#f3faf4,stroke:#a7d9b4,stroke-width:1px,color:#1d4d2b

    click R1 "https://sso.sotobotero.com" "Abrir la pantalla de acceso"
```


[Reportar un fallo](INDEX.md#si-un-paso-no-da-lo-esperado) · [Ver los flujos de prueba](INDEX.md)

## 🚪 Las direcciones

> [!IMPORTANT]
> **Este es el único documento del repositorio que escribe direcciones.** Si el dominio cambia, se cambia solo esta
> tabla y todo lo demás sigue funcionando: los otros documentos enlazan aquí.

| Adónde vas | Producción | Ensayo |
|---|---|---|
| 📱 **S&B Pulso Mobile** — tarjetas y botones grandes; se usa en el celular, para atender al público | [ada.sotobotero.com](https://ada.sotobotero.com/) | [stg-ada.sotobotero.com](https://stg-ada.sotobotero.com/) |
| 🖥️ **S&B Pulso Backoffice** — la pantalla clásica, con muchos campos; en el computador, para facturar y cuadrar cuentas | [ada.sotobotero.com/ADA_ENTERPISE_CORE/](https://ada.sotobotero.com/ADA_ENTERPISE_CORE/) | [stg-ada…/ADA_ENTERPISE_CORE/](https://stg-ada.sotobotero.com/ADA_ENTERPISE_CORE/) |
| 📊 **BI S&B Sirius** — tableros con gráficas; se abre solo para mirar | [ada.sotobotero.com/grafana/](https://ada.sotobotero.com/grafana/) | — solo existe en producción |
| 🔧 **Administración de plataforma** — crear y aprobar negocios (equipo de plataforma) | [ada.sotobotero.com/platform-admin](https://ada.sotobotero.com/platform-admin) | [stg-ada…/platform-admin](https://stg-ada.sotobotero.com/platform-admin) |
| ✍️ **Alta de negocio** — el formulario público para registrarse | [ada.sotobotero.com/empezar](https://ada.sotobotero.com/empezar) | [stg-ada…/empezar](https://stg-ada.sotobotero.com/empezar) |
| 🔑 **Acceso** — la pantalla de la contraseña | [sso.sotobotero.com](https://sso.sotobotero.com) | la misma |
| 🛒 **Sitio del negocio** — el marketplace público: catálogo, carrito y seguimiento del pedido | `<negocio>.sotobotero.com` | el negocio de pruebas: [la-espiga-dora.sotobotero.com](https://la-espiga-dora.sotobotero.com) · el marketplace Toki: [stg-toki.sotobotero.com](https://stg-toki.sotobotero.com) |

En el marketplace la API vive bajo `/ADA_ENTERPISE_CORE/`: `https://la-espiga-dora.sotobotero.com/ADA_ENTERPISE_CORE/restapi/v1/public/...`.

**Mira la barra de direcciones antes de llenar algo:** si dice `stg-ada`, estás en ensayo; si dice `ada`, en producción.
Una prueba enviada en producción crea un negocio real.

<details>
<summary><strong>Para QA — qué se prueba en cada cara</strong></summary>

| Cara | Qué se prueba | Detalle |
|---|---|---|
| **Pulso Mobile** | inventario, reservas, mesas y pedidos del restaurante, catálogo del proveedor; se usa en celular | [flujos](INDEX.md#qué-probar-si-no-hay-un-negocio-concreto) |
| **el Backoffice** | facturación y caja, separados y cartera, compras y cuentas por pagar, contabilidad y reportes | [flujos](INDEX.md#qué-probar-si-no-hay-un-negocio-concreto) |
| **Sirius** | que los tableros muestren lo que se acaba de generar (puede tardar unos minutos; solo el negocio con permiso) | [índice de QA](INDEX.md) |
| **Administración** | alta, aprobación y aprovisionamiento de negocios (lo usa el equipo de plataforma) | [cómo se monta un restaurante](restaurante-configuracion.md) |

</details>

<details>
<summary><strong>Para QA — nombre técnico de la pantalla de acceso</strong></summary>

La pantalla de acceso es **Keycloak**, en `sso.sotobotero.com`. El usuario final no necesita ese nombre: para él es
«la pantalla de acceso». Los nombres de producto son **S&B Pulso Mobile**, **S&B Pulso Backoffice** y **BI S&B Sirius**
(ver `infraestructure-ascode/docs/brand/naming.md`).

</details>

**Última revisión:** 2026-11-19.
