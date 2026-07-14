# Fase 11 — Documents e Integrations Hub

**Estado:** 🟡 En curso — work order especificado. Checkpoint humano **parcial**: el pipeline
de documentos y los conectores genéricos (Amazon SP-API, Wayfair) no requieren revisión
obligatoria; los conectores de aduana (`carm-cbsa`, `ace-cbp`) **sí la requieren** en su
totalidad (`CLAUDE.md` regla 3) y además dependen de registro externo (ver
`docs/00-master-index.md`, sección "Dependencias externas").

## Contexto

Depende de: Fase 1 (cerrada). Dos bounded contexts distintos en una sola fase (así los
agrupa `docs/00-master-index.md`):

- **Documents**, fila 9 de `docs/phases/01-architecture.md` sección 3 — servicio satélite
  `services/document-pipeline-service`. Patrón *conformist*: Shipment/Customs/Accounting
  consumen su salida ya validada, no reinterpretan el documento crudo.
- **Integrations Hub**, fila 10 — vive en `integrations/*` (conectores) + la app
  `integrations-hub-app` de Twenty, **ya escafoldada** en
  `core/twenty-apps/integrations-hub-app/` (ver `docs/decisions/003-twenty-apps-scaffolding.md`).
  Anti-Corruption Layer genérico: nadie más en el proyecto habla directo con un sistema
  externo.

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

## Explícitamente fuera de esta fase

- **Conectores `carm-cbsa` y `ace-cbp` reales** — el `Connector` con `provider = carm_cbsa` o
  `ace_cbp` puede existir como registro (`status = pending_registration`), pero **ninguna
  lógica de integración real, ningún mapeo de campos, ninguna llamada a esos sistemas** se
  implementa sin: (1) el registro externo como Trade Chain Partner/proveedor EDI ante CBSA
  (o el equivalente ACE/CBP) completado — ver `docs/00-master-index.md` — y (2) revisión
  humana explícita, sin excepción (`CLAUDE.md` reglas 3 y 4). **REQUIERE REVISIÓN HUMANA.**
- Reglas de negocio de qué campos mapear hacia/desde CBSA/CARM o ACE/CBP — no se inventan
  (regla 4 de `CLAUDE.md`).
- Elección de proveedor de OCR/clasificación/gestor de secretos — decisiones técnicas que se
  toman al implementar, no se fuerzan en esta fase.
- Cualquier lógica de aduana o cálculo de duties sobre los datos extraídos — Customs
  (vive conceptualmente en esta misma fase 11 según el master index, pero su lógica de
  cálculo es checkpoint humano obligatorio y no se aborda en este documento).
- El código de los servicios/conectores.

## Criterios de aceptación

- [x] Modelo de datos de `Document`/`OcrResult`/`ClassificationResult`/`ExtractionResult`
      definido con aprobación humana obligatoria antes de que el dato extraído se use en
      otro contexto.
- [x] Modelo de `Connector`/`ConnectorCredential`/`IntegrationEvent` definido, con los
      secretos explícitamente fuera del modelo (solo referencia).
- [ ] `services/document-pipeline-service` implementado y verificado.
- [ ] Conectores `amazon-sp-api` y `wayfair-api` implementados siguiendo el patrón "thin
      connector" (`integrations/shared/connector-interface.ts` en `PROJECT_STRUCTURE.md`).
- [ ] Conectores `carm-cbsa`/`ace-cbp`: **no avanzar sin (a) registro externo confirmado y
      (b) revisión humana explícita** — ver `docs/00-master-index.md`.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo para la porción Documents +
      conectores genéricos; los conectores de aduana pueden quedar 🟡/⛔ indefinidamente sin
      que eso bloquee cerrar el resto de esta fase.

## Notas para el agente

- Esta fase mezcla dos niveles de riesgo distintos en un solo documento (igual que
  `docs/00-master-index.md` los agrupa en una fila): Documents y los conectores genéricos son
  trabajo normal de agente; los conectores de aduana son checkpoint humano obligatorio y
  dependencia externa combinados — no tratar toda la fase con el mismo nivel de autonomía.
- Bounded contexts involucrados: **Documents** (conformist, alimenta a Customs/Accounting) e
  **Integrations Hub** (ACL genérico). `ConnectorCredential.credential_reference` es
  intencionalmente un placeholder — no inventar un mecanismo de secretos aquí, eso es Fase 13.
- `integrations-hub-app` ya existe como scaffold (`core/twenty-apps/integrations-hub-app/`,
  ver ADR-003) pero, igual que `sales-extensions-app`/`quotation-app`, tiene pendiente
  `yarn install` + conexión a una instancia de Twenty antes de poder implementarse de verdad.
