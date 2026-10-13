# Plan: datos de prueba de lotes/vencimientos + base para performance y E2E

**Estado:** fase 1 — seed del escenario y limpieza hechos y verificados en el tenant demo (2026-10-06); falta datasource Grafana y validar dashboards · **Contexto:** reportes Grafana "Inventario — Vencimientos" y "Inventario — Trazabilidad por lote" (repo `infraestructure-ascode`, `docker-compose/services/grafana/provisioning/dashboards/inventario/`). Compromiso comercial: Cotización Droguería Ahorro y Salud v1 (trazabilidad por lote, alertas de vencimiento, analítica básica).

Referencia de modelo: `docs/KARDEX_LOT_EXPIRATION_ARCHITECTURE.md` (el kardex se alimenta por trigger desde `standard_document_detail`; ALBE=14 entrada, ALBS=15 salida; kardex type 19 entrada / 20 salida).

## Objetivo

Un generador de datos repetible, con dos modos, que sirva a tres consumidores: validar los dashboards/alertas de Grafana, correr un E2E de humo con Playwright, y medir rendimiento de consultas con volumen.

## Decisiones tomadas

| Tema | Decisión |
|---|---|
| Mecanismo | SQL directo, insertando documentos (`standard_document` + `standard_document_detail` + join con `invoice_buy` / `billing.invoice`). El kardex y `product.stock` los llenan los triggers existentes; **no** insertar en `kardex` a mano. |
| Base de datos | Tenant nuevo, clonado de `qa` con `provision-tenant-local.sh db:qa public <NIT> "Demo Lotes"` (sin `--clean-data`, para reutilizar clientes, cajas y productos). El script deriva el slug: **`de<NIT>`** (no admite `demo_lotes`). No se siembra sobre `qa`. |
| Marca de datos | Prefijo `DEMO-` en códigos de producto, lotes y números de factura, para poder limpiar. |
| Fechas | Relativas a `CURRENT_DATE` (el escenario no caduca con el tiempo). |
| Selenium | No. Se usa Playwright (ya existe en `tests/playwright`). |
| Orden de producción | Fuera del alcance de la fase 1 (los reportes leen compras/ventas/kardex). |

## Ubicación de los entregables (repo ADA_ENTERPISE_CORE)

```
db/seed/demo-lotes.sql            # modo escenario (correctitud) — idempotente
db/seed/demo-lotes-volumen.sql    # modo volumen (generate_series, parametrizado)
db/seed/clean-demo-lotes.sql      # borra todo lo marcado DEMO-
tests/playwright/specs/lotes-trazabilidad.spec.ts   # E2E de humo
tests/performance/                # (fase 3, opcional) k6 contra el API
```

Los dashboards de Grafana siguen en `infraestructure-ascode`; el datasource `demo_lotes` se agrega en su `provisioning/datasources/default.yml`.

## Escenario (modo correctitud, ~6 productos)

| Caso | Qué valida |
|---|---|
| Lote vencido con saldo | contador "vencidos" |
| Lotes que vencen en +15 y +45 días | umbrales 30 y 60 |
| Lote que vence en +120 días | queda fuera del umbral de 90 |
| Mismo producto, 2 lotes de proveedores distintos | origen en trazabilidad |
| Un lote vendido a 3 clientes; otro agotado (saldo 0) | destino y saldo |
| Producto sin lote ni vencimiento | los reportes no se rompen |

## Modo volumen (tope de referencia: 3.000 referencias, según la cotización)

Parámetros: nº de productos, nº de lotes por producto, nº de ventas, rango de fechas. Objetivo: verificar que las consultas de los dashboards (joins sobre `standard_document_detail`) siguen respondiendo en tiempos aceptables con cientos de miles de movimientos; revisar `EXPLAIN ANALYZE` y añadir índices si hace falta (candidatos: `standard_document_detail(lot)`, `kardex(product_id, lot)`).

## Fases

1. **Escenario + validación en Grafana.** Escribir `demo-lotes.sql` y `clean-demo-lotes.sql`; provisionar `demo_lotes` local; agregar datasource; comprobar que ambos dashboards muestran los casos esperados. Antes de escribir, leer la estructura exacta de `accounting.document`, `standard_document`, `invoice_buy` y `billing.invoice` y el trigger `f_standard_document_detail_manage_kardex`.
2. **Volumen.** `demo-lotes-volumen.sql`, medición de tiempos, índices si aplica.
3. **E2E Playwright.** Spec corto: compra con lote → venta → ver el lote en el dashboard de trazabilidad. Copiar convención de los specs DIAN existentes (login Keycloak, `.env`).
4. **(Opcional) k6** en `tests/performance/` si se quiere medir el API de facturación bajo carga.
5. **Alertas Grafana** reutilizando las consultas: alertar si hay filas con `Días` ≤ umbral en la tabla de vencimientos.

## Pendientes / riesgos

- Gap conocido: el dropdown de lotes en factura/albarán llama a un stub (`findLotesBYProductIdMoventType`, ver sección "Gaps conocidos" del doc de kardex). Un E2E por UI que dependa de ese dropdown puede fallar por eso, no por el seed.
- El sistema no bloquea ventas de lotes vencidos; el reporte de vencimientos es la única señal.
- Verificar con el kardex real si las salidas conservan `lot`/`expiration` (en `qa` hoy las salidas tienen `lot` vacío), porque la columna "Vendido" del reporte de trazabilidad depende de `standard_document_detail.lot` en el albarán de salida.
