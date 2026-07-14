# ADR 003 — Cómo se materializa `core/twenty` (sin submódulo del monorepo)

**Estado:** 🟢 Resuelto.

## Contexto

`PROJECT_STRUCTURE.md` (versión original del scaffold) describía `core/twenty/` como "un
submódulo o fork mínimo de Twenty (solo config, sin tocar su código fuente)", con
`twenty-apps/` y `docker-compose.override.yml` como hijos de esa carpeta.

Al intentar resolver ese bloqueo (ver `docs/phases/02-crm.md`, `docs/phases/03-customers.md`)
se probaron ambos caminos:

1. **Clonar el monorepo de Twenty como git submodule**, pinneado a `v2.9.0` — funcionó
   técnicamente (337 MB de working tree), pero al inspeccionar el propio monorepo
   (`packages/create-twenty-app`, `packages/twenty-sdk`) quedó claro que **no hace falta**:
   el flujo oficial soportado (`npx create-twenty-app@latest`) es un CLI de npm que:
   - Descarga `twenty-sdk`/`twenty-client-sdk` desde npm, sin necesidad del código fuente
     del monorepo.
   - Levanta un servidor Twenty local vía la imagen Docker publicada
     (`twentycrm/twenty:latest`), no compilando desde el monorepo clonado.
   - No hay forma de "instalar" ese submódulo dentro de sí mismo sin convertirlo en un fork
     (un submódulo puro no admite archivos locales no comiteados a su propio historial), lo
     que hubiera contradicho la restricción de `CLAUDE.md` de nunca modificar el código
     fuente de Twenty.
2. Se **revirtió** el submódulo y en su lugar se generaron las tres apps planeadas
   (`quotation-app`, `sales-extensions-app`, `integrations-hub-app`) directamente con
   `npx create-twenty-app@latest <nombre>` como proyectos npm independientes bajo
   `core/twenty-apps/` (sin anidar dentro de un `core/twenty/`, porque ya no existe tal
   carpeta).

## Decisión

- `core/twenty/` **no existe** como carpeta del repo. No se clona ni se referencia el
  monorepo de Twenty en ningún punto del proyecto — ni como submódulo ni como fork.
- Las apps viven directamente en `core/twenty-apps/<nombre-app>/`, cada una generada con
  `npx create-twenty-app@latest` y comiteada como proyecto npm normal (con su propio
  `package.json`, `src/`, etc. — sin `.git` anidado, se elimina tras generar).
- La referencia de auto-hosting de Twenty en producción (`docker-compose.yml` real, imagen
  `twentycrm/twenty`, Postgres 16, Redis) se retoma en Fase 14 (Deployment) copiando o
  referenciando `packages/twenty-docker/docker-compose.yml` del repo oficial en ese momento
  — no se vendorea antes de que haga falta.
- `PROJECT_STRUCTURE.md` se actualiza para reflejar esta estructura real (ver diff de esta
  misma sesión).

**Razonamiento:** menos peso en el repo (nada de un monorepo de cientos de MB para un
proyecto que es, ante todo, un portfolio personal con esta carpeta como fallback — ver
histórico de la sesión), cero riesgo de tocar accidentalmente código fuente de Twenty porque
ni siquiera está presente, y alineado 1:1 con el flujo oficial soportado por Twenty para
construir Apps (`docs.twenty.com/developers/extend/apps`).

## Regla para agentes

- No volver a agregar `core/twenty` como submódulo/clon del monorepo salvo que una fase
  futura lo justifique explícitamente (por ejemplo, si Fase 14 necesita inspeccionar el
  código fuente para un self-host avanzado) — y en ese caso, documentarlo como una ADR nueva,
  no reabrir esta.
- Cualquier app nueva bajo `core/twenty-apps/` se crea con `npx create-twenty-app@latest`,
  nunca a mano — para mantener consistencia con el CLI oficial (`twenty-sdk`) y no
  reinventar la estructura de proyecto.
- `yarn install` y el arranque del servidor local (`yarn twenty dev`, requiere Docker)
  **no se pueden completar en un sandbox sin daemon de Docker ni acceso de red completo a
  registries de yarn/corepack** — ver `core/twenty-apps/README.md` para el detalle exacto
  del bloqueo encontrado en esta sesión. Cualquier sesión futura que retome esto debe
  verificar primero si tiene Docker y red completa antes de asumir que puede continuar.
