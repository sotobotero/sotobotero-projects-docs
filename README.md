# @sotobotero — Documentación de Proyectos

> Documentación para quien **usa** o **prueba** lo que hay montado, no para quien lo mantiene. Lo publican
> automáticamente los repositorios privados; aquí no hay código fuente.

## 🖨️ Instalar el agente de impresión

El agente local que corre en el PC del cajero y manda los trabajos a la impresora USB. Es lo único que se instala en el
equipo del cliente.

| Qué | Dónde |
|---|---|
| Instalador de Windows y Linux, con su `sha256` y el `print-agent.env.example` | [ada/infraesturadomain/print-agent/latest/](ada/infraesturadomain/print-agent/latest/) |
| Cómo funciona, cómo se instala y el detalle del papel de 80 mm | [ada/infraesturadomain/print-agent/README.md](ada/infraesturadomain/print-agent/README.md) |

## 🍽️ Usar el producto

Las guías están escritas para el equipo del negocio: sin jerga y con el recorrido del día.

| Guía | Para quién |
|---|---|
| [Cómo entrar](ada/qa/00-acceso.md) | cualquiera: una dirección, una contraseña y las tres caras del producto |
| [Cómo dejar listo el restaurante](ada/qa/restaurante-configuracion.md) | el dueño: mesas, carta, platos con foto y precio, personas y caja |
| [Cómo atender el salón](ada/qa/guia-restaurante.md) | mesero, cocina y quien cobra: pedido, cocina, cobro y reservas |

## 🧪 Probar y reportar

| Documento | Para qué |
|---|---|
| [Índice de pruebas](ada/qa/README.md) | los flujos en el orden que conviene probarlos, y cómo reportar un fallo |
| [Caso gimnasio](ada/qa/e2e-gym-manual-tests.md) | reservas y membresías de punta a punta |

## 🧱 Para quien desarrolla

| Documento | Tema |
|---|---|
| [Arquitectura de microfrontends](ada/frontend/MICROFRONTEND-CONFIG-ARCHITECTURE.md) | Module Federation, rutas entre shells y configuración |

## Lo que no está aquí

La documentación interna —arquitectura del backend, módulos, operación, aprovisionamiento de negocios— **no se publica**:
vive en los repositorios privados del ecosistema. Este repositorio solo reúne lo que necesita quien instala, usa o prueba
el producto.

## Cómo se actualiza

Cada repo privado sincroniza su carpeta en cada push a `main/master/development`. Nadie edita este repositorio a mano
salvo este `README.md`.

```
ADA_ENTERPISE_CORE/docs/qa/                     ──────► ada/qa/
arcadia/infraestructuradomain/print-agent/      ──────► ada/infraesturadomain/print-agent/latest/
arcadia/docs/MICROFRONTEND-CONFIG-ARCHITECTURE.md     ─► ada/frontend/
arcadia/infraestructuradomain/print-agent/README.md  ─► ada/infraesturadomain/print-agent/
```

> Si encuentras algo desactualizado, el arreglo va en el repo privado del que sale — nunca aquí directamente.