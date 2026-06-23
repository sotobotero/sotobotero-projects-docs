# Estrategia de Onboarding de Inventario — Tienda de Ropa

> **Contexto:** Negocio con cientos de referencias sin sistema previo. El objetivo es sistematizar el inventario de la forma más rápida, práctica y económica posible, y que el punto de venta sea ágil desde el primer día.

---

## 1. Filosofía: no paralizar la operación, cargar en movimiento

El objetivo no es tener el inventario en el sistema — el objetivo es:
1. **Saber qué hay en stock** en todo momento (control de inventario)
2. **Registrar cada venta** correctamente (control de ventas)
3. **Generar reportería útil**: qué se vende más, qué marca rota mejor, qué talla se agota primero, cuánto se vendió por día/semana/mes

### El error más común al iniciar
Intentar cargar **todo el inventario en Excel antes de empezar a vender**. Eso:
- Paraliza la tienda días o semanas
- Genera errores masivos (stock que no coincide con la realidad)
- Desmotiva al equipo antes de arrancar
- Solo funciona si el cliente ya tiene algo sistematizado o si el proveedor envía albaranes digitales

### La estrategia correcta: carga en movimiento (live loading)

**Cargar el sistema directamente mientras se opera.** No hay plantilla de Excel de por medio.

```
Llega un cliente → se toma la prenda
     ↓
¿Existe el producto en el sistema?
     ├─ SÍ → escanear → vender → listo
     └─ NO → crear el producto rápido (nombre + precio + talla + marca)
              → vender → el sistema ya tiene esa referencia registrada
```

Con este enfoque:
- **La operación no se detiene nunca**
- El inventario se construye orgánicamente con cada venta
- Los productos más vendidos quedan en el sistema primero (los que más importan)
- Los que nunca se venden nunca generan trabajo de digitación innecesario

### Cuándo sí usar Excel o importación masiva
- Cuando el proveedor envía su listado de productos en Excel o CSV
- Cuando ya existe un inventario previo en otro sistema para migrar
- Para cargar una colección nueva que llega toda de una vez (ej: pedido de temporada)

### Lo mínimo para crear un producto en el momento de la venta
Solo **3 campos** son necesarios para no bloquear la caja:

| Campo | Por qué es suficiente en el momento |
|---|---|
| `nombre` (con marca) | Identifica el producto — `Levi's Camiseta Negra M` |
| `precio_venta` | Para poder cobrar |
| `talla` | Para controlar stock por variante desde el primer registro |

El `codigo` puede auto-generarse. `marca`, `categoria`, `cantidad` se completan después o en el mismo momento si hay tiempo.

---

## 2. Campos mínimos del inventario para carga inicial (manual o plantilla de Excel)

Estos **7 campos obligatorios** son lo mínimo real para una tienda de ropa — sin talla ni marca el inventario no funciona bien desde el día 1:

| Campo | Descripción | Ejemplo |
|---|---|---|
| `codigo` | Código interno único del producto | `LEV-CAM-NEG-S` |
| `nombre` | Marca + descripción corta (legible por humanos) | `Levi's Camiseta Negra Cuello V` |
| `marca` | Marca o proveedor principal | `Levi's` |
| `categoria` | Tipo de prenda | `Camisetas` |
| `talla` | Talla de la prenda | `S`, `M`, `L`, `XL`, `32`, `34`... |
| `precio_venta` | Precio al público | `45000` |
| `cantidad` | Stock inicial disponible | `12` |

**Por qué `talla` y `marca` son obligatorios (no opcionales):**
- Sin `talla`: el inventario no sabe cuántas S vs L hay — imposible gestionar reposición
- Sin `marca`: no se puede filtrar por proveedor ni saber a quién reordenar

**Campos opcionales útiles (para fase 2):**
- `color` — si el código ya lo incluye, puede omitirse como campo separado inicialmente
- `precio_costo` — para calcular margen
- `proveedor_contacto` — teléfono o nota del distribuidor

> **Regla de oro:** si un campo no cambia el precio ni el stock, no es urgente. Se puede completar después. Pero talla y marca sí cambian cómo se gestiona el stock.

---

## 3. Estrategia de código de producto

### El problema
Una tienda de ropa local raramente tiene EAN/UPC estándares impresos. Los proveedores locales no los usan. Crear el inventario con los códigos del proveedor es caótico porque cada uno tiene su propio sistema.

### La solución: código interno propio con marca primero

La marca va **primero** porque así es como se piensa: "¿qué marca es?" antes que la categoría.

```
[MARCA(3)]-[CAT(3)]-[COLOR(3)]-[TALLA]
     ↑          ↑         ↑        ↑
  3 letras   3 letras  3 letras  código talla

Ejemplos:
  LEV-CAM-NEG-S    → Levi's Camiseta Negra Talla S
  ZAR-PAN-AZU-32   → Zara Pantalón Azul Talla 32
  MAR-VES-ROJ-M    → Marca X Vestido Rojo Talla M
  ZAR-BLU-BLA-L    → Zara Blusa Blanca Talla L
```

**Si la tienda maneja una sola marca o pocas:**
```
[CAT(3)]-[COLOR(3)]-[TALLA]   ← más corto, igual de útil
  CAM-NEG-S, CAM-NEG-M, CAM-NEG-L...
```

**El nombre siempre incluye la marca** para que sea legible en pantalla y en reportes:
```
nombre = "Levi's Camiseta Negra Cuello V"   ← no solo "Camiseta negra"
```

### Cuándo usar el código del proveedor o EAN
- Si el proveedor ya trae etiqueta con código de barras **legible** → escanear y registrar ese código en ADA directamente
- Si no trae nada → generar el código interno y pegar etiqueta propia

---

## 4. Dos modelos de operación: con etiquetado vs a ojímetro

> **Escala de referencia:** tienda con ~20.000 referencias y ~500.000 unidades en stock.

### Modelo A — Etiquetado propio con código de barras (ordenado y escalable)

Cada prenda lleva una etiqueta impresa con código de barras propio pegada antes de salir al piso de venta. Al vender: se escanea, el sistema la identifica en milisegundos.

**Ventajas operativas:**
- **Cualquier empleado puede operar la caja** — no se necesita al experto de 20 años
- **Velocidad real en hora pico** — escaneo es más rápido que buscar a mano
- **Inventario siempre actualizado** — cada venta descuenta automáticamente
- **Sin errores de precio** — el sistema manda, no la memoria de nadie
- **Reportería confiable** — los datos son exactos porque vienen del escaneo
- **Control de hurto** — es fácil detectar diferencias entre stock sistema vs físico
- **Escalable a cualquier volumen** — 500 o 500.000 prendas, el proceso es idéntico

**Hardware para esta escala: TSC TE210 o equivalente**

| Característica | TSC TE210 ✅ | Impresora de consumo (NIIMBOT, etc.) |
|---|---|---|
| Velocidad de impresión | 127 mm/s | ~30 mm/s |
| Tecnología | Transferencia térmica + Directa | Térmica directa solo |
| Durabilidad etiqueta | Alta (ribbon cera-resina) | Baja (se borra con el tiempo) |
| Conectividad | USB + Ethernet (red local) | Bluetooth (teléfono) |
| Durabilidad del equipo | Industrial, uso continuo | Uso ocasional |
| Costo aprox. | $200–350 USD | $35–50 USD |
| Para 20k referencias / 500k unidades | ✅ Adecuada | ❌ Demasiado lenta e inadecuada |

> A 127 mm/s una TSC TE210 imprime ~1.500–2.000 etiquetas/hora. Para 500.000 unidades en stock inicial: ~250–330 horas de impresión → se necesita un sprint dedicado de etiquetado con 2–3 personas durante 2–3 semanas antes o en paralelo con la operación.

**Contenido mínimo de la etiqueta:**
```
┌─────────────────────────┐
│  Levi's Camiseta Negra  │
│  Talla: M   Color: NEG  │
│  |||||||||||||||||||    │  ← código de barras Code 128
│  LEV-CAM-NEG-M          │
│  $ 45.000               │
└─────────────────────────┘
```

---

### Modelo B — Sin etiquetado físico ("a ojímetro")

No hay etiquetas. El cajero identifica el producto visualmente, lo busca en el sistema por nombre, selecciona y cobra.

**Cómo funciona en la práctica:**
- Depende completamente del empleado que conoce el inventario de memoria
- A 20 referencias funciona. A 200 es lento. A 20.000 es inviable.

**Problemas reales a escala:**
| Problema | Impacto |
|---|---|
| El experto falta un día | La tienda no puede operar normalmente |
| Dos prendas similares, precio distinto | Error de cobro casi seguro |
| Rush de fin de semana | Cola larga, errores bajo presión |
| Inventario físico vs sistema | Nunca van a coincidir |
| Reportería | Los datos son basura — no se puede confiar en ellos |
| Capacitación de nuevo empleado | Semanas o meses para "aprender el inventario" |

**Cuándo tiene sentido:**
- Tiendas muy pequeñas (< 100 referencias, 1 sola persona)
- Como modelo transitorio los primeros días mientras llegan las etiquetas
- Nunca como modelo permanente a partir de ~500 referencias

---

### Comparación directa a escala

| Criterio | Modelo A (etiquetado) | Modelo B (ojímetro) |
|---|---|---|
| **Velocidad en caja** | ⚡ Muy rápida (1 scan) | 🐢 Lenta (búsqueda manual) |
| **Precisión de cobro** | ✅ 100% (sistema manda) | ⚠️ Depende del humano |
| **Control de inventario** | ✅ Tiempo real | ❌ Imposible a escala |
| **Reportería útil** | ✅ Confiable | ❌ Datos no confiables |
| **Dependencia de persona clave** | ✅ Ninguna | 🔴 Alta (punto único de falla) |
| **Capacitación nuevo cajero** | Horas | Semanas/meses |
| **Escalabilidad** | ✅ Ilimitada | ❌ Techo ~200–500 refs |
| **Costo inicial extra** | $200–400 USD (impresora) | $0 |

**Conclusión:** con 20.000 referencias y 500.000 unidades, el Modelo B no es una opción viable. El Modelo A es la única forma de operar con control y precisión. El costo de la impresora se recupera en la primera semana de operación eficiente.

---

## 4b. Decisión: Código de barras vs QR

### Recomendación: **Código de barras Code 128** (no QR)

| Criterio | Código de Barras (Code 128) | QR |
|---|---|---|
| Costo del lector | $15–35 USD | Similar (lectores modernos leen ambos) |
| Velocidad de escaneo | ⚡ Muy rápida (1 pasada) | Rápida |
| Impresión | Impresora térmica básica | Impresora térmica básica |
| Tamaño mínimo útil | ~1.5 cm de ancho | ~1.5 × 1.5 cm |
| Lectura con teléfono | ✅ App de cámara nativa | ✅ App de cámara nativa |
| Información que almacena | Solo el código (suficiente) | Más datos (innecesario aquí) |
| Estándar en retail | ✅ Universal | Menos común en punto de venta |

**Conclusión:** Code 128 es suficiente. Un código como `CAM-NEG-S` cabe perfectamente en una etiqueta y se escanea en milisegundos. El QR no agrega valor para este caso de uso.

---

## 5. Hardware recomendado

### Impresora de etiquetas: TSC TE210 203 USB+Ethernet

**[Ver en MercadoLibre Colombia](https://www.mercadolibre.com.co/impresora-de-etiquetas-termica-tsc-te210-203-usb-ethernet/up/MCOU3931234253)**

| Especificación | Valor |
|---|---|
| **Tecnología** | Térmica directa + Transferencia térmica (dos modos) |
| **Velocidad** | 127 mm/s |
| **Resolución** | 203 dpi (suficiente para Code 128 legible) |
| **Ancho máximo de etiqueta** | 108 mm (4.25") |
| **Conectividad** | USB + Ethernet (conecta directo a red local) |
| **Núcleo del rollo compatible** | 25 mm (1") o 38 mm (1.5") |
| **Diámetro máximo del rollo** | 127 mm (5") |
| **Precio** | Verificar en el link — aprox. $400.000–$600.000 COP |

---

### Rollos de etiquetas — 3 opciones disponibles

Los tres rollos son **térmica directa**, lo que significa: sin ribbon, sin cinta, solo el rollo. La TSC TE210 los soporta en modo térmico directo. Para una tienda de ropa con rotación razonable (menos de 6 meses en stock), la térmica directa es suficiente y simplifica la operación.

| | Opción 1 | Opción 2 | Opción 3 ✅ |
|---|---|---|---|
| **Tamaño** | 32 × 25 mm | 100 × 50 mm | **50 × 30 mm** |
| **Cantidad** | 2.000 / rollo | Ver link | 1.000 / rollo |
| **Link ML** | [Ver](https://www.mercadolibre.com.co/rollo-de-etiquetas-adhesivas-32x25mm-termicas-directas-x2000/up/MCOU2867558313) | [Ver](https://www.mercadolibre.com.co/etiqueta-adhesiva-termica-100x50mm-rollo/up/MCOU3807251195) | [Ver](https://www.mercadolibre.com.co/wg-etiquetas-termicas-adhesivas-50x30mm-rollo-blanco-1000-unidades/p/MCO44780468) |
| **Para ropa** | ⚠️ Muy pequeña | ❌ Demasiado grande | ✅ Ideal |
| **Por qué** | Difícil meter barcode + nombre + talla + precio legibles | Desperdicio de material, muy grande para una prenda | Cabe todo el contenido con texto legible |

#### Recomendación: **Opción 3 — 50×30 mm**

Es el tamaño estándar en retail de ropa. Permite incluir en una sola etiqueta:

```
┌───────────────────────────┐
│ Levi's Camiseta Negra     │  ← nombre (fuente pequeña)
│ Talla: M    Color: Negro  │  ← atributos clave
│ |||||||||||||||||||||||   │  ← código de barras Code 128
│ LEV-CAM-NEG-M             │  ← código legible
│ $ 45.000                  │  ← precio (fuente grande)
└───────────────────────────┘
     50 mm × 30 mm
```

#### Costo estimado por 1.000 etiquetas impresas (térmica directa)

| Concepto | Costo aprox. |
|---|---|
| Etiquetas 50×30 mm (x1.000) | Verificar precio en ML |
| Ribbon | **No aplica** — térmica directa |
| **Costo operativo por 1.000** | Solo el costo del rollo |
| **Para 500.000 unidades** | 500 rollos × precio unitario |

> Sin ribbon el costo operativo baja significativamente vs transferencia térmica. El único insumo recurrente es el rollo de etiquetas.

---

### Impresora de tickets/recibos: Digitalpos DIG-K200L USB+LAN

**[Ver en MercadoLibre Colombia](https://www.mercadolibre.com.co/impresora-pos-digitalpos-dig-k200l-usb-usblan-color-negro/p/MCO35034412)**

Impresora POS para el recibo/factura al cliente en el punto de caja. Va junto con la TSC TE210 pero cumplen funciones distintas:

| | TSC TE210 | Digitalpos DIG-K200L |
|---|---|---|
| **Función** | Imprimir etiquetas de producto | Imprimir ticket/recibo de venta |
| **Va en** | Bodega / área de etiquetado | Caja / punto de venta |
| **Se usa** | Al recibir mercancía nueva | Con cada venta |
| **Papel** | Rollos de etiquetas adhesivas | Rollo de papel térmico 80 mm |
| **Precio** | Verificar en ML | Verificar en ML |

---

### Lector de código de barras

**Recomendado: Jaltech 2D QR y Barras USB**
**[Ver en MercadoLibre Colombia](https://www.mercadolibre.com.co/lector-codigo-de-barras-jaltech-2d-qr-y-barras-usb/up/MCOU3332948402)**

> ⚠️ **Ojo al comprar:** en el mercado hay lectores **1D** (más baratos) que solo leen códigos de barras lineales y **no leen QR**. Siempre comprar **2D**, que lee tanto barras como QR. La diferencia de precio es poca y evita quedarse bloqueado si en algún momento se necesita escanear un QR (guías de envío, documentos de proveedor, etc.).

| Opción | Uso | Costo aprox. |
|---|---|---|
| **Jaltech 2D USB** (recomendado) | Caja fija, conectado a PC | Verificar en ML |
| Bluetooth inalámbrico 2D | Libertad en tienda, inventariar en bodega | $150.000–$250.000 COP |
| Teléfono con app (Barcode Scanner) | Backup o inventariar puntualmente | $0 |

> Para una tienda con varios puntos de caja: un lector USB por caja + uno Bluetooth para el área de bodega/recepción de mercancía.

---

## 6. Flujo de onboarding: de cero al sistema sin parar ventas

### Día 1 — Arranque inmediato

```
1. CONFIGURAR EL SISTEMA (medio día)
   └─ Crear el tenant del negocio
   └─ Definir las categorías principales (Camisetas, Pantalones, Vestidos...)
   └─ Configurar impresora de etiquetas si ya se tiene

2. EMPEZAR A VENDER (ese mismo día)
   └─ Por cada producto que llega a caja:
       ├─ Si ya existe → escanear → vender
       └─ Si no existe → crear en 30 segundos → vender → etiqueta impresa

3. TRABAJO PARALELO (fuera de horas pico)
   └─ Un empleado va cargando el inventario físico de los estantes
   └─ Se prioriza lo que más rota (lo que está en vitrina o vende rápido)
   └─ Se imprimen etiquetas por lotes para la mercancía ya cargada
```

### Semana 1-2 — Consolidación orgánica

```
CADA VEZ QUE SE HACE UNA VENTA:
   └─ Si el producto no existía → queda registrado
   └─ Si existía → el stock se descuenta automáticamente

CADA VEZ QUE LLEGA MERCANCÍA NUEVA:
   └─ Registrar antes de colgar (aprovechar que está en la mano)
   └─ Imprimir etiquetas del nuevo lote
   └─ Registrar cantidad recibida como entrada de inventario

AL FINAL DE LA SEMANA 2:
   └─ Los productos más vendidos ya están todos en el sistema
   └─ El inventario restante se puede cargar por lotes en momentos de baja actividad
```

### Cuándo usar importación desde Excel
Solo en estos casos específicos:
- El proveedor envía su catálogo en Excel o CSV → importar directamente
- Colección nueva de temporada que llega toda junta → cargar el lote antes de etiquetar
- Migración desde otro sistema que permite exportar los datos

---

## 7. Flujo de venta con escaneo

```
Cajero escanea etiqueta
     ↓
Sistema ADA busca por código
     ↓
Agrega producto a la factura con precio
     ↓
Descuenta automáticamente del inventario
     ↓
Al finalizar: imprime factura / ticket
```

**Nota para manejo de tallas y colores:**  
Si una misma referencia existe en varias tallas, se recomienda un código por variante (`CAM-NEG-S`, `CAM-NEG-M`, `CAM-NEG-L`) en lugar de un único código con campo talla. Esto simplifica el escaneo y el control de stock por talla.

---

## 8. Reportería útil — por qué los campos importan

Los 7 campos mínimos no son arbitrarios: cada uno habilita un reporte accionable:

| Campo | Reporte que habilita |
|---|---|
| `marca` | Ventas por marca — ¿qué marca rota más? ¿cuál deja más margen? |
| `categoria` | Ventas por tipo de prenda — ¿camisetas vs pantalones? |
| `talla` | Stock y ventas por talla — ¿cuál se agota primero? ¿cuál sobra? |
| `precio_venta` | Ingresos totales, ticket promedio por venta |
| `cantidad` | Alertas de stock bajo, necesidad de reposición |
| `codigo` | Trazabilidad completa: qué se vendió, cuándo y a qué precio |
| `nombre` (con marca) | Búsqueda rápida en caja y en reportes sin ambigüedad |

**Reportes mínimos que el sistema debe generar:**
- Ventas del día / semana / mes (en pesos y en unidades)
- Top 10 productos más vendidos
- Stock actual por categoría y talla
- Productos con stock bajo (para reposición)
- Ventas por marca (para saber a qué proveedor reordenar más)

> Si los campos de marca y talla se dejan vacíos o inconsistentes en el Excel inicial, estos reportes no funcionan. Vale la pena dedicar el tiempo extra en la carga inicial para llenarlos bien.

---

## 9. Crecimiento: nueva mercancía

Cuando llega mercancía nueva:
1. Revisar si el proveedor trae código de barras propio → registrar ese código en ADA
2. Si no trae → asignar código interno siguiente en la secuencia
3. Imprimir etiquetas antes de colgar en tienda
4. Registrar cantidad recibida como entrada de inventario

---

## Referencias internas

- Tabla en BD: `inventory.product` (esquema ADA)
- Limpieza de datos al crear tenant: ver [TENANT_PROVISIONING.md](TENANT_PROVISIONING.md)
- Permisos de app user sobre esquema inventory: ver [DEVELOPER_GETTING_STARTING.md](../DEVELOPER_GETTING_STARTING.md)
