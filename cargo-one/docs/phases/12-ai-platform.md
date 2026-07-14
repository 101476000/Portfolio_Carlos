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

### Entidad: `AIProviderConfig` (relación many-to-one a `Connector` de Fase 11) — **dónde se configuran las credenciales de IA**

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant — cada tenant configura sus propios proveedores, igual que `OrganizationProfile` (Fase 2, ADR-005) |
| `connector_id` | UUID | FK a `Connector` (Fase 11, Integrations Hub) — **no se inventa un mecanismo de credenciales nuevo**: un proveedor de IA es un `Connector` más, con `provider` extendido a `anthropic_claude`, `openai_chatgpt`, `llm_other` (ver actualización en Fase 11). La API key vive en `ConnectorCredential.credential_reference`, exactamente igual que la credencial de un conector de aduana o de Amazon SP-API |
| `default_model` | texto | Ej. `claude-sonnet-5`, `gpt-5` — texto libre, no enum: los nombres de modelo cambian con frecuencia y no tiene sentido migrar el schema cada vez |
| `is_active` | booleano | Si este proveedor está habilitado para usarse ahora mismo |
| `priority` | entero | Orden de fallback si hay más de un proveedor configurado (ej. Claude como principal, ChatGPT como respaldo) — sin lógica automática de fallback definida en esta fase, solo el dato de orden |

**Por qué se modela así y no con un objeto de credenciales separado:** Integrations Hub
(Fase 11) ya resolvió "cómo guardamos una referencia a un secreto de un sistema externo sin
guardar el secreto mismo" — un proveedor de IA (Claude, ChatGPT, u otro) es, para efectos de
este modelo, un sistema externo más. Reutilizar `Connector`/`ConnectorCredential` en vez de
crear `AIProviderCredential` evita tener dos lugares distintos donde un tenant configura
tokens, lo cual sería confuso — precisamente lo contrario de "user-friendly e intuitivo" que
pidió Carlos (ver "Principio de UX" abajo).

**Dónde vive la UI de configuración:** `integrations-hub-app` (Fase 11, ya escafoldada en
`core/twenty-apps/integrations-hub-app/`) es el lugar natural — ya es "el admin de tokens"
del proyecto (`PROJECT_STRUCTURE.md`). La pantalla de configuración de IA es una sección más
ahí, no una app nueva ni una pantalla separada que el usuario tenga que descubrir por su
cuenta.

## Casos de uso concretos (para qué sirve la IA en Cargo One, no solo que "hay IA")

Ninguno de estos se implementa en esta fase — es el modelo de datos y la regla de checkpoints
lo que se especifica aquí. Se documentan para que quede claro qué problema resuelve
`AgentSession`/`PromptTemplate`/`EmbeddingIndex`, y para que futuras fases de implementación
no tengan que adivinar el propósito:

- **Redacción asistida de cotizaciones** (Fase 4): a partir de un email o mensaje del cliente
  con la carga/ruta deseada, sugerir un borrador de `Quotation` con `QuotationLine` — el
  usuario revisa y confirma, no se envía nada automáticamente.
- **Asistencia en el pipeline de Documents** (Fase 11): `OcrResult`/`ClassificationResult`/
  `ExtractionResult` ya asumían un modelo detrás (`engine`, `model_version`) — ahora queda
  explícito que ese modelo puede ser el proveedor de IA configurado en `AIProviderConfig`.
  Sigue pasando siempre por `pending_approval` antes de que Customs/Accounting use el dato
  (esa regla no cambia).
- **Copiloto conversacional para operaciones** (ej. "¿en qué estado está el shipment X?",
  "resume los milestones de esta semana para el cliente Y") — lectura vía API/MCP sobre
  Shipment/Warehouse, sin acciones de escritura en dominios con checkpoint.
- **Sugerencia (no decisión) de clasificación arancelaria** en `CustomsDeclarationLine.hs_code`
  (Fase 11) — el modelo puede proponer un HS code candidato, pero queda como sugerencia
  editable; la captura y responsabilidad de la clasificación final sigue siendo humana o del
  broker, sin excepción (`CLAUDE.md` regla 3/4 no se relaja por tener IA disponible).

## Principio de UX (aplica a esta fase y, en general, a cualquier pantalla nueva del proyecto)

Carlos pidió explícitamente que el producto sea **user-friendly e intuitivo** — esto no es
específico de IA, pero se documenta aquí porque la configuración de proveedores de IA es la
pantalla nueva más concreta que esta ronda agrega. Regla práctica: cualquier pantalla de
configuración (empezando por `AIProviderConfig` en `integrations-hub-app`) debe poder
completarse sin que el usuario necesite leer documentación técnica — nombres de campo en
lenguaje llano (ej. "Proveedor de IA" y "Clave API", no `provider`/`credential_reference`
expuestos tal cual al usuario final), valores por defecto razonables donde aplique, y errores
de configuración (ej. una API key inválida) mostrados de forma clara en el momento, no
descubiertos después en un log. Esto es un principio de diseño para cuando se implemente la
UI, no una tarea de esta fase — no hay mockups ni componentes definidos aquí.

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
`AIInteractionLog`/`AIProviderConfig`, la regla de que los checkpoints humanos de `CLAUDE.md`
se aplican a nivel de capacidades del agente, el principio de UX para la pantalla de
configuración, y los casos de uso concretos que justifican el modelo.

**Explícitamente fuera de esta fase:**
- Elección de proveedor de vector store, framework de RAG, o arquitectura de agentes
  (single-agent vs. multi-agente) — decisiones técnicas de implementación.
- Qué herramientas MCP concretas se exponen a los agentes — se define cuando cada servicio
  satélite (Shipment, Warehouse, etc.) decida qué de su API expone vía MCP, no todas a la vez.
- Facturación de uso de IA a clientes — mencionado como campo informativo (`cost_estimate`),
  no implementado como flujo de negocio.
- Lógica real de fallback entre proveedores (qué pasa si `Claude` falla, ¿reintenta con
  `ChatGPT` automáticamente?) — `AIProviderConfig.priority` solo guarda el orden, la lógica de
  conmutación es de implementación.
- Mockups/componentes de UI concretos — el principio de UX queda como guía, no como diseño.
- El código del servicio (`services/ai-gateway-service`) y la extensión de
  `integrations-hub-app` para la pantalla de configuración.

## Criterios de aceptación

- [x] Modelo de datos definido, incluyendo `AIProviderConfig` reutilizando el patrón de
      credenciales de Integrations Hub (Fase 11) en vez de uno nuevo.
- [x] Regla de checkpoints heredados para agentes de IA documentada explícitamente.
- [x] Casos de uso concretos documentados (redacción de cotizaciones, asistencia en Documents,
      copiloto de operaciones, sugerencia de HS code) para que la implementación tenga
      objetivo claro, no "IA en general".
- [ ] `services/ai-gateway-service` implementado con las migraciones correspondientes.
- [ ] Pantalla de configuración de `AIProviderConfig` implementada dentro de
      `integrations-hub-app`, siguiendo el principio de UX (lenguaje llano, validación clara).
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
- `AIProviderConfig` depende de `Connector`/`ConnectorCredential` (Fase 11) — no crear un
  segundo lugar para guardar API keys de IA "porque es más simple". Si Integrations Hub
  cambia su patrón de credenciales en el futuro, esta fase lo hereda automáticamente por ser
  la misma tabla, no una copia.
- El modelo BYO-key (cada tenant trae su propia API key de Claude/ChatGPT/otro) es el diseño
  por defecto aquí, porque es lo que Carlos pidió explícitamente ("un sitio donde se
  configuren las credenciales del modelo de IA que se usará"). No se decidió (ni se descarta)
  un modo alternativo donde Cargo One provea una clave compartida con uso facturado — si se
  necesita en el futuro, es una decisión de producto nueva, no una que este documento asuma.
