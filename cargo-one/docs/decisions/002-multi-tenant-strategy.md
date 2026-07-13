# ADR 002 — Estrategia Multi-Tenant

**Estado:** 🟢 Resuelto (Fase 1). Ver análisis completo en `docs/phases/01-architecture.md`,
sección 2.

## Contexto

Ver `docs/phases/01-architecture.md`, sección 2, para el análisis completo de las tres
opciones evaluadas:

- **Opción A** — PostgreSQL Row Level Security
- **Opción B** — Schema per Tenant (Twenty ya usa este patrón nativamente para sus propios
  workspaces — `workspace_{uuid}`)
- **Opción C** — Database per Tenant

## Decisión

**Opción B — Schema per Tenant** para los servicios de dominio logístico (`services/*`),
usando el mismo `workspace_id` (UUID) que asigna Twenty como identificador de tenant, con
convención de nombre de schema `logistics_{workspace_uuid}` en la base de datos propia de
cada servicio.

**Razonamiento (resumen — detalle completo en `docs/phases/01-architecture.md` sección 2):**
- Consistencia operativa con el patrón nativo de Twenty (un solo modelo mental para núcleo y
  satélites, relevante con un developer único orquestando agentes).
- Aislamiento más fuerte que RLS, apropiado para un dominio con datos de aduana y facturación
  donde una fuga entre tenants es una violación de compliance, no solo un bug.
- Evita el sobre-costo operativo de "database per tenant" para un negocio B2B de volumen medio
  (decenas/cientos de forwarders, no millones de usuarios finales).
- Escape hatch: un tenant puntual que exija aislamiento físico total puede "promoverse" a
  base de datos dedicada sin cambiar el modelo de datos ni el código de aplicación.

**Backup/migración:** backup por tenant vía `pg_dump --schema=logistics_{uuid}`; migraciones
corren iterando sobre los schemas de tenant activos (runbook detallado a definir en Fase 14).

## Regla para agentes

Ninguna fase de dominio logístico (5 en adelante) debe empezar a definir modelos de datos
sin seguir esta decisión: toda tabla de un servicio satélite vive dentro del schema
`logistics_{workspace_uuid}` correspondiente al tenant, nunca en un schema compartido entre
tenants.
