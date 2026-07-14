# Fase 11 — Documents, Customs e Integrations Hub

**Estado:** 🟡 En curso — work order especificado, incluyendo el **diseño** de los conectores
de aduana fundamentado en fuentes oficiales (ver sección de investigación abajo; decisión de
Carlos vía `docs/decisions/004-human-checkpoint-waiver.md`). Lo que sigue gateado, sin
excepción, no es el diseño sino **transmitir datos reales**: eso requiere un customs broker
licenciado con delegación de autoridad (requisito legal externo, no un checkpoint de este
proyecto) y el registro externo ante CBSA/CBP (ver `docs/00-master-index.md`, "Dependencias
externas").

## Contexto

Depende de: Fase 1 (cerrada), Fase 5 (Customs referencia `Shipment`). **Tres** bounded
contexts en una sola fase (originalmente eran dos — ver "Gap resuelto" más abajo):

- **Documents**, fila 9 de `docs/phases/01-architecture.md` sección 3 — servicio satélite
  `services/document-pipeline-service`. Patrón *conformist*: Shipment/Customs/Accounting
  consumen su salida ya validada, no reinterpretan el documento crudo.
- **Customs**, fila 7 de `docs/phases/01-architecture.md` sección 3 — servicio satélite
  propio `services/customs-service` (ya anticipado en `PROJECT_STRUCTURE.md`, nunca
  modelado hasta ahora). Downstream de Shipment, consumidor de `Document` (vía
  `ComplianceDocument`) y usuario del conector `carm-cbsa`/`ace-cbp` de Integrations Hub para
  la transmisión real.
- **Integrations Hub**, fila 10 — vive en `integrations/*` (conectores) + la app
  `integrations-hub-app` de Twenty, **ya escafoldada** en
  `core/twenty-apps/integrations-hub-app/` (ver `docs/decisions/003-twenty-apps-scaffolding.md`).
  Anti-Corruption Layer genérico: nadie más en el proyecto habla directo con un sistema
  externo.

### Gap resuelto: por qué Customs no tenía modelo hasta esta actualización

`docs/phases/01-architecture.md` identificó a Customs como bounded context propio desde
Fase 1, pero `docs/00-master-index.md` nunca le dio una fase numerada dedicada (a diferencia
de Accounting, que sí tiene su Fase 10) — quedó mencionado solo de forma implícita dentro del
nombre de esta fase. Se resuelve **sin agregar un número de fase nuevo** (evita renumerar
Fases 12-16 y todas sus referencias cruzadas ya escritas): Customs se modela formalmente
dentro de esta misma Fase 11, junto a Documents e Integrations Hub, con quienes de hecho ya
comparte dependencias directas (`Document` para la evidencia documental, el conector
`carm-cbsa`/`ace-cbp` para la transmisión). El título de la fase se actualizó para reflejarlo.

## Modelo de datos — Documents (`services/document-pipeline-service`)

### Agregado raíz: `Document`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `related_shipment_id` | UUID, nullable | Referencia no forzada a `Shipment` (Fase 5) |
| `file_reference` | texto | Ruta/URL en el storage configurado — el storage en sí (S3, local) es una decisión de Fase 14, no de aquí |
| `status` | enum | `uploaded`, `processing`, `classified`, `extracted`, `pending_approval`, `approved`, `rejected` |
| `document_type` | texto, nullable | Se completa por `ClassificationResult`, nunca se asume al subir el archivo |
| `uploaded_at` | timestamp | |

### Entidad: `OcrResult` (relación 1:1 con `Document`)

| Campo | Tipo | Notas |
|---|---|---|
| `document_id` | UUID, único | FK a `Document` |
| `raw_text` | texto | |
| `confidence_score` | decimal, nullable | |
| `engine` | texto | Qué proveedor de OCR se usó — no se elige un proveedor específico en esta fase |

### Entidad: `ClassificationResult` (relación 1:1 con `Document`)

| Campo | Tipo | Notas |
|---|---|---|
| `document_id` | UUID, único | FK a `Document` |
| `predicted_type` | texto | Ej. `commercial_invoice`, `packing_list`, `bill_of_lading` — catálogo abierto, no cerrado: nuevos tipos de documento no deberían requerir migración |
| `confidence_score` | decimal, nullable | |
| `model_version` | texto | |

### Entidad: `ExtractionResult` (relación 1:1 con `Document`)

| Campo | Tipo | Notas |
|---|---|---|
| `document_id` | UUID, único | FK a `Document` |
| `extracted_fields` | JSON | Estructura depende de `document_type` — no se fuerza un schema único por fase |
| `reviewed_by` | UUID, nullable | Referencia a `Person`/usuario de Twenty que aprobó — el pipeline **siempre** pasa por `pending_approval` antes de que otro contexto (Customs, Accounting) use el dato extraído; no hay aprobación automática en esta fase |
| `approved_at` | timestamp, nullable | |

## Modelo de datos — Customs (`services/customs-service`)

Nota de nomenclatura: `docs/phases/01-architecture.md` había nombrado la entidad de cálculo
`DutyCalculation`. Se renombra aquí a `DutyAssessment` — ver justificación en la fila
correspondiente: no calculamos nada, solo guardamos lo que CARM/ACE ya calcularon.

### Agregado raíz: `CustomsDeclaration`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `related_shipment_id` | UUID | Referencia no forzada a `Shipment` (Fase 5) — a diferencia de otras referencias cruzadas de este proyecto, esta **no es nullable**: una declaración de aduana siempre es sobre un shipment concreto |
| `jurisdiction` | enum | `cbsa_carm`, `cbp_ace` — de qué lado de la frontera es esta declaración; un shipment que cruza ambas requiere dos `CustomsDeclaration`, no una con dos jurisdicciones mezcladas |
| `broker_of_record_company_id` | UUID, nullable | Referencia a `Company` de Twenty con rol `customs_broker` (Fase 2) — quién presenta esto ante CBSA/CBP en nombre de Sealion Cargo/el importador. **Nullable hasta que se resuelva la pregunta de negocio abierta** (ver sección de investigación) — sin esto poblado, esta declaración no puede pasar de `draft` |
| `cad_reference_number` | texto, nullable | Número de referencia oficial (CAD de CARM, o el entry number de ACE) — se completa solo después de una transmisión real, nunca antes |
| `status` | enum | `draft`, `pending_broker_submission`, `submitted`, `accepted`, `rejected` — estados de **nuestro tracking**, no el state machine interno de CBSA/CBP, que no controlamos ni replicamos |
| `submitted_at` / `decided_at` | timestamp, nullable | |

### Entidad: `CustomsDeclarationLine` (relación many-to-one a `CustomsDeclaration`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `customs_declaration_id` | UUID | FK a `CustomsDeclaration` |
| `source_cargo_unit_id` | UUID, nullable | Referencia no forzada a `CargoUnit` (Fase 5) |
| `hs_code` | texto | **Sin validación ni lookup automático en esta fase** — se captura tal como lo provee el broker/importador. Clasificar mal esto es exactamente el riesgo que sigue gateado por `CLAUDE.md` regla 3 |
| `description` | texto | Descripción de la mercancía para esta línea |
| `country_of_origin` | texto (ISO 3166-1 alpha-2) | |
| `quantity` / `unit_of_measure` | — | |
| `value_for_duty` / `value_for_duty_currency` | decimal / ISO 4217 | Valor declarado, no un cálculo — lo provee quien clasifica, no se infiere |

### Entidad: `DutyAssessment` (relación 1:1 con `CustomsDeclaration`)

| Campo | Tipo | Notas |
|---|---|---|
| `customs_declaration_id` | UUID, único | FK a `CustomsDeclaration` |
| `total_duties_amount` / `total_taxes_amount` | decimal | **Eco de la respuesta oficial de CARM/ACE, no un cálculo propio** — CARM calcula esto automáticamente a partir del CAD (ver investigación abajo); esta tabla solo guarda lo que la respuesta oficial trajo, para que Cargo One pueda mostrárselo al cliente sin volver a consultar la fuente cada vez |
| `currency` | ISO 4217 | |
| `assessed_by` | texto | `cbsa_carm` o `cbp_ace` — de dónde vino este número, siempre explícito para que nadie confunda esto con un cálculo interno |
| `received_at` | timestamp | |

### Entidad: `ComplianceDocument` (relación many-to-one a `CustomsDeclaration`, many-to-one a `Document`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `customs_declaration_id` | UUID | FK a `CustomsDeclaration` |
| `document_id` | UUID | FK a `Document` (definido arriba, en Documents) — **no duplica el archivo ni su contenido**, solo asocia un documento ya existente a esta declaración |
| `document_role` | enum | `commercial_invoice`, `certificate_of_origin`, `packing_list`, `bill_of_lading`, `other` |
| `required` | booleano | Si este tipo de documento es obligatorio para esta declaración — no hay lógica automática que determine esto por tipo de mercancía/jurisdicción en esta fase, se marca a mano |

## Modelo de datos — Integrations Hub (`integrations/*`)

### Agregado raíz: `Connector`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `provider` | enum | `amazon_sp_api`, `wayfair_api`, `carm_cbsa`, `ace_cbp` |
| `capabilities` | texto[] | Ej. `order_sync`, `document_submission` — qué puede hacer este conector, informativo |
| `status` | enum | `active`, `inactive`, `pending_registration` (este último para `carm_cbsa`/`ace_cbp` mientras el registro externo no esté listo) |

### Entidad: `ConnectorCredential` (relación 1:1 con `Connector`)

| Campo | Tipo | Notas |
|---|---|---|
| `connector_id` | UUID, único | FK a `Connector` |
| `credential_reference` | texto | **Referencia** a un secreto en un gestor de secretos (ej. path en Vault/AWS Secrets Manager) — este modelo **nunca** almacena el secreto en texto plano en esta tabla. Qué gestor de secretos usar es una decisión de Fase 13 (Security), no de aquí — marcado explícitamente como pendiente |

### Entidad: `IntegrationEvent` (relación many-to-one a `Connector`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `connector_id` | UUID | FK a `Connector` |
| `event_type` | texto | |
| `payload` | JSON | |
| `occurred_at` | timestamp | |

## Investigación: CBSA/CARM y CBP/ACE (fuentes oficiales)

Hecha en esta sesión para fundamentar el diseño de los conectores sin inventar reglas
(`CLAUDE.md` regla 4, vía ADR-004: investigar antes que preguntar). Resumen — ver fuentes al
final de esta sección.

**CBSA/CARM:**
- CARM es, desde el 21 de octubre de 2024, el sistema oficial de registro para duties/taxes
  de importaciones comerciales a Canadá.
- El **CAD (Commercial Accounting Declaration)** reemplaza los formularios previos B3
  (Customs Coding) y B2 (Request for Adjustment). Se presenta vía EDI o API dentro de los 5
  días hábiles posteriores al release de la carga.
- **CARM calcula duties/taxes automáticamente** a partir de los datos declarados en el CAD —
  Cargo One no necesita (ni debe) implementar una fórmula de impuestos propia; el riesgo real
  está en la exactitud de la clasificación arancelaria (HS code) y valuación que se declara,
  no en un cálculo que hagamos nosotros.
- **Solo un customs broker licenciado, con delegación de autoridad, puede presentar un CAD o
  una corrección en nombre de un importador.** Esto es un requisito legal externo que
  determina la arquitectura real del conector: `integrations/carm-cbsa` probablemente no es
  "Cargo One habla directo con CBSA", sino "Cargo One envía datos estructurados a un broker
  licenciado (propio de Sealion Cargo o un partner) que los presenta". **Pregunta de negocio
  abierta, no técnica:** ¿Sealion Cargo opera con un broker in-house, un partner externo, o
  planea licenciarse? Esto no lo puede decidir un agente — es la única parte de esta
  investigación que sigue necesitando una respuesta de Carlos, porque es un dato de la
  realidad del negocio, no algo que esté en una página oficial.
- Registro: Trade Chain Partner vía CARM Client Portal, con seguridad financiera (bond RPP —
  Release Prior to Payment). A partir del 1 de enero de 2026, el business number de un broker
  ya no puede usarse para liberar/contabilizar carga en nombre de un importador — cada
  importador necesita su propio registro.

**CBP/ACE:**
- CATAIR (CBP and Trade Automated Interface Requirements) es la especificación técnica para
  transmitir datos a ACE vía ABI (Automated Broker Interface).
- Participantes elegibles: customs brokers, importadores (auto-declarando su propia carga), o
  ABI service bureaus.
- Requiere un **ISA (Interconnection Security Agreement)** firmado con CBP para transmisión
  directa/SFTP, y certificación técnica del sistema participante (testing conforme a la
  Publication 552/CATAIR) antes de ir en vivo.

**Fuentes:**
- [CARM: Assess and pay duties and taxes on imported commercial goods](https://www.cbsa-asfc.gc.ca/services/carm-gcra/menu-eng.html) (cbsa-asfc.gc.ca)
- [Get started with CARM](https://www.cbsa-asfc.gc.ca/services/carm-gcra/start-passer-eng.html) (cbsa-asfc.gc.ca)
- [Commercial Accounting Declaration (CAD) — GHY International](https://www.ghy.com/carm/commercial-accounting-declaration-cad/)
- [ACE Automated Broker Interface (ABI) / CATAIR — CBP](https://www.cbp.gov/trade/automated/catair)
- [How to Use ACE — CBP](https://www.cbp.gov/trade/automated/how-to-use-ace)
- [19 CFR Part 143 Subpart A — Automated Broker Interface](https://www.ecfr.gov/current/title-19/chapter-I/part-143/subpart-A)

**Nota de honestidad sobre el alcance de esta investigación:** esto es una pasada de nivel
"entender el proceso y las obligaciones", con fuentes citadas y verificables — no es lectura
completa de las especificaciones técnicas EDI/API línea por línea (los layouts de mensaje
reales de CATAIR son documentos de cientos de páginas). Antes de implementar el conector real,
la sesión que lo haga debe leer la especificación técnica completa vigente en ese momento
(las reglas cambian — ver el aviso de enero 2026 arriba, que ya cambió mientras se investigaba
esto), no asumir que este resumen sigue vigente sin revalidar.

## Explícitamente fuera de esta fase

- **Transmisión real de datos a `carm-cbsa`/`ace-cbp`** — el diseño del conector ya está
  fundamentado (arriba), pero ninguna llamada real a esos sistemas se hace sin: (1) el
  registro externo (Trade Chain Partner/CARM, ISA/CATAIR para ACE) completado, y (2) la
  pregunta de negocio abierta (broker propio vs. partner vs. licenciarse) resuelta por Carlos
  — esto no es un checkpoint del proyecto, es que la arquitectura del conector literalmente
  depende de esa respuesta.
- Clasificación arancelaria (HS code) o valuación aduanera de un embarque real — sigue
  gateado por `CLAUDE.md` regla 3 (ver la corrección hecha en esta sesión), un error ahí es
  una sanción real. `CustomsDeclarationLine.hs_code`/`value_for_duty` son campos de captura,
  no de inferencia — nada en este modelo clasifica o valúa automáticamente.
- Validación automática de qué `ComplianceDocument` son obligatorios por tipo de mercancía o
  jurisdicción — se marca a mano en esta fase, no se infiere una regla general.
- Elección de proveedor de OCR/clasificación de documentos — decisión técnica que se toma al
  implementar, no se fuerza en esta fase.
- El código de los servicios/conectores (incluye `services/customs-service`, que ahora tiene
  modelo pero sigue sin código, igual que `document-pipeline-service`).

## Criterios de aceptación

- [x] Modelo de datos de `Document`/`OcrResult`/`ClassificationResult`/`ExtractionResult`
      definido con aprobación humana obligatoria antes de que el dato extraído se use en
      otro contexto.
- [x] Modelo de `Connector`/`ConnectorCredential`/`IntegrationEvent` definido, con los
      secretos explícitamente fuera del modelo (solo referencia — mecanismo decidido en
      Fase 13: variables de entorno, no un gestor de secretos dedicado todavía).
- [x] Diseño de los conectores `carm-cbsa`/`ace-cbp` fundamentado en fuentes oficiales (ver
      sección de investigación).
- [x] Modelo de datos de `CustomsDeclaration`/`CustomsDeclarationLine`/`DutyAssessment`/
      `ComplianceDocument` definido, cerrando el gap de bounded context detectado — Customs ya
      no vive solo implícito en el nombre de la fase.
- [ ] **Pregunta de negocio abierta:** ¿Sealion Cargo opera con broker propio, partner, o
      planea licenciarse? — respuesta de Carlos, necesaria para poblar
      `CustomsDeclaration.broker_of_record_company_id` y terminar de definir la arquitectura
      exacta de `carm-cbsa`/`ace-cbp` (a quién le habla el conector realmente).
- [ ] `services/document-pipeline-service` implementado y verificado.
- [ ] `services/customs-service` implementado con las migraciones correspondientes.
- [ ] Conectores `amazon-sp-api` y `wayfair-api` implementados siguiendo el patrón "thin
      connector" (`integrations/shared/connector-interface.ts` en `PROJECT_STRUCTURE.md`).
- [ ] Conectores `carm-cbsa`/`ace-cbp`: código real solo tras (a) registro externo confirmado
      y (b) la pregunta de negocio de arriba resuelta — no requiere ya una revisión humana
      adicional del diseño en sí, eso ya se investigó y quedó documentado.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo para la porción Documents/Customs
      + conectores genéricos; los conectores de aduana reales pueden quedar 🟡/⛔
      indefinidamente sin que eso bloquee cerrar el resto de esta fase.

## Notas para el agente

- El diseño de los conectores de aduana y del modelo de Customs ya no requiere pausar a pedir
  revisión humana (ADR-004) — sí requiere, antes de escribir código real, resolver la
  pregunta de negocio abierta (broker propio/partner/licenciarse) y tener el registro externo
  confirmado. Esos dos son bloqueantes de hecho, no de política.
- Bounded contexts involucrados: **Documents** (conformist, alimenta a Customs/Accounting),
  **Customs** (downstream de Shipment y de Documents, usuario del conector de Integrations
  Hub) e **Integrations Hub** (ACL genérico). `ConnectorCredential.credential_reference`
  apunta a una variable de entorno (decidido en Fase 13), no a un gestor de secretos dedicado
  todavía.
- `integrations-hub-app` ya existe como scaffold (`core/twenty-apps/integrations-hub-app/`,
  ver ADR-003) pero, igual que `sales-extensions-app`/`quotation-app`, tiene pendiente
  `yarn install` + conexión a una instancia de Twenty antes de poder implementarse de verdad.
  `customs-service` y `document-pipeline-service` son servicios NestJS aparte (como
  `shipment-service`/`warehouse-service`) — no dependen de ese scaffold de Twenty, dependen de
  su propia infraestructura (Fase 14).
- Si una sesión futura retoma la implementación real del conector `carm-cbsa`/`ace-cbp`, debe
  revalidar la investigación de esta fase contra las fuentes oficiales vigentes en ese
  momento — las reglas cambian (ver el aviso de enero 2026 citado arriba, que cambió durante
  esta misma investigación).
- `DutyAssessment` se renombró desde `DutyCalculation` (nombre original de
  `docs/phases/01-architecture.md`) — si se busca "DutyCalculation" en el resto del repo y no
  aparece en este archivo, es por esto, no un error.
