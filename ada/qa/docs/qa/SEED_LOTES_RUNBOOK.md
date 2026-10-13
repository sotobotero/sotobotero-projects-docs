# Runbook: seed de lotes/vencimientos

Operativo y corto. Contexto y decisiones: [PLAN_SEED_LOTES_Y_PERFORMANCE.md](PLAN_SEED_LOTES_Y_PERFORMANCE.md).

## 1. Crear el tenant (clon de `qa`)

```bash
cd infraestructure-ascode/docker-compose/postgres/db/dbfiles/otherfiles
ADA_ENV_FILE=<ruta-a-tu-.env.debug.local> \
bash provision-tenant-local.sh db:qa public <NIT> "Demo Lotes" <email>
```

- Slug resultante: `de<NIT>` (dos primeras letras del nombre + NIT). Credenciales iniciales: ver `TENANT_PROVISIONING.md`.
- **No** usar `--clean-data`: el seed reutiliza personas, caja y productos de `qa`.
- Necesita `ADA_KEYCLOAK_*` en el `.env.debug.local` (crea el usuario en el Keycloak real) y `qa` sin conexiones activas (clone por `TEMPLATE`).

## 2. Sembrar

```bash
export PGPASSWORD=<password-postgres-local>
psql -h localhost -U <usuario> -d <slug> -v ON_ERROR_STOP=1 -f db/seed/demo-lotes.sql
```

Idempotente: si ya hay productos `DEMO-%` no hace nada. Crea 27 productos de droguería, condimentos e insumos y embutidos, repartidos en 3 líneas y 8 grupos, con 6 proveedores, 10 clientes (con su cuenta de facturación), 28 compras (ALBE) y 22 ventas (ALBS); fechas relativas a `CURRENT_DATE`. El kardex y el stock los llenan los triggers.

## 3. Limpiar / resembrar

```bash
psql -h localhost -U <usuario> -d <slug> -v ON_ERROR_STOP=1 -f db/seed/clean-demo-lotes.sql
```

Borra todo lo marcado (productos `DEMO-*`, facturas `DEMO-*`, terceros con `internal_code='DEMO'`, sus cuentas, y las líneas/grupos creados) y deja la base como el clon de `qa`. Luego se puede volver a correr el paso 2.

## 4. Verificar

```sql
SELECT l.name AS linea, g.name AS grupo, count(*) AS productos, sum(p.stock) AS unidades
FROM inventory.product p JOIN inventory."group" g ON g.id = p.group_id JOIN inventory.line l ON l.id = g.line
WHERE p.code LIKE 'DEMO-%' GROUP BY 1, 2 ORDER BY 1, 2;
-- esperado: 8 filas / 27 productos; ej. Embutidos cocidos = 3 productos (225 u.), Medicamentos de venta libre = 9 (321 u.)
```

Casos cubiertos: lote vencido (Acetaminofén, Jamón, Diclofenaco), vencen en +8/+10/+15/+25/+30/+45 días, uno en +120 (fuera del umbral de 90), harina con 2 lotes de 2 proveedores y un lote vendido a 3 clientes, leche con lote agotado (saldo 0) y termómetro sin lote.

## Supuestos del seed (si el clon cambia, se rompe aquí)

Solo se asume de `qa`: producto plantilla `562`, caja `1`, usuario `1` y shelf `1`. Líneas, grupos, productos y terceros los crea el propio seed.

## Pendiente

Validación en Grafana (el tenant aparece solo por membresía; ver `GRAFANA_ANALYTICS_ACCESS.md`) y validación de los dos dashboards. Modo volumen, E2E Playwright y alertas: ver el plan.
