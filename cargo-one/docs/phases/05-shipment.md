# Fase 5 — Shipment

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**: `services/shipment-service/` todavía no existe como proyecto.

## Contexto

Depende de: Fase 1 (cerrada). No depende de Fases 2-4 para poder especificarse (viven en
bounded contexts distintos), pero sí las **consume** vía evento (ver más abajo).

Bounded context: **Shipment**, fila 2 de la tabla de `docs/phases/01-architecture.md`
sección 3 — el **shared kernel** del que dependen Ocean (Fase 6), Air (Fase 7), Ground
(Fase 8) y, parcialmente, Warehouse (Fase 9). Es el **core domain** del proyecto: la razón de
ser de Cargo One frente a operar sobre Twenty sin extensión logística.

**Por qué esta fase pide revisión de modelo de datos** (`docs/00-master-index.md` la marca
distinto a Fases 2-4: "Sí, con revisión de modelo de datos"): todo lo que se construya encima
en Fases 6-11 hereda las decisiones de esquema que se tomen aquí. Un error de modelado en
`Shipment` se propaga a seis fases más. Por eso esta fase entrega el modelo completo pero dos
veces marcada como pendiente de revisión humana antes de que una sesión de agente empiece a
escribir migraciones reales — ver "Notas para el agente".

A diferencia de Fases 2-4 (que viven dentro de Twenty, vía Apps framework), `Shipment` vive en
`services/shipment-service`, un satélite NestJS con su propia base de datos Postgres —
schema `logistics_{workspace_uuid}` por tenant, según `docs/decisions/002-multi-tenant-strategy.md`.

## Modelo de datos

### Agregado raíz: `Shipment`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant — igual al `workspace_id` de Twenty (ADR-002) |
| `shipment_number` | texto, único por tenant | Identificador legible para el cliente |
| `status` | enum | `draft`, `booked`, `in_transit`, `arrived`, `delivered`, `cancelled` — máquina de estados genérica de modo; cada modo (Fase 6-8) puede tener sub-estados propios en su propia tabla, no aquí |
| `transport_mode` | enum | `ocean`, `air`, `ground`, `multimodal` — mismo catálogo que `Quotation.transport_mode` (Fase 4) |
| `quotation_id` | UUID, nullable | Referencia al `Quotation` de Twenty que originó este shipment (ACL — ver sección Integración). Nullable porque un shipment puede crearse sin cotización previa (booking directo/spot) |
| `customer_company_id` | UUID | Referencia al `Company` de Twenty (rol `shipper` o `consignee`, Fase 2) — ACL, no se duplica el dato |
| `trade_lane_id` | UUID, nullable | Referencia al `Trade Lane` de Twenty (Fase 2) — ACL |
| `incoterm` | texto | Copiado del `Quotation` al crear el shipment (snapshot, no referencia viva — un shipment no debe cambiar si alguien edita una cotización vieja) |
| `created_at` / `updated_at` | timestamp | |

### Entidad: `Booking` (relación many-to-one a `Shipment`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID | FK a `Shipment` |
| `booking_number` | texto | Confirmación del carrier |
| `carrier_name` | texto | Texto libre en esta fase — el modelo estructurado de carrier/vessel/flight es de Fase 6/7/8, no se inventa aquí |
| `requested_pickup_date` / `confirmed_pickup_date` | fecha | |
| `status` | enum | `requested`, `confirmed`, `cancelled` |
| `is_current` | booleano | Un shipment puede tener varias `Booking` a lo largo del tiempo (rebooking tras cancelación) — exactamente una con `is_current = true` es la vigente. Evita que la app tenga que inferir "cuál booking manda" por fecha |

### Entidad: `CargoUnit` (relación many-to-one a `Shipment`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID | FK a `Shipment` |
| `unit_type` | enum | `container`, `package`, `pallet`, `loose_cargo` — genérico; el tipo ISO de contenedor (Fase 6) o detalles de AWB (Fase 7) van en tablas de extensión de modo, no aquí |
| `identifier` | texto, nullable | Número de contenedor u otro identificador físico, si ya se conoce en este punto. Único dentro del shipment, **no globalmente** — un número de contenedor se reutiliza entre distintos shipments a lo largo del tiempo, no es un identificador universal |
| `weight_kg` / `volume_cbm` | decimal | Peso/volumen de esta unidad de carga individual (no el total del shipment — el total es la suma de sus `CargoUnit`, calculado en la capa de aplicación, no almacenado) |
| `description` | texto | Descripción de la carga (no es la descripción arancelaria — eso es Fase 10-11, Customs) |

### Entidad: `Milestone` (relación many-to-one a `Shipment`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID | FK a `Shipment` |
| `milestone_type` | **texto, no enum de base de datos** | Ver razonamiento abajo — deliberadamente no es un enum cerrado |
| `occurred_at` | timestamp | |
| `location` | texto, nullable | |
| `source` | texto | Qué sistema/servicio reportó el hito (ej. `ocean-service`, `customs-service`, `manual`) — trazabilidad, no lógica |

**Por qué `milestone_type` no es un enum cerrado (corrección de autorevisión):** la primera
versión de este documento lo definía como enum con un valor `customs_cleared` incluido
directamente en el shared kernel. Eso es un error de bounded context: obliga a migrar
`Shipment` (que Fases 6-11 heredan) cada vez que Customs, Warehouse o cualquier fase futura
necesite registrar un tipo de hito nuevo — exactamente lo que la sección "Por qué esta fase
pide revisión de modelo de datos" advierte que hay que evitar. En su lugar:
- `milestone_type` es texto libre, validado en la capa de aplicación, no en el schema.
- El **núcleo genérico** que sí pertenece a esta fase (mode-agnóstico): `booked`, `picked_up`,
  `departed_origin`, `arrived_destination`, `delivered`.
- Cualquier otro contexto (Customs, Warehouse, Ocean/Air/Ground) usa su propio namespace al
  emitir milestones — ej. `customs.cleared`, `warehouse.received` — sin tocar esta tabla ni
  este documento.

### Entidad: `Party` (relación many-to-one a `Shipment`)

**Corrección de autorevisión:** la primera versión llamaba a esto "Value Object", pero un VO
no tiene identidad propia ni cardinalidad múltiple direccionable — esto sí la tiene (varias
filas por shipment, cada una editable/eliminable independientemente), así que es una Entidad
dentro del agregado, no un VO. Se corrige también la falta de PK.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK (faltaba en la versión anterior) |
| `shipment_id` | UUID | FK a `Shipment` |
| `role` | enum | `shipper`, `consignee`, `notify_party`, `forwarder_agent` |
| `company_id` o `person_id` | UUID | Referencia a Twenty (`Company`/`Person`) — ACL, nunca se copian nombre/dirección/etc. localmente; si se necesita mostrarlos, se consultan a la API de Twenty en el momento |

**Cardinalidad por rol** (regla de aplicación, no de schema): a lo sumo un `shipper` y a lo
sumo un `consignee` por shipment; `notify_party` y `forwarder_agent` pueden repetirse. No se
modela como constraint de base de datos en esta fase — se valida en el servicio, para no
atarse a una regla de negocio que podría tener excepciones no anticipadas.

## Integración

- **Consume** el evento `quotation.accepted` documentado en `docs/phases/04-sales.md`
  ("Notas para el agente"): al recibirlo, crea un `Shipment` en estado `draft` con
  `transport_mode`, `customer_company_id`, `trade_lane_id`, `incoterm` y `quotation_id`
  tomados del payload del evento. Este es el consumidor real que Fase 4 dejó pendiente —
  con esta fase especificada, ya existe "alguien" para quien Fase 4 puede implementar el
  publisher cuando ambas se codifiquen.
- **Lee** datos de `Company`/`Person`/`Trade Lane` desde Twenty vía API GraphQL con token
  scoped al workspace (nunca acceso directo a su base de datos — regla de
  `docs/phases/01-architecture.md` sección 4). No se cachea localmente en esta fase; si el
  volumen de consultas lo justifica, se evalúa cache de solo-lectura en una fase posterior,
  no se pre-optimiza aquí.
- **Emite** (contrato documentado, publisher no implementado hasta que exista consumidor
  real en Fase 6-11): `shipment.created`, `shipment.milestone.recorded` — mismo envelope de
  `docs/phases/01-architecture.md` sección 4.
- **Creación manual (spot booking):** además de reaccionar a `quotation.accepted`, el
  servicio expone una vía de creación directa (API) para shipments sin cotización previa —
  mismo conjunto de campos obligatorios en `Shipment` (`transport_mode`,
  `customer_company_id`; `quotation_id`/`trade_lane_id` quedan `null`, `incoterm` se captura
  a mano). No es un flujo distinto en el modelo, solo un origen distinto del mismo agregado.

## Alcance de esta fase

**Incluido:** modelo de datos completo de `Shipment`/`Booking`/`CargoUnit`/`Milestone`/`Party`
descrito arriba, contrato de consumo del evento `quotation.accepted`, contratos de eventos
salientes (documentados, no implementados).

**Explícitamente fuera de esta fase:**
- Cualquier campo específico de modo (tipo ISO de contenedor, AWB, ruta terrestre) — Fases
  6, 7, 8 respectivamente, cada una en su propia tabla de extensión referenciando
  `shipment_id`, sin modificar las tablas de esta fase.
- Warehouse (Fase 9) — aunque `Shipment` puede eventualmente relacionarse con un
  `WarehouseReceipt`, esa relación se define en Fase 9, no aquí.
- Cualquier cálculo de duties/taxes/CAD o integración con CBSA/CARM/ACE/CBP — Customs
  (Fase 10-11, checkpoint humano obligatorio).
- Cualquier lógica de facturación sobre el shipment — Accounting (Fase 10, checkpoint humano
  obligatorio).
- El código del servicio en sí (proyecto NestJS, migraciones, endpoints) — esta fase entrega
  el modelo de datos y los contratos de integración; la implementación se hace en una sesión
  posterior, después de que el modelo sea revisado por el humano (ver "Notas para el agente").

## Criterios de aceptación

- [x] Modelo de datos de `Shipment` y sus entidades relacionadas definido con campos, tipos y
      justificación de cada decisión de scope (qué queda fuera y por qué).
- [x] Relación con Fase 4 (`quotation.accepted`) y con Fase 2 (`Trade Lane`) definida vía ACL,
      sin duplicar datos que ya viven en Twenty.
- [ ] **Revisión humana del modelo de datos** (Carlos) — ver nota abajo, no marcar 🟢 sin ella.
- [ ] `services/shipment-service/` creado como proyecto NestJS con las migraciones
      correspondientes a este modelo, corriendo contra un schema `logistics_{workspace_uuid}`
      de prueba.
- [ ] Endpoint o listener que consume `quotation.accepted` y crea el `Shipment` en `draft`,
      verificado con un evento de prueba.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado, no solo el modelo.

## Autorevisión (agente, no reemplaza revisión humana)

Antes de continuar a Fase 6-8, el mismo agente que escribió este modelo lo revisó
críticamente y corrigió cuatro problemas reales encontrados (detalle inline en cada sección):

1. `Milestone.milestone_type` era un enum cerrado con un valor (`customs_cleared`) que
   pertenece al vocabulario de Customs, no de Shipment — violaba el límite de bounded
   context que esta misma fase pide respetar. Corregido a texto abierto con un núcleo
   mode-agnóstico sugerido.
2. `Party` estaba mal clasificado como Value Object teniendo identidad y cardinalidad propia
   — corregido a Entidad, y se le agregó el PK que le faltaba.
3. `Booking` no distinguía cuál registro es el vigente cuando hay rebooking — se agregó
   `is_current`.
4. El flujo de creación manual/spot (sin `Quotation` previa) estaba mencionado como posible
   (`quotation_id` nullable) pero no descrito como camino de integración — se agregó.

Esto **no sustituye** la revisión humana que esta fase sigue pidiendo (ver criterios de
aceptación) — es la revisión que un agente puede hacer solo, no la que requiere criterio de
negocio de Carlos (ej. si `notify_party` debería poder repetirse, o si hace falta un quinto
rol de `Party` que el agente no tiene forma de saber sin preguntarle).

## Notas para el agente

- **No es un checkpoint humano obligatorio en el sentido estricto de `CLAUDE.md`** (no está
  en la lista de duties/customs/accounting/migraciones de producción), pero
  `docs/00-master-index.md` marca esta fase como "con revisión de modelo de datos" y es el
  *core domain* del que dependen seis fases más — la recomendación fuerte es que Carlos
  revise el modelo de esta sección antes de que una sesión de agente futura escriba las
  migraciones reales contra una base de datos persistente. Si esa revisión no ha ocurrido,
  una sesión que retome esta fase debe generar las migraciones contra una base de desarrollo
  descartable primero, no contra nada que vaya a persistir datos reales.
- `services/shipment-service` no existe todavía como carpeta/proyecto — a diferencia de
  Fases 2-4, aquí no hay ni scaffold. Antes de escribir código hace falta decidir (no
  asumido en esta fase, para no repetir el error de sobre-especificar infraestructura antes
  de tiempo — ver ADR-003): ORM (TypeORM ya está en el stack de referencia de `CLAUDE.md`),
  estructura de migraciones por schema de tenant, y cómo se levanta Postgres/Redis en local
  (Docker Compose, a definir junto con Fase 14 o antes si hace falta para probar esta fase).
- Bounded context involucrado: **Shipment** (shared kernel). Cualquier campo que aparezca
  "específico de modo" durante la implementación (ej. alguien intenta agregar
  `container_iso_type` directo en `CargoUnit`) es señal de que pertenece a la tabla de
  extensión de Ocean (Fase 6), no a esta — moverlo, no forzarlo aquí.
- El evento `quotation.accepted` (Fase 4) y los eventos salientes de esta fase
  (`shipment.created`, `shipment.milestone.recorded`) comparten el mismo envelope
  (`docs/phases/01-architecture.md` sección 4) — cualquier cambio a ese formato debe
  reflejarse en ambas fases, no solo en una.
