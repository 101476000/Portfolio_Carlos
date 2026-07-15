# CLAUDE.md — Cargo One (Sealion Cargo)

Este archivo es lo primero que cualquier agente de Claude Code debe leer antes de tocar
código en este repositorio. Si estás retomando trabajo en una sesión nueva, léelo completo
antes de escribir nada.

## Qué es este proyecto

Cargo One es una plataforma **SaaS multi-tenant** de freight forwarding (ocean, air, ground,
rail, warehouse, customs), construida **sobre Twenty CRM**
(https://github.com/twentyhq/twenty) como núcleo — no desde cero. Twenty provee: identidad,
CRM, permisos, motor de objetos custom, API GraphQL/REST. El dominio logístico (shipments,
containers, customs, warehouse, accounting) se construye como servicios satélite conectados a
Twenty vía API/eventos, no forzados dentro de su motor de metadata.

**Aclaración importante de producto (no es negociable, afecta cómo se diseña todo lo
demás):** Cargo One **no es software a medida para un solo cliente**. Sealion Cargo es la
empresa de Carlos y el primer tenant/cliente ancla del producto, pero **cualquier freight
forwarder que contrate el servicio debe poder crear su propio workspace y configurar los
datos de su propia empresa ahí** — sin que eso requiera tocar código ni que un dato de un
tenant contamine o dependa de otro. Esto ya estaba reflejado en la decisión de arquitectura
(ver ADR-002, multi-tenancy schema-per-tenant), pero se documenta aquí de forma explícita
porque varias fases fueron escritas asumiendo implícitamente a Sealion Cargo como si fuera
el único cliente — ver `docs/decisions/005-multi-tenant-product-clarification.md` para el
detalle de qué se corrigió y por qué.

**Principios de producto adicionales (Carlos, misma ronda que la aclaración multi-tenant):**
- **User-friendly e intuitivo.** Ninguna pantalla debería requerir leer documentación técnica
  para usarse — lenguaje llano en vez de nombres de campo técnicos expuestos al usuario final,
  valores por defecto razonables, errores claros en el momento. Ver el detalle aplicado a la
  pantalla de configuración de IA en `docs/phases/12-ai-platform.md`, sección "Principio de
  UX" — el mismo criterio aplica a cualquier pantalla nueva del proyecto, no solo esa.
- **IA como funcionalidad central, no accesorio**, para agilizar el trabajo del usuario
  (redactar cotizaciones, asistir en clasificación de documentos, responder preguntas sobre
  el estado de un shipment). Cada tenant configura sus propias credenciales del proveedor de
  IA que prefiera usar (Claude, ChatGPT, u otro) — ver `docs/phases/12-ai-platform.md`
  (`AIProviderConfig`) y `docs/phases/11-documents.md` (`Connector`/`ConnectorCredential`,
  de donde `AIProviderConfig` reutiliza el mecanismo de credenciales, no uno nuevo). La IA
  **no** está exenta de los checkpoints humanos de este archivo — ver regla explícita en
  Fase 12.

Operador humano del proyecto: Carlos Figuera (developer único, opera Sealion Cargo). Todo el
desarrollo se ejecuta mediante agentes de Claude Code orquestados por él. Esto significa: los
agentes deben trabajar en unidades acotadas y auto-contenidas, dejar todo documentado para que
la siguiente sesión de agente (que no tendrá memoria de esta) pueda retomar sin fricción.

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
- **Nombres de campos/objetos custom en Twenty (corregido contra el servidor real, no
  asumido):** `name` en **camelCase**, no snake_case — el servidor de Twenty rechaza
  guiones/underscores en el `name` de un `FieldMetadata` ("must start with lowercase letter
  and contain only alphanumeric letters"). Labels en Title Case ("Company Role"). Valores de
  opciones de `SELECT`/`MULTI_SELECT` en **UPPER_CASE** snake_case (ej. `CUSTOMS_BROKER`), no
  minúsculas — también rechazado por el servidor si no. Esta regla se descubrió recién en la
  sesión que implementó el primer campo real (`company_role` → `companyRole` en
  `sales-extensions-app`, Fase 2) — antes de eso, la documentación de fases (2, 3, 4) tenía
  los valores de opciones en minúscula; se están corrigiendo a medida que se implementa cada
  campo, no todas de una vez.
- Comentarios y documentación de negocio (docs/, ADRs): español. Código y nombres técnicos: inglés.

## Stack de referencia (heredado del análisis de Twenty)

- Frontend núcleo: React 19, Vite, Jotai, Apollo Client (provisto por Twenty, no se reescribe).
- Backend núcleo: NestJS, TypeORM + twenty-orm (provisto por Twenty).
- Backend de dominio logístico (nuevo, a construir): NestJS, PostgreSQL propio, Redis, BullMQ.
- IA: Claude (Anthropic) vía API, con MCP para agentes que operan sobre datos de Twenty.
- Infra: Docker Compose (dev/self-host), a definir en Fase 14 (Deployment).
