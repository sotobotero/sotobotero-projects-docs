> **TEMPLATE** — Este archivo es el `README.md` raíz del repo público (ej. `sotobotero-projects-docs`).
> Copiarlo manualmente al crear el repo. No es publicado automáticamente por el workflow.

# @sotobotero — Documentación de Proyectos

> Documentación técnica y operativa de los proyectos del ecosistema.  
> Publicado automáticamente desde los repositorios privados; no contiene código fuente.

## Proyectos

| Proyecto | Descripción |
|---|---|
| [ADA Platform](ada/) | Plataforma SaaS multitenant — gestión de negocios, facturación, agenda, booking |

---

## ADA Platform

| Sección | Qué cubre | Repo origen |
|---|---|---|
| [Backend](ada/backend/) | API REST, GraphQL, módulos, contratos, guías operativas | `ADA_ENTERPISE_CORE` |
| [Frontend](ada/frontend/) | Microfrontends, Module Federation, rutas entre shells | `arcadia` |
| [Infraestructura](ada/infrastructure/) | Docker Compose, Swarm, Nginx, Keycloak, Vault | `infraestructure-ascode` |

### Highlights

- [Patrón de disponibilidad para booking](ada/backend/docs/BOOKING_AVAILABILITY_PATTERN.md)
- [Arquitectura multitenant](ada/backend/docs/TENANT_CONTEXT_ARCHITECTURE.md)
- [Tests E2E manuales — caso gimnasio](ada/backend/docs/e2e-gym-manual-tests.md)
- [API GraphQL — queries de referencia](ada/backend/docs/graphql/)
- [Arquitectura de microfrontends](ada/frontend/MICROFRONTEND-CONFIG-ARCHITECTURE.md)

---

## Cómo se actualiza esta documentación

Cada repositorio privado sincroniza su carpeta automáticamente en cada push a `main/master/development`.
Nadie edita este repo directamente salvo el `README.md` raíz.

```
ADA_ENTERPISE_CORE/docs/  ─────────► ada/backend/
arcadia/frontend/*.md     ─────────► ada/frontend/
infraestructure-ascode/   ─────────► ada/infrastructure/
```

> Si encuentras documentación desactualizada, el fix va en el repo fuente — nunca aquí directamente.
