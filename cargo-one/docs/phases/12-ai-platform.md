# Fase 12 — AI Platform

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**.

## Contexto

Depende de: Fase 1 (cerrada), Fase 11 (spec lista — AI Platform consume el pipeline de
Documents para RAG y el admin de credenciales de Integrations Hub para tokens de proveedores
LLM). Bounded context: **AI Platform**, fila 11 de `docs/phases/01-architecture.md` sección 3
— **generic domain**, consumidor transversal (vía API/MCP) de todos los demás contextos, no
dueño de datos de negocio. Vive en `services/ai-gateway-service`.

Stack de referencia (`CLAUDE.md`): Claude (Anthropic) vía API, con MCP para agentes que
operan sobre datos de Twenty.

## Modelo de datos

### Agregado raíz: `AgentSession`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `user_id` | UUID | Referencia a `Person`/usuario de Twenty (ACL) — quién inició la sesión |
| `started_at` / `ended_at` | timestamp, nullable | |
| `context_summary` | texto, nullable | Resumen de qué se conversó, no el historial completo (eso vive en `AIInteractionLog`) |

### Entidad: `PromptTemplate`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | |
| `name` | texto | |
| `template_text` | texto | |
| `version` | entero | Versionado simple — sin esto, cambiar un prompt en producción no es auditable |

### Entidad: `EmbeddingIndex`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | |
| `source_type` | texto | Ej. `document`, `shipment_note` — de qué contexto viene el contenido indexado |
| `source_id` | UUID | Referencia no forzada a la entidad origen (ej. `Document.id` de Fase 11) |
| `vector_reference` | texto | Puntero al vector store (ej. `pgvector`, servicio externo) — **no** se decide el motor de vectores en esta fase, es una decisión técnica de implementación |

### Entidad: `AIInteractionLog` (relación many-to-one a `AgentSession`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `session_id` | UUID | FK a `AgentSession` |
| `prompt` / `response` | texto | |
| `model_used` | texto | |
| `tokens_used` | entero, nullable | |
| `cost_estimate` | decimal, nullable | Informativo — no alimenta `Invoice` (Fase 10) automáticamente; si el negocio decide facturar uso de IA a clientes, es una decisión de Accounting que requiere su propio checkpoint, no se conecta aquí |

## Regla de diseño no negociable: los checkpoints de `CLAUDE.md` aplican también a agentes de IA

Esta es la decisión más importante de esta fase, y la razón por la que se documenta aparte en
vez de solo en la tabla de arriba: **cualquier agente que corra sobre `ai-gateway-service` y
que pueda ejecutar acciones (no solo responder preguntas) está sujeto exactamente a las mismas
restricciones de `CLAUDE.md` que un agente de Claude Code operado por Carlos.** En concreto:

- Un agente de IA **no puede** aprobar/ejecutar cálculos de duties/taxes/CAD, tocar
  Accounting/Billing/Payments, ni integrarse con CBSA/CARM/ACE/CBP de forma autónoma — esas
  acciones requieren el mismo checkpoint humano que ya aplica en Fases 10 y 11.
- Esto se implementa como una restricción de **capacidades del agente** (qué herramientas/MCP
  tools tiene disponibles), no como una instrucción de prompt que el modelo podría ignorar o
  que un usuario podría intentar puentear — la autorización vive en el gateway, no en el texto
  del prompt.
- Cualquier acción de un agente de IA sobre esos dominios se registra como propuesta
  pendiente de revisión (mismo patrón "REQUIERE REVISIÓN HUMANA" de `CLAUDE.md`), nunca se
  ejecuta directo.

## Alcance de esta fase

**Incluido:** modelo de datos de `AgentSession`/`PromptTemplate`/`EmbeddingIndex`/
`AIInteractionLog`, y la regla de que los checkpoints humanos de `CLAUDE.md` se aplican a
nivel de capacidades del agente, no de instrucciones.

**Explícitamente fuera de esta fase:**
- Elección de proveedor de vector store, framework de RAG, o arquitectura de agentes
  (single-agent vs. multi-agente) — decisiones técnicas de implementación.
- Qué herramientas MCP concretas se exponen a los agentes — se define cuando cada servicio
  satélite (Shipment, Warehouse, etc.) decida qué de su API expone vía MCP, no todas a la vez.
- Facturación de uso de IA a clientes — mencionado como campo informativo (`cost_estimate`),
  no implementado como flujo de negocio.
- El código del servicio (`services/ai-gateway-service`).

## Criterios de aceptación

- [x] Modelo de datos definido.
- [x] Regla de checkpoints heredados para agentes de IA documentada explícitamente.
- [ ] `services/ai-gateway-service` implementado con las migraciones correspondientes.
- [ ] Verificado (con un caso de prueba concreto, no solo revisión de código) que un agente
      de IA no puede ejecutar una acción de Accounting/Customs sin pasar por el mismo flujo
      de revisión humana que un agente de Claude Code.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado.

## Notas para el agente

- No hay checkpoint humano obligatorio para *especificar o construir* esta fase, pero el
  gateway que se construya aquí es el punto donde **otros** checkpoints (Accounting, Customs)
  se hacen cumplir a nivel de agentes de IA — un error de diseño acá debilita esos checkpoints
  indirectamente. Tratarlo con el mismo cuidado aunque no esté marcado igual en
  `docs/00-master-index.md`.
- Bounded context involucrado: **AI Platform** (generic domain, consumidor transversal).
  No duplica datos de negocio de otros contextos — todo lo que necesita lo consulta vía
  API/MCP en el momento, salvo lo que indexa explícitamente en `EmbeddingIndex`.
