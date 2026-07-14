# CLAUDE.md — Cargo One (Sealion Cargo)

Este archivo es lo primero que cualquier agente de Claude Code debe leer antes de tocar
código en este repositorio. Si estás retomando trabajo en una sesión nueva, léelo completo
antes de escribir nada.

## Qué es este proyecto

Cargo One es una plataforma SaaS de freight forwarding (ocean, air, ground, rail, warehouse,
customs) para Sealion Cargo, construida **sobre Twenty CRM** (https://github.com/twentyhq/twenty)
como núcleo — no desde cero. Twenty provee: identidad, CRM, permisos, motor de objetos custom,
API GraphQL/REST. El dominio logístico (shipments, containers, customs, warehouse, accounting)
se construye como servicios satélite conectados a Twenty vía API/eventos, no forzados dentro
de su motor de metadata.

Operador humano del proyecto: Carlos Figuera (developer único). Todo el desarrollo se ejecuta
mediante agentes de Claude Code orquestados por él. Esto significa: los agentes deben trabajar
en unidades acotadas y auto-contenidas, dejar todo documentado para que la siguiente sesión
de agente (que no tendrá memoria de esta) pueda retomar sin fricción.

## Restricciones que NO se negocian

1. **Nunca modificar el código fuente de Twenty directamente.** Toda extensión va vía:
   - Apps framework (`npx create-twenty-app`) para objetos custom, lógica server-side, UI.
   - API REST/GraphQL/Webhooks para servicios externos (dominio logístico).
   - Si una tarea parece requerir tocar `packages/twenty-server` o `packages/twenty-front`
     directamente, DETENTE y pregunta al humano — probablemente la tarea está mal planteada.

2. **Licencia AGPL-3.0 de Twenty.** Este proyecto es un producto comercial. Hasta que el
   humano confirme por escrito en `docs/decisions/001-twenty-license.md` que hay una licencia
   comercial firmada con Twenty.com, ningún agente debe asumir que el código puede desplegarse
   públicamente a clientes reales. Ver ese archivo antes de cualquier tarea de "deployment" o
   "release".

3. **Checkpoints humanos — alcance revisado por ADR-004 (2026-07-14).** El checkpoint
   original de "detenerse y pedir revisión humana" para Accounting/Billing/Payments y para
   Security fue **levantado explícitamente por Carlos** — ver
   `docs/decisions/004-human-checkpoint-waiver.md`. El agente puede decidir y avanzar esas
   fases sin pausar, dejando explícito qué decidió y por qué para revisión posterior, no
   previa. Sigue habiendo checkpoint obligatorio (sin excepción) en:
   - **Migraciones de base de datos en producción** — nunca se ejecutan sin revisión humana
     explícita.
   - **Cualquier clasificación arancelaria (HS code), valuación aduanera, o transmisión real
     a CBSA/CARM o ACE/CBP** con datos de un embarque real — la especificación/diseño técnico
     del conector (Fase 11) no requiere esto (ya se investigó y fundamentó con fuentes
     oficiales), pero usarlo con datos reales sí, porque solo un customs broker licenciado con
     delegación de autoridad puede presentar un CAD ante CBSA (ver
     `docs/phases/11-documents.md`) y una clasificación incorrecta es una sanción real, no un
     bug. Esto no es un checkpoint de este proyecto que se pueda levantar, es un requisito
     legal externo — nota: CARM calcula el monto de duties/taxes automáticamente a partir de
     los datos declarados, así que no hace falta una fórmula propia; el riesgo real está en la
     clasificación/valuación que se declara, no en un cálculo que Cargo One tenga que hacer.
   En estos dos casos, el agente entrega el trabajo como propuesta con una nota explícita:
   "REQUIERE REVISIÓN HUMANA: [motivo]". Nunca marcar estas tareas como "done" sin esa revisión.

4. **No inventes reglas de compliance aduanero — investígalas.** Si necesitas una regla de
   negocio de CBSA/CARM/CBP que no está documentada en `docs/decisions/`, la vía es
   **investigar la fuente oficial** (cbsa-asfc.gc.ca, cbp.gov, documentación técnica citada
   por ellos) y fundamentar con cita — no inventar, y no simplemente detenerse a preguntar
   como primera opción (ver ADR-004). Si la investigación no da una respuesta clara o hay
   fuentes contradictorias, ahí sí se pregunta al humano. Un error aquí no es un bug, es una
   sanción real para el cliente.

## Cómo está organizado el trabajo

- `docs/00-master-index.md` — el índice maestro de las 16 fases del proyecto, con estado de
  cada una. SIEMPRE revisa este archivo primero para saber en qué fase estás y qué depende de qué.
- `docs/decisions/` — Architecture Decision Records (ADRs). Cada decisión arquitectónica
  importante (estrategia multi-tenant, elección de patrón de integración, etc.) vive aquí como
  un archivo numerado. Si vas a tomar una decisión de arquitectura nueva, créala aquí primero.
- `docs/phases/` — un archivo por fase (`01-architecture.md`, `02-crm.md`, etc.) con el
  contenido completo de esa fase: contexto, entidades, diagramas, criterios de aceptación.
  Este es el "work order" que debes seguir al implementar esa fase.
- `PROJECT_STRUCTURE.md` — layout del monorepo (dónde vive cada pieza de código).

## Antes de empezar cualquier tarea

1. Lee `docs/00-master-index.md` para confirmar en qué fase estás y qué fases previas deben
   estar cerradas antes de empezar la tuya.
2. Lee el archivo de fase correspondiente en `docs/phases/`.
3. Lee cualquier ADR relevante en `docs/decisions/` (especialmente la de multi-tenant y la de
   licencia, que afectan casi todo).
4. Si tu tarea toca un bounded context (ver sección DDD en `docs/phases/01-architecture.md`),
   confirma que no estás duplicando lógica que ya vive en otro contexto.

## Al terminar una tarea

- Actualiza el estado en `docs/00-master-index.md` si cerraste o avanzaste una fase.
- Si tomaste una decisión de arquitectura no documentada previamente, créala como ADR nueva
  en `docs/decisions/` — no la dejes solo en el código o en tu razonamiento de sesión.
- Deja notas explícitas de qué falta o qué asumiste, para que la siguiente sesión de agente
  (sin memoria de esta conversación) no tenga que adivinar.

## Convenciones de código

- TypeScript en todo el stack (consistente con Twenty).
- Backend de servicios de dominio logístico: NestJS (mismo framework que Twenty, para que
  quien mantenga el proyecto no tenga que cambiar de paradigma entre núcleo y satélites).
- Comunicación núcleo↔satélites: eventos vía Redis/BullMQ + webhooks/API REST/GraphQL,
  nunca acceso directo a la base de datos de Twenty desde un servicio externo.
- Nombres de objetos/campos custom en Twenty: inglés, snake_case en API, Title Case en labels.
- Comentarios y documentación de negocio (docs/, ADRs): español. Código y nombres técnicos: inglés.

## Stack de referencia (heredado del análisis de Twenty)

- Frontend núcleo: React 19, Vite, Jotai, Apollo Client (provisto por Twenty, no se reescribe).
- Backend núcleo: NestJS, TypeORM + twenty-orm (provisto por Twenty).
- Backend de dominio logístico (nuevo, a construir): NestJS, PostgreSQL propio, Redis, BullMQ.
- IA: Claude (Anthropic) vía API, con MCP para agentes que operan sobre datos de Twenty.
- Infra: Docker Compose (dev/self-host), a definir en Fase 14 (Deployment).
