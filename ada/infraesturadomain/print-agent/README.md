# print-agent

Agente local de impresión para terminales POS. Recibe peticiones HTTP del navegador y envía los comandos a las impresoras USB conectadas al PC del cajero. Corre como servicio Windows o systemd en Linux y escribe logs en `print-agent.log`.

| Endpoint | Impresora | Protocolo |
|---|---|---|
| `POST /print/label` | DIG-2406T PRO (etiquetas 50×25mm) | TSPL directo |
| `POST /print/ticket` | DIG-K200L (tickets) | ESC/POS texto |
| `POST /print/pdf` | DIG-K200L (tickets) | PDF → ghostscript → raster ESC/POS |
| `POST /drawer/open` | Cajón monedero DIG-KR410 | ESC/POS kick via impresora de tickets |
| `GET /health` | — | estado de ambos dispositivos |

---

## Conceptos clave

### El estándar POS y el papel de 80mm

Las impresoras de punto de venta (POS) usan casi universalmente **papel térmico de 80mm de ancho**. El cabezal de impresión físico cubre entre 72 y 76mm del ancho total — los milímetros restantes son margen mecánico que el hardware descarta. Por eso el área imprimible real ronda los **72mm (~204pt en Jasper)**. Ignorar esto es la causa más común de contenido cortado en el margen derecho, aunque el preview del navegador lo muestre perfecto.

### Glosario técnico

| Término | Qué es | Dónde aparece |
|---|---|---|
| **ESC/POS** | Lenguaje de comandos binario para impresoras térmicas POS, creado por Epson y adoptado como estándar de facto. Los comandos son secuencias de bytes que controlan corte de papel, avance, negrita, alineación, impresión de imágenes raster y apertura de cajón. | `/print/ticket`, `/print/pdf`, `/drawer/open` |
| **GS v 0** | Comando ESC/POS específico para imprimir una imagen raster: `1D 76 30 00 xL xH yL yH [datos]`. El agente convierte el PNG de Ghostscript a este formato bit a bit antes de enviarlo a la impresora. | `pdf.go → imageToESCPOS()` |
| **ESC p** | Comando ESC/POS de apertura de cajón monedero: `1B 70 m t1 t2`. `m=0` activa el pin 2 del puerto RJ11, `m=1` el pin 5. Los tiempos `t1`/`t2` controlan la duración del pulso en unidades de 2ms. | `escpos.go → escDrawerKick` |
| **CUPS** | Sistema de impresión de Unix/Linux. Gestiona colas de impresión y aplica filtros (PPD/driver) para convertir el documento al lenguaje de la impresora. Cuando el usuario hace Ctrl+P en el navegador, el trabajo pasa por CUPS. Interfaz web de administración en `http://localhost:631/admin/` — pide usuario y contraseña del usuario Linux normal del sistema. | Ruta browser → CUPS → `rastertopdt` |
| **rastertopdt** | Filtro CUPS del fabricante Gainscha. Convierte el raster de CUPS al protocolo propietario de la DIG-K200L. Se instala desde el driver oficial Linux de Gainscha. | `/usr/lib/cups/filter/rastertopdt` |
| **PPD** | *PostScript Printer Description*. Archivo de texto que describe las capacidades de la impresora a CUPS: tamaños de papel, márgenes, opciones de corte. El PPD de Gainscha declara `ImageableArea 226pt` (80mm completos) sin márgenes, lo que no refleja el cabezal físico de ~72mm. | `POS_Printer.ppd.gz` |
| **Ghostscript** | Intérprete de PostScript/PDF de código abierto. El agente lo usa para rasterizar el PDF generado por Jasper a una imagen PNG en escala de grises a 203 DPI — la resolución nativa de la DIG-K200L. | `pdf.go → pdfToESCPOS()` |
| **203 DPI** | Resolución estándar del cabezal térmico de la mayoría de impresoras POS de 80mm. A 203 DPI, 80mm equivalen a ~637px y 72mm a ~576px. Renderizar a esta resolución sin escalar produce una salida idéntica a la ruta CUPS. | `-r203` en el comando `gs` |
| **TSPL** | *Thermal Shipping/Printer Language*. Lenguaje de comandos para impresoras de etiquetas (distinto de ESC/POS). La GS-2406T lo usa. Los comandos son texto plano (`SIZE`, `GAP`, `BARCODE`, `TEXT`, `PRINT`). | `/print/label`, `label.go` |
| **usblp** | Módulo del kernel Linux que crea nodos `/dev/usb/lpX` para impresoras USB. CUPS en modo `libusb` lo evita y habla directo con el hardware — por eso no hay nodo `/dev/usb/lp*` de la ticket printer hasta que se fuerza el binding manual. | Sección *Conectar impresora de tickets* |
| **RJ11** | Conector telefónico de 6 pines usado por los cajones monedero POS para conectarse a la impresora. No es USB ni red — la impresora envía el pulso ESC/POS al cajón a través de este cable. | *Conectar cajón monedero* |

---

## Índice

1. [Guía del desarrollador](#guía-del-desarrollador)
   - [Prerequisitos](#prerequisitos)
   - [Carpeta de trabajo](#carpeta-de-trabajo)
   - [Comandos](#comandos)
   - [Conectar impresora de etiquetas en Linux (GS-2406T)](#conectar-impresora-de-etiquetas-en-linux-gs-2406t)
   - [Smoke test](#smoke-test)
   - [Ejemplo de llamada — etiqueta mínima](#ejemplo-de-llamada--etiqueta-mínima)
   - [Ejemplo de llamada — etiqueta completa](#ejemplo-de-llamada--etiqueta-completa)
2. [Conectar impresora de tickets en Linux (DIG-K200L)](#conectar-impresora-de-tickets-en-linux-dig-k200l)
3. [Conectar cajón monedero (DIG-KR410)](#conectar-cajón-monedero-dig-kr410)
4. [Configurar el lector de códigos de barras](#configurar-el-lector-de-códigos-de-barras)
5. [Guía del usuario (instalar el servicio)](#guía-del-usuario-instalar-el-servicio)
   - [Linux](#linux)
   - [Windows](#windows)
     - [Paso 1 — Instalar las impresoras y sus drivers](#paso-1--instalar-las-impresoras-y-sus-drivers)
     - [Paso 2 — Instalar Ghostscript](#paso-2--instalar-ghostscript-prerequisito-para-printpdf)
     - [Paso 3 — Instalar el agente](#paso-3--instalar-el-agente)
     - [Paso 4 — Abrir el puerto en el Firewall de Windows](#paso-4--abrir-el-puerto-en-el-firewall-de-windows)

---

## Guía del desarrollador

### Prerequisitos

```bash
# Verificar versión de Go (necesaria 1.22+)
go version
# debe mostrar: go version go1.22.x ... o superior
```

Si el comando no se encuentra (`bash: go: command not found`), Go no está en el PATH o no está instalado. Consulta la [guía oficial de instalación de Go](https://go.dev/doc/install).

**Carpetas de Go — referencia rápida**

| Carpeta | Qué es | Quién la crea | Contiene | ¿Tocar? |
|---|---|---|---|---|
| `~/go-sdk/go/` | Compilador (toolchain) | GoLand al descargar el SDK | `bin/go`, stdlib, fuentes | Nunca |
| `/usr/local/go/` | Compilador (toolchain) | Instalación manual desde go.dev | `bin/go`, stdlib, fuentes | Nunca |
| `~/go/pkg/` | Caché de módulos | Go automáticamente | Dependencias descargadas | Nunca |
| `~/go/bin/` | Binarios instalados | `go install` | Herramientas instaladas | Solo añadir al PATH |

```bash
# Añadir el compilador al PATH de forma permanente (~/.bashrc o ~/.zshrc)
export PATH=$PATH:$HOME/go-sdk/go/bin   # si lo instaló GoLand/VSCode
# o
export PATH=$PATH:/usr/local/go/bin     # si se instaló manualmente
source ~/.bashrc
```

Si no tienes Go instalado en absoluto: [go.dev/dl](https://go.dev/dl) → descargar el `.tar.gz` para Linux o el `.msi` para Windows.

En Linux, el usuario necesita permiso sobre los dispositivos USB:
```bash
sudo usermod -aG lp $USER
# cerrar sesión y volver a entrar para que tome efecto
```

### Carpeta de trabajo

Todos los comandos se ejecutan desde la carpeta del proyecto:

```
arcadia/infraestructuradomain/print-agent/
```

```bash
# Situarse en la carpeta correcta
cd arcadia/infraestructuradomain/print-agent

# Verificar que estás en el lugar correcto (debe mostrar go.mod)
ls go.mod
```

### Archivos del proyecto

| Archivo | Para qué sirve |
|---|---|
| `main.go` | Punto de entrada. Procesa flags (`--install`, `--uninstall`) y arranca el servidor HTTP. |
| `config.go` | Lee `print-agent.env` y expone la configuración al resto del programa. |
| `tspl.go` | Genera los comandos TSPL para la impresora de etiquetas. |
| `escpos.go` | Genera los comandos ESC/POS para la impresora de tickets. |
| `service_linux.go` | Implementa `--install` / `--uninstall` para Linux (systemd). Solo se compila en Linux. |
| `service_windows.go` | Implementa `--install` / `--uninstall` para Windows (Services). Solo se compila en Windows. |
| `print-agent.service` | Referencia del unit systemd — solo para documentación. El instalador genera el real en `/etc/systemd/system/`. |

> **¿Por qué dos archivos de servicio?** Go incluye en el binario solo el código del sistema operativo destino. `service_linux.go` y `service_windows.go` contienen la misma interfaz (`installService`, `uninstallService`, `serve`) pero con implementaciones distintas. Al compilar en Linux solo entra `service_linux.go`; al cross-compilar para Windows solo entra `service_windows.go`. El resultado es un único ejecutable sin dependencias externas.

### Comandos

```bash
# Crear configuración local (solo la primera vez)
cp print-agent.env.example print-agent.env
# editar print-agent.env con tus valores

# Compilar binario para Linux
go build -o print-agent .

# Cross-compilar para Windows (desde Linux)
GOOS=windows GOARCH=amd64 go build -o print-agent.exe .

# Correr en modo desarrollo (sin compilar)
go run .
```

### Distribucion para usuarios Windows

Publica en formato `.zip`, no el `.exe` suelto.

Motivo:

- Evita bloqueos del navegador/antivirus al descargar un `.exe` directo.
- Permite incluir todo lo necesario en un solo paquete.
- Reduce errores de instalacion para usuario no tecnico.

Contenido recomendado del zip:

- `print-agent.exe`
- `print-agent.env.example`
- `README.md` (o un `INSTALL_WINDOWS.md` corto)

Nombre sugerido del artefacto:

- `print-agent-windows-amd64-sha-<commit>.zip`

### Conectar impresora de etiquetas en Linux (GS-2406T)

**1. Conectar y encender**

Conecta el cable USB con la impresora **apagada**, luego enciéndela.

**2. Verificar detección**

```bash
# El kernel debe reportar el dispositivo
sudo dmesg | tail -20
# Buscar líneas como:
# usb 3-2: Product: GS-2406T
# usblp1: USB Bidirectional printer ...

# Ver nodo de dispositivo creado
ls -la /dev/usb/lp*
# crw-rw---- 1 root lp 180, 1 ... /dev/usb/lp1
```

**3. Añadir usuario al grupo `lp`**

El dispositivo pertenece al grupo `lp` — **distinto de `lpadmin`**, que es el grupo de administración de CUPS y no da acceso directo al dispositivo USB.

```bash
sudo usermod -aG lp $USER
newgrp lp          # activa el grupo en el terminal actual sin cerrar sesión
groups             # debe mostrar "lp" en la lista
```

> Si `groups` no muestra `lp` después de `newgrp lp`, cierra sesión y vuelve a entrar. Luego relanza `go run .` desde esa misma terminal.

**4. Actualizar `print-agent.env`**

```
LABEL_DEVICE=/dev/usb/lp1   # ajustar al nodo detectado en el paso 2
```

**5. Calibrar el papel (una vez por sesión)**

Presiona el botón **FEED** con la impresora ya encendida. Avanza hasta el inicio de la siguiente etiqueta y sincroniza el gap. Si el papel sale mal posicionado, vuelve a presionar FEED antes de imprimir.

---

### Smoke test

Una vez arrancado (`go run .` o el binario), verificar que responde:

```bash
curl http://localhost:5000/health
# {"label_device":"not found","status":"ok","ticket_device":"not found"}
# label/ticket_device = "not found" es normal si la impresora no está conectada
```

O desde el navegador: [http://localhost:5000/health](http://localhost:5000/health)

### Ejemplo de llamada — ticket (Windows PowerShell)

```powershell
$body = '{"businessName":"Mi Negocio","items":[{"name":"Producto","quantity":1,"price":10000}],"total":10000}'
curl.exe -X POST http://localhost:5000/print/ticket -H "Content-Type: application/json" -H "Authorization: Bearer cambia-esto" -d $body
```

Respuesta esperada: **HTTP 204** y el ticket sale impreso.

> Usar `curl.exe` (no `curl`) para invocar el binario real y no el alias de PowerShell `Invoke-WebRequest`. El JSON en una variable `$body` evita problemas de escaping con las comillas.

### Ejemplo de llamada — etiqueta mínima

Solo los campos obligatorios. Es el caso habitual para productos sin talla ni material.

```bash
curl -X POST http://localhost:5000/print/label \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer cambia-esto" \
  -d '{
    "code": "LOVE-02568-856487",
    "name": "My Mona Hermosa Cup",
    "price": 1000000,
    "quantity": 1
  }'
```

### Ejemplo de llamada — etiqueta completa

Todos los campos opcionales activos. Los campos vacíos u omitidos no aparecen en la etiqueta.

```bash
curl -X POST http://localhost:5000/print/label \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer cambia-esto" \
  -d '{
    "code": "LOVE-02568-856487",
    "name": "My Mona Hermosa Cup",
    "price": 1000000,
    "quantity": 2,
    "slogan": "Vístete con propósito",
    "edition": "Edición Limitada",
    "size": "M",
    "material": "95% Lana / 5% Seda"
  }'
```

**Campos disponibles:**

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `code` | string | ✓ | Código de barras |
| `name` | string | ✓ | Nombre del producto |
| `price` | number | ✓ | Precio sin decimales |
| `quantity` | int | ✓ | Copias a imprimir |
| `slogan` | string | — | Línea superior de la etiqueta; acepta tildes |
| `edition` | string | — | Ej: "Edición Limitada" |
| `size` | string | — | Talla, máx 10 caracteres |
| `material` | string | — | Composición, máx 20 caracteres |

---

## Conectar impresora de tickets en Linux (DIG-K200L)

**1. Conectar y encender**

Conecta el cable USB y enciende la impresora. Si el papel sale al revés, dale la vuelta al rollo — el lado térmico (brillante) debe quedar hacia afuera.

**2. Verificar detección**

```bash
sudo dmesg | tail -20
# Buscar líneas como:
# usb 3-x: New USB device found ...
# usblp1: USB Bidirectional printer ...

ls -la /dev/usb/lp*
# crw-rw---- 1 root lp 180, 1 ... /dev/usb/lp1   ← etiquetas (GS-2406T)
# crw-rw---- 1 root lp 180, 2 ... /dev/usb/lp2   ← tickets  (DIG-K200L)
```

El número de nodo (`lp1`, `lp2`, …) depende del orden en que el kernel detecta los dispositivos. Si solo hay una impresora conectada, será `lp1`.

**3. Añadir usuario al grupo `lp`** (si no está ya)

```bash
sudo usermod -aG lp $USER
newgrp lp
groups   # debe mostrar "lp"
```

**4. Smoke test — texto plano directo al dispositivo**

Esta impresora acepta texto ASCII directamente, sin comandos ESC/POS. Verificar con:

```bash
# Test mínimo
printf 'Hola ticket\n\n\n\n\n\n' | sudo tee /dev/usb/lp2 > /dev/null

# Test con formato de ticket real
printf 'Mi Negocio\n2026-07-05 16:30\n--------------------------------\nCamiseta ZOE     1x  $45.000\nPantaloneta      2x  $32.000\n--------------------------------\nTOTAL: $109.000\n\n\n\n\n\n\n' | sudo tee /dev/usb/lp2 > /dev/null
```

Si imprime correctamente → la impresora está lista.

**Smoke test — vía CUPS (`lp`)**

Una vez registrada en CUPS (paso 5), verificar que el spooler entrega datos correctamente:

```bash
printf 'CUPS test\nMi Negocio\n2026-07-05\n\n\n\n' | lp -d DIG-K200L
# CUPS responde: request id is DIG-K200L-1 (0 file(s))
# La impresora debe imprimir las 3 líneas de texto.

# Ver estado del job
lpstat -p DIG-K200L
```

**Smoke test — PDF vía agente (`/print/pdf`) — método en uso**

El agente usa **ghostscript** para convertir el PDF a imagen raster y lo envía como ESC/POS bitmap. No requiere CUPS ni drivers adicionales — solo `gs` (preinstalado en la mayoría de distribuciones Linux).

Requisito: el PDF debe estar en formato receipt (≈88mm de ancho). El reporte `InvoiceTicket88mm.jrxml` de Jasper genera este formato correctamente.

```bash
# Codificar el PDF de la factura en base64
base64 -w 0 /ruta/a/factura.pdf > /tmp/b64.txt

# Enviar al agente — respuesta esperada: HTTP 204
curl -X POST http://localhost:5000/print/pdf \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer cambia-esto" \
  -d "{\"pdf\": \"$(cat /tmp/b64.txt)\", \"copies\": 1}"
```

Resultado verificado: logo, tablas de IVA, método de pago y pie de página imprimen correctamente a resolución 203 DPI en papel de 58mm.

---

### Driver oficial Gainscha — impresión directa desde el navegador ✅

La DIG-K200L es una impresora de **marca blanca fabricada por Gainscha** (la placa interna aparece como `usb://Gainscha/GA-E200I` en el sistema). Gainscha publica su propio driver Linux para impresoras de ticket en su web oficial.

#### Instalación del driver

**Requisitos:**

```bash
sudo apt-get install -y libcupsimage2   # librería CUPS de rasterizado requerida por el filtro
sudo apt-get install -y unrar           # para extraer el paquete del fabricante
```

**Descarga:** [gainscha.com.tw/download.html](https://www.gainscha.com.tw/download.html) → sección **Driver** → **Linux Driver — Receipt Printer Linux Driver**

**Instalación manual** (el script `setup` del paquete hace exactamente esto):

```bash
# Extraer el RAR
unrar x "Receipt Printer Linux Driver.rar" /tmp/gainscha/

cd "/tmp/gainscha/Receipt Printer Linux Driver/20240820_PDDRVlinuxn_CentOS7"

# Instalar el filtro CUPS
sudo cp rastertopdt /usr/lib/cups/filter/
sudo chmod +x /usr/lib/cups/filter/rastertopdt

# Instalar el PPD
sudo mkdir -p /usr/share/cups/model/generic
sudo cp POS_Printer.ppd.gz /usr/share/cups/model/generic/

sudo systemctl restart cups
```

**Configurar la impresora en CUPS:**

```bash
# Registrar con el driver del fabricante
sudo lpadmin -p DIG-K200L -E \
  -v "usb://Gainscha/GA-E200I?serial=111111151111" \
  -m "generic/POS_Printer.ppd.gz"

# Papel: 80mm — área imprimible 72mm (X72MM en puntos CUPS)
sudo lpadmin -p DIG-K200L -o media=X80MMY2000MM

# Corte automático al final de cada documento
sudo lpadmin -p DIG-K200L -o CutType=2DocCutMode
```

> El warning *"Printer drivers are deprecated"* es normal en CUPS 2.x — es un aviso de futuro, no un error. El driver funciona correctamente.
>
> `CutType=2DocCutMode` activa el corte al final de cada job. Sin esta opción el default del PPD es `0NoCutPage` (sin corte).

#### Smoke test — imprimir PDF desde terminal

```bash
lp -d DIG-K200L -o fit-to-page /ruta/a/factura.pdf
```

Resultado verificado: la factura imprime con cabecera, artículos, tabla IVA, método de pago y corte automático. ✅

#### Estado de las dos rutas de impresión

| Ruta | Estado | Cuándo usarla |
|---|---|---|
| **`/print/pdf` del agente** (ghostscript → ESC/POS raster) | ✅ funciona | Botón de impresión desde el frontend — no depende del driver en cada PC |
| **CUPS `lp` con driver Gainscha** (`rastertopdt`) | ✅ funciona | Impresión desde terminal o desde el diálogo del navegador (`Ctrl+P`) en el PC con el driver instalado |

> Para el flujo POS en producción, el agente es la opción recomendada: una sola instalación en el servidor local, sin gestionar drivers en cada equipo cliente.

---

#### Diagnóstico de problemas de impresión — identifica la ruta antes de aplicar cambios

**Regla fundamental:** antes de tocar cualquier parámetro, determina desde dónde se lanza la impresión. Un fix que soluciona el margen en una ruta puede ser irrelevante en la otra.

| Síntoma | Ruta activa | Dónde aplicar el fix |
|---|---|---|
| El usuario pulsa Ctrl+P o un botón nativo del browser | **CUPS** (`rastertopdt`) | Template JRXML (márgenes, `columnWidth`) |
| El frontend llama a `POST /print/pdf` del agente | **Ghostscript** (agente) | `pdf.go` / `TICKET_RASTER_WIDTH` env var |
| `lp -d DIG-K200L` desde terminal | **CUPS** | Template JRXML |
| `curl ... /print/pdf` | **Ghostscript** | `pdf.go` / env var |

**Cómo confirmar la ruta en uso:**

- Ruta CUPS: el job aparece en `lpstat -W completed -l` y en `/var/log/cups/`.
- Ruta agente: el log del agente muestra `pdf printed: copies=N` (`journalctl -u print-agent -f`).

---

#### Área imprimible física de la DIG-K200L — lección aprendida

La DIG-K200L usa papel de **80mm** pero la cabeza térmica imprime solo **~72mm** de ancho. El PPD del driver Gainscha declara incorrectamente `ImageableArea = 226pt` (80mm completos) sin márgenes, de modo que el driver envía los 80mm al hardware y el hardware descarta físicamente lo que excede ~204pt (~72mm) desde el borde izquierdo del papel.

**Impacto en el template JRXML (ruta CUPS):**

El template `Ticket80mmMain.jrxml` tiene `pageWidth="226"` (80mm). Si el `columnWidth` acerca el contenido al borde derecho del área de 80mm, ese contenido se corta en la impresión física aunque el preview del browser lo muestre correctamente.

Valores correctos para que el contenido quede dentro del área imprimible:

| Parámetro JRXML | Valor anterior | Valor correcto | Razonamiento |
|---|---|---|---|
| `columnWidth` | 218pt | **196pt** | Borde derecho del contenido a 200pt (4pt margen + 196pt) < 204pt límite físico |
| `rightMargin` | 4pt | **26pt** | 226 - 4 - 196 = 26pt de zona de seguridad hacia el borde del papel |

Aplica lo mismo al subreporte `Ticket80mmDetailWithPrice.jrxml`: escalar todas las posiciones `x` y anchos de columna con ratio `196/218 ≈ 0.899`.

**Impacto en la ruta Ghostscript (agente `/print/pdf`):**

Ghostscript **ignora** la flag `-g{W}x{H}` si el PDF declara su propio `MediaBox` (que Jasper siempre incluye), a menos que se añada `-dFIXEDMEDIA`. Sin esa flag, cambiar `TICKET_RASTER_WIDTH` no tiene ningún efecto — Ghostscript siempre rasteriza al tamaño nativo del PDF (226pt × 203/72 ≈ 637px), y el contenido que supere el ancho del cabezal (~576px = 72mm a 203 DPI) se corta.

Fix aplicado en `pdf.go`:

```go
exec.Command("gs",
    "-dNOPAUSE", "-dBATCH", "-dQUIET",
    "-sDEVICE=pnggray",
    "-r203",
    fmt.Sprintf("-g%dx%d", width, heightPx),
    "-dFIXEDMEDIA",   // ← obligatorio para que -g sea respetado
    "-dPDFFitPage",
    "-sOutputFile="+pngPath,
    pdfPath,
)
```

Con el JRXML ya corregido a 196pt y `-dFIXEDMEDIA` activo, el contenido queda en ~509px dentro de los 576px del cabezal, sin cortes.

---

#### Nota: la DIG-K200L no crea `/dev/usb/lp` cuando CUPS gestiona el dispositivo

CUPS accede a la DIG-K200L vía su backend `libusb` directo (URI `usb://Gainscha/GA-E200I?serial=...`). En este modo, el módulo del kernel `usblp` **no se enlaza** al dispositivo y por tanto no se crea ningún nodo `/dev/usb/lp` para el ticket printer — aunque `lsusb` sí muestre el dispositivo (`ID 0471:0055 GA-E200I`).

Consecuencia: el agente (`sendRasterToDevice`) no puede abrir un nodo inexistente. Para que el agente pueda escribir al dispositivo directamente hay que forzar que `usblp` se enlace:

```bash
sudo modprobe usblp
sudo sh -c 'echo "0471 0055" > /sys/bus/usb/drivers/usblp/new_id'
ls /dev/usb/lp*   # aparece el nuevo nodo (ej. lp2)
```

CUPS detacha `usblp` mientras imprime y lo vuelve a attachar al terminar, de modo que ambas rutas pueden coexistir siempre que no intenten imprimir simultáneamente. Actualiza `TICKET_DEVICE` en `/opt/print-agent/print-agent.env` con el nodo detectado y reinicia el servicio.

---

#### Hub USB — los nodos `lpX` pueden intercambiarse al apagar/encender las impresoras

El kernel Linux asigna los números de nodo (`lp0`, `lp1`, `lp2` …) según el **orden en que detecta los dispositivos USB**. Cuando ambas impresoras pasan por un hub USB, ese orden depende de cuál responde primero al scan del bus — y puede variar entre arranques o tras apagar y volver a encender las impresoras.

**Síntoma:** después de un ciclo de apagado, lo que debía imprimir etiquetas sale por la impresora de tickets y viceversa.

**Por qué ocurre:** si en el arranque anterior `GS-2406T = lp1` y `GA-E200I = lp2`, pero en el siguiente ciclo el hub detecta primero a la GA-E200I, los nodos se invierten sin que el `.env` haya cambiado.

**Solución: udev rules con symlinks estables**

Crea el fichero `/etc/udev/rules.d/99-printers.rules` con las reglas que asocian cada impresora por su `idVendor`/`idProduct` a un nombre fijo:

```
# GS-2406T — impresora de etiquetas
SUBSYSTEM=="usb", ATTRS{idVendor}=="0471", ATTRS{idProduct}=="8e00", SYMLINK+="label-printer"

# GA-E200I / DIG-K200L — impresora de tickets
SUBSYSTEM=="usb", ATTRS{idVendor}=="0471", ATTRS{idProduct}=="0055", SYMLINK+="ticket-printer"
```

Aplica sin reiniciar:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -l /dev/label-printer /dev/ticket-printer
```

Luego actualiza `/opt/print-agent/print-agent.env`:

```
LABEL_DEVICE=/dev/label-printer
TICKET_DEVICE=/dev/ticket-printer
```

Con los symlinks en lugar, el agente siempre llega al dispositivo correcto independientemente del orden de arranque.

**5. Registrar la impresora en CUPS**

CUPS permite que otras aplicaciones impriman documentos (PDF, imágenes) a la impresora sin necesidad de abrir `/dev/usb/lp*` directamente.

```bash
# Obtener el URI exacto de la impresora (buscar la línea que diga "usb://")
sudo lpinfo -v | grep -i usb
# Ejemplo de salida:
# direct usb://Gainscha/GA-E200I?serial=111111151111   ← ticket (DIG-K200L)
# direct usb://GAINSCHA/GS-2406T?serial=001            ← etiquetas

# Registrar la impresora en CUPS con el URI encontrado
sudo lpadmin -p DIG-K200L -E \
  -v "usb://Gainscha/GA-E200I?serial=111111151111"

# Verificar que quedó activa
lpstat -p DIG-K200L
# debe mostrar: printer DIG-K200L is idle.
```

> **No usar `-m everywhere`** — ese flag requiere que la impresora soporte IPP (protocolo de red). La DIG-K200L es USB puro y no lo soporta; el comando falla con *"IPP Everywhere driver requires an IPP connection"*.

**6. Actualizar `print-agent.env`**

```
TICKET_DEVICE=/dev/usb/lp2       # para /print/ticket y /print/pdf (ESC/POS)
```

**Nota — comportamiento del driver USB en Linux**

El driver `usblp` del kernel procesa solo el primer paquete bulk de un `write()` grande y descarta el resto. Si el agente escribiera todo el ticket en un único `write()`, solo imprimiría las primeras líneas. Por eso `sendToDevice` escribe línea por línea: cada llamada a `write()` entrega exactamente una línea al driver, replicando el comportamiento de `printf ... | tee /dev/usb/lpN`. En Windows este problema no ha sido observado — si apareciera un síntoma similar (solo imprime la primera línea), la causa y el fix son los mismos.

---

## Conectar cajón monedero (DIG-KR410)

El DIG-KR410 se conecta a la impresora de tickets mediante un cable RJ11 — no va al PC directamente.

### Conexión física

1. Enchufa el cable del cajón en el puerto **"Cash Drawer"** de la DIG-K200L. Este puerto es **RJ11** (el conector pequeño de 4/6 pines), situado en la parte trasera de la impresora. No confundir con el puerto RJ45 (el grande, igual al de red) — conectar ahí hace que el cajón no responda aunque el software envíe el comando correctamente.
2. El otro extremo va al cajón monedero.
3. Gira la llave del cajón a la posición **B — Electrónica en Línea**. En esa posición el cajón queda bajo el control de la impresora. Las posiciones A (apertura manual) y C (bloqueado) no responden a comandos ESC/POS.

> **Diagnóstico rápido si el cajón no abre:** (1) verifica que el cable está en el puerto **RJ11** de la impresora, no en el RJ45; (2) comprueba que la llave del cajón está en posición **B**.

### Smoke test

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:5000/drawer/open \
  -H "Authorization: Bearer cambia-esto"
```

Debe devolver `204`. Si el cajón no abre físicamente pero el comando devuelve 204, revisa que la llave esté en posición B.

### Cómo funciona según la ruta de impresión

| Ruta | Quién abre el cajón | Cómo |
|---|---|---|
| **Ghostscript** (agente `POST /print/pdf`) | El agente Go | El drawer kick ESC/POS se envía automáticamente al final de cada impresión |
| **CUPS** (browser → Ctrl+P) | El backend Java (`DialogFrameworkBean`) | Tras generar el PDF del ticket, el backend llama a `POST /drawer/open` del agente en background |

En ambos casos es transparente para el usuario: el cajón abre al imprimir sin ninguna acción adicional.

Si no hay cajón conectado o la URL del agente no está configurada en el company profile, el comando se ignora silenciosamente — no produce error ni afecta la impresión.

---

## Configurar el lector de códigos de barras

El lector funciona en modo **teclado HID**: cuando escanea un barcode simula pulsaciones de teclas. Si el lector está configurado con un layout de teclado que no sea inglés US, ciertos caracteres se remapean — por ejemplo, el guión `-` puede leerse como `'` en layouts españoles.

### Síntoma

El barcode imprime `LOVE-02568-856487` correctamente (verificable con cualquier app de escaneo en el móvil), pero el lector físico entrega `LOVE'02568'856487` al sistema.

### Solución A — Restaurar valores de fábrica (recomendada)

Busca en el manual del lector la sección **"Restaurar valores de fábrica"** (suele estar en el capítulo 1) y escanea el barcode de programación correspondiente. Esto restaura el layout a inglés US y limpia cualquier reemplazo de caracteres.

### Solución B — Cambiar solo el layout de teclado

Si no quieres perder otras configuraciones del lector, busca en el manual la sección **"Configuración de teclado para diferentes países"** y escanea el barcode de **English / USA**.

### Verificar que quedó correcto

Después de reprogramar el lector, abre un editor de texto, escanea una etiqueta y confirma que el resultado contiene `-` (guión) y no `'` (apóstrofe).

---

## Guía del usuario (instalar el servicio)

No se necesita instalar Go ni ninguna herramienta adicional. Solo el binario compilado y el archivo de configuración.

### Dirección IP — regla importante

El agente escucha en el PC donde está conectada la impresora. Cuando el backend (ADA) o el navegador corren en **otra máquina**, `localhost` no funciona — cada dispositivo tiene su propio `localhost`. Hay que usar la **IP local del PC con la impresora**.

```bash
# Linux — obtener IP local
hostname -I          # ej: 192.168.1.45
ip route get 1       # muestra la IP usada para salir a la red

# Windows
ipconfig             # buscar "Dirección IPv4" en el adaptador de red activo
```

Luego registrar esa IP en el perfil de empresa dentro de ADA:

```
http://192.168.1.45:5000   ← URL del agente a registrar en ADA
```

> Si la impresora puede estar en distintas redes o accederse desde fuera del local, usar la IP pública del router y abrir el puerto `5000` con redirección al PC de la impresora.

---

### Linux

Ruta de instalación: `/opt/print-agent`

`/opt` es el directorio estándar en Linux para software de terceros. Usar esta ruta garantiza que el servicio funciona independientemente del usuario que esté conectado.

El instalador registra un servicio systemd que apunta al binario en `/opt/print-agent`. **No mover ni borrar esa carpeta** después de instalar — el servicio arranca desde ahí cada vez que se enciende el PC.

```bash
# 1. Crear la carpeta de instalación y copiar los archivos
sudo mkdir -p /opt/print-agent
sudo cp print-agent     /opt/print-agent/
sudo cp print-agent.env /opt/print-agent/   # o copia print-agent.env.example y renómbralo

# 2. Editar la configuración
#    Ajustar: LABEL_DEVICE, TICKET_DEVICE, PORT, AGENT_SECRET
sudo nano /opt/print-agent/print-agent.env

# 3. Instalar el servicio (desde la carpeta de instalación)
cd /opt/print-agent
sudo ./print-agent --install
# → Crea /etc/systemd/system/print-agent.service apuntando a /opt/print-agent/print-agent
# → Habilita e inicia el servicio automáticamente

# Verificar que arrancó
journalctl -u print-agent -f    # Ctrl+C para salir
curl http://localhost:5000/health

# Actualizar el binario (cuando haya una versión nueva)
sudo systemctl stop print-agent
#recompilar si es necesrio
go build -o print-agent .
sudo cp print-agent /opt/print-agent/print-agent   # reemplazar el binario con el nuevo
sudo systemctl start print-agent
# Nota: no usar cp con el servicio corriendo — el SO bloquea el archivo ("Text file busy")

# Desinstalar (detiene y elimina el servicio; la carpeta queda intacta)
sudo /opt/print-agent/print-agent --uninstall
```

### Windows

Ruta de instalación: `C:\PrintAgent`

#### Paso 1 — Instalar las impresoras y sus drivers

Antes de instalar el agente, las impresoras deben estar instaladas en Windows y visibles en **Panel de control → Dispositivos e impresoras**. Instala el driver de cada impresora según las instrucciones del fabricante y verifica que aparezcan en la lista de impresoras del sistema.

Una vez instaladas, abre PowerShell y ejecuta este comando para identificar el nombre de puerto de cada impresora:

```powershell
Get-Printer | Select-Object Name, PortName, DriverName | Format-Table -Auto
```

La salida muestra algo como:

```
Name               PortName  DriverName
----               --------  ----------
Gainscha GS-2406T  USB002    Gainscha GS-2406T
POS Printer 203DPI USB001    POS Printer 203DPI Series
```

El valor de la columna **`Name`** (no `PortName`) es el que va en el `.env`:

- `LABEL_DEVICE=Gainscha GS-2406T`        ← nombre exacto de la impresora de etiquetas
- `TICKET_DEVICE=POS Printer 203DPI Series` ← nombre exacto de la impresora de tickets

> En Windows el agente usa la API del spooler (`winspool.drv`) con el nombre de la cola, no el puerto USB directo. Copia el nombre tal como aparece en la columna `Name`.

#### Paso 2 — Instalar Ghostscript (prerequisito para `/print/pdf`)

El endpoint `/print/pdf` rasteriza el PDF a imagen a 203 DPI y lo envía como bytes ESC/POS RAW al driver de la impresora. Los drivers POS solo aceptan ESC/POS — herramientas como SumatraPDF usan GDI y fallan silenciosamente con estos drivers. El agente usa **Ghostscript** para la rasterización, el mismo motor que en Linux.

1. Descarga el instalador desde [ghostscript.com/releases/gsdnld.html](https://ghostscript.com/releases/gsdnld.html) → sección **AGPL Release** → archivo `gs...w64.exe` (64 bits).
2. Instala con las opciones predeterminadas. No es necesario agregar al PATH — el agente detecta Ghostscript automáticamente en `C:\Program Files\gs\`.
3. Opcional: si lo instalaste en una ruta no estándar, configura `GS_BINARY` en `print-agent.env` con la ruta completa a `gswin64c.exe`.

> **Lección aprendida — no usar Windows.Data.Pdf (WinRT):** se intentó usar esta API integrada en Windows 10/11 para evitar instalar Ghostscript. La API requiere una sesión de usuario interactiva (Session 1+) y no funciona en el contexto de Windows Service (Session 0). El intento de migrar a Scheduled Task para salvar esta limitación resultó inestable. La solución correcta es Ghostscript: mismo motor que Linux, mismo código, cero complejidad adicional.

#### Paso 3 — Instalar el agente

Abrir PowerShell como Administrador.

```powershell
# 1. Crear la carpeta de instalación y copiar los archivos
New-Item -ItemType Directory -Force C:\PrintAgent
Copy-Item print-agent.exe   C:\PrintAgent\
Copy-Item print-agent.env   C:\PrintAgent\   # o copia print-agent.env.example y renómbralo

# 2. Editar la configuración con los puertos identificados en el paso anterior
notepad C:\PrintAgent\print-agent.env

# 3. Instalar como servicio Windows
C:\PrintAgent\print-agent.exe --install
# → Registra el servicio "PrintAgent" en el SCM de Windows
# → Lo inicia automáticamente (y en cada arranque del PC)

# Verificar que arrancó y que detecta las impresoras
curl http://localhost:5000/health
# Respuesta esperada: {"label_device":"ok","status":"ok","ticket_device":"ok"}

# Gestionar desde la consola de servicios o con sc.exe:
sc.exe stop  PrintAgent
sc.exe start PrintAgent

# Desinstalar
C:\PrintAgent\print-agent.exe --uninstall
```

> Los logs del servicio se escriben en `C:\PrintAgent\print-agent.log`. Útil para diagnosticar si una impresión no llega o falla.

#### Paso 4 — Abrir el puerto en el Firewall de Windows

El agente escucha en el puerto 5000. Windows Firewall bloquea por defecto las conexiones entrantes en ese puerto, por lo que otros equipos de la red (cajeros, tablets, Android) no pueden alcanzarlo aunque el servicio esté corriendo.

Desde PowerShell como Administrador:

```powershell
New-NetFirewallRule -DisplayName "POS Print Agent" -Direction Inbound -Protocol TCP -LocalPort 5000 -Action Allow
```

Verificar desde otro equipo de la red que el agente ya es accesible:

```bash
curl http://192.168.1.XX:5000/health
# Respuesta esperada: {"label_device":"ok","status":"ok","ticket_device":"ok"}
```

> **Lección aprendida:** el síntoma de un firewall bloqueando el puerto es `ERR_CONNECTION_TIMED_OUT` en el navegador y timeout en `curl` desde otro equipo, aunque el `curl http://localhost:5000/health` desde la misma máquina responda correctamente. Si el agente funciona en local pero falla desde cualquier otro dispositivo de la red, este es el primer punto a verificar.
