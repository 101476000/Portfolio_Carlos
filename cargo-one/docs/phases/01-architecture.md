# Fase 1 — Arquitectura, Twenty Analysis, DDD, Multi-Tenant, Legal

**Estado:** 🟢 Cerrada (documentación). Bloquea/desbloquea el resto del proyecto — ver
`docs/00-master-index.md`.

## Contexto

Esta fase no produce código: produce las decisiones estructurales de las que depende todo lo
demás. Antes de escribir el primer objeto custom en Twenty o el primer servicio satélite hace
falta responder tres preguntas:

1. ¿Qué provee Twenty y por dónde se extiende sin tocar su código fuente?
2. ¿Cómo se modela multi-tenancy de forma consistente entre el núcleo (Twenty) y los
   servicios satélite (dominio logístico)?
3. ¿Cuáles son los bounded contexts del dominio, quién es dueño de qué entidad, y cómo se
   comunican entre sí y con Twenty?

Este documento resuelve las tres. También resuelve formalmente `docs/decisions/002-multi-tenant-strategy.md`.
`docs/decisions/001-twenty-license.md` (licencia AGPL/comercial) queda explícitamente **fuera**
de esta fase: es una decisión humana/legal, no arquitectónica — ver nota en "Notas para el agente".

## 1. Análisis de Twenty CRM

Twenty (github.com/twentyhq/twenty) es el núcleo. Lo que provee y cómo se usa sin modificar
`packages/twenty-server` ni `packages/twenty-front`:

| Capacidad de Twenty | Qué provee | Cómo lo consume Cargo One |
|---|---|---|
| Identidad y auth | Login, SSO, sesiones, API tokens por workspace | Servicios satélite usan API tokens de workspace para llamar a Twenty; nunca comparten su propia auth con Twenty |
| Permisos/roles | RBAC nativo a nivel de workspace y objeto | Se reutiliza tal cual para CRM/Sales; los servicios satélite implementan su propia autorización para operaciones que Twenty no modela (ej. aprobar una CAD) |
| Motor de objetos custom (metadata engine) | Permite crear objetos/campos custom por workspace sin migraciones manuales, expuestos automáticamente en GraphQL/REST | Fases 2-4 (CRM, Customers, Sales) y partes de Fase 11 (Documents) se construyen aquí — no requieren servicio propio |
| Apps framework (`npx create-twenty-app`) | Extensión soportada oficialmente: objetos custom, lógica server-side (serverless functions), UI embebida | Vía este framework viven `quotation-app`, `sales-extensions-app`, `integrations-hub-app` (ver `PROJECT_STRUCTURE.md`) |
| API GraphQL/REST dinámica | Se regenera automáticamente según los objetos custom del workspace | Contrato principal para que los servicios satélite lean/escriban datos de Twenty (Company, Person, Opportunity, y los objetos custom que definamos) |
| Webhooks salientes | Notifica eventos de Twenty (creación/edición de registros) | Fuente de eventos para que los servicios satélite reaccionen a cambios en CRM (ej. nueva Opportunity ganada dispara creación de Shipment) |
| Multi-tenancy nativo | Un workspace = un tenant, aislado por **schema propio en Postgres** (`workspace_{uuid}`) | Ver decisión de la sección 2 — se adopta el mismo patrón para los servicios satélite |

**Regla de extensión (ya en `CLAUDE.md`, reafirmada aquí):** toda funcionalidad nueva que quepa
dentro del modelo de objetos/permisos de Twenty se construye vía Apps framework u objetos custom.
Todo lo que requiera lógica de dominio pesada, bases de datos propias, colas, o integraciones
externas complejas (customs, accounting, warehouse, document pipeline, AI) vive en `services/`
como satélite independiente, hablando con Twenty solo por API/eventos.

## 2. Decisión de Multi-Tenancy (resuelve ADR-002)

**Estado de la decisión:** 🟢 Resuelto. Detalle completo en `docs/decisions/002-multi-tenant-strategy.md`.

### Opciones evaluadas

| Opción | Aislamiento | Costo operativo | Consistencia con Twenty | Riesgo principal |
|---|---|---|---|---|
| A — Row Level Security (RLS) | Lógico, por policy | Bajo (una sola base) | Ninguna — Twenty no usa este patrón | Fuga de datos entre tenants por policy mal configurada; grave en un dominio con datos de aduana/facturación |
| B — Schema per Tenant | Fuerte (namespace propio por tenant) | Medio (N schemas, migraciones por schema) | **Alta — es el mismo patrón que usa Twenty (`workspace_{uuid}`)** | Escalado a miles de tenants requiere tooling de migración por schema (no es el caso de Cargo One: B2B, decenas/cientos de forwarders, no millones de usuarios finales) |
| C — Database per Tenant | Máximo | Alto (N bases, N conexiones, N pipelines de backup) | Ninguna | Sobre-ingeniería para el volumen esperado; solo se justifica para un cliente enterprise que exija infraestructura dedicada |

### Decisión

**Opción B — Schema per Tenant**, para los servicios de dominio logístico (`services/*`),
usando como identificador de tenant el mismo `workspace_id` (UUID) que ya asigna Twenty a
cada workspace.

**Razonamiento:**
- Consistencia operativa: un solo modelo mental y un solo set de herramientas de
  backup/restore/migración para todo el sistema (núcleo y satélites), relevante porque el
  proyecto lo mantiene un developer único orquestando agentes.
- Aislamiento más fuerte que RLS para un dominio donde un error de fuga de datos entre
  tenants (ej. dos forwarders viendo documentos de aduana ajenos) no es un bug cualquiera,
  es una violación de compliance.
- Evita la sobre-ingeniería de "database per tenant" para un negocio B2B de volumen medio.
- Escape hatch disponible: si un tenant específico exige aislamiento físico total (cliente
  enterprise, requisito regulatorio de residencia de datos), se puede "promover" ese schema
  a una base de datos dedicada sin cambiar el modelo de datos ni el código de aplicación —
  es una extensión natural del patrón, no un cambio de arquitectura.

**Convención de nombres:** cada servicio satélite con base propia usa
`logistics_{workspace_uuid}` como nombre de schema (prefijo `logistics_` para no colisionar
nunca con los schemas `workspace_{uuid}` que Twenty gestiona en su propia base de datos —
son bases de datos Postgres completamente distintas, pero se mantiene la convención por
claridad si alguna vez comparten instancia de Postgres en un entorno de desarrollo).

**Migraciones:** cada servicio corre sus migraciones (TypeORM/Prisma, a definir por servicio
en su fase correspondiente) iterando sobre todos los schemas de tenant activos. Se documenta
el runbook de migración en Fase 14 (Deployment).

**Backup/restore:** por ser Postgres nativo (`pg_dump --schema=logistics_{uuid}`), el backup
por tenant es directo y no requiere infraestructura adicional — otra ventaja frente a la
Opción C sin pagar el costo operativo completo de bases separadas.

## 3. Domain-Driven Design — Bounded Contexts

### Clasificación estratégica

- **Core domain** (lo que diferencia a Cargo One, máxima inversión): **Shipment** y sus
  extensiones por modo (**Ocean, Air, Ground**) y **Warehouse**.
- **Supporting domain** (necesario, no diferenciador): **Sales/CRM extensions**, **Customs**,
  **Accounting**, **Documents**.
- **Generic domain** (podría resolverse con soluciones de terceros a futuro):
  **Integrations Hub**, **AI Platform**.

### Contextos, agregados y relaciones

| # | Bounded Context | Vive en | Agregado(s) raíz | Entidades/Value Objects clave | Relación con otros contextos |
|---|---|---|---|---|---|
| 1 | CRM & Sales | Twenty (Apps framework + objetos custom) | — (usa Company/Person/Opportunity nativos de Twenty) | Quotation, CustomerLogisticsProfile (VO: INCOTERMS preferido, términos de crédito) | Upstream de Shipment (una Opportunity ganada puede originar un Shipment) |
| 2 | Shipment (shared kernel) | `services/shipment-service` | `Shipment` | Booking, Container/Package, Milestone, Party (rol Shipper/Consignee/Notify — referencia por ID a Company/Person de Twenty vía ACL, no duplica el dato) | Núcleo compartido por Ocean/Air/Ground/Warehouse; downstream de CRM & Sales; upstream de Customs, Accounting, Documents |
| 3 | Ocean | `services/shipment-service` (extensión de modo) | — (extiende `Shipment`) | Vessel, BillOfLading, ContainerType (ISO) | Downstream de Shipment (shared kernel) |
| 4 | Air | `services/shipment-service` (extensión de modo) | — (extiende `Shipment`) | AirWaybill (AWB), Flight | Downstream de Shipment (shared kernel) |
| 5 | Ground | `services/shipment-service` (extensión de modo) | — (extiende `Shipment`) | Truck, BillOfLading (ground), Route | Downstream de Shipment (shared kernel) |
| 6 | Warehouse | `services/warehouse-service` | `WarehouseReceipt` | Inventory/StorageUnit, InboundOrder, OutboundOrder | Downstream y a la vez upstream de Shipment (la carga puede pasar por warehouse antes/después de un tramo de transporte) |
| 7 | Customs | `services/customs-service` | `CustomsDeclaration` (CAD) | `CustomsDeclarationLine`, `DutyAssessment` (renombrada desde `DutyCalculation` — no calculamos duties, CARM/ACE sí), `ComplianceDocument` — modelo completo en `docs/phases/11-documents.md` (fase compartida con Documents/Integrations Hub, ver esa fase) | Downstream de Shipment y de Documents; Anti-Corruption Layer hacia CBSA/CARM y ACE/CBP vía `integrations/carm-cbsa`, `integrations/ace-cbp`. Diseño fundamentado en fuentes oficiales (ADR-004); **transmisión real sigue gateada** por requisito legal externo (broker licenciado), no por checkpoint de este proyecto |
| 8 | Accounting | `services/accounting-service` | `Invoice` | Payment, LedgerEntry, Billing | Downstream de Sales (Quotation→Invoice) y de Shipment (eventos facturables). **Checkpoint humano obligatorio** |
| 9 | Documents | `services/document-pipeline-service` | `DocumentPipeline` | Document, OcrResult, ClassificationResult | Upstream/proveedor de datos estructurados para Shipment/Customs/Accounting (patrón *conformist*: esos contextos consumen la salida ya validada, no reinterpretan el documento crudo) |
| 10 | Integrations Hub | `integrations/*` + `integrations-hub-app` (Twenty) | `Connector` | ConnectorCredential (token admin), IntegrationEvent | Anti-Corruption Layer genérico para todo sistema externo (Amazon SP-API, Wayfair, CARM-CBSA, ACE-CBP) — nadie más habla directo con un sistema externo |
| 11 | AI Platform | `services/ai-gateway-service` | `AgentSession` | PromptTemplate, EmbeddingIndex | Consumidor transversal (vía API/MCP) de todos los demás contextos; no es dueño de datos de negocio, solo de sus propios artefactos de IA (prompts, embeddings, sesiones) |

### Mapa de contexto (relaciones)

```
CRM & Sales (Twenty) ──upstream──▶ Shipment (shared kernel) ──▶ Ocean
                                          │                  ├─▶ Air
                                          │                  └─▶ Ground
                                          │
                                          ├──▶ Warehouse (bidireccional)
                                          ├──▶ Customs ──ACL──▶ CARM/CBSA, ACE/CBP
                                          ├──▶ Accounting
                                          └──▶ Documents (conformist upstream de Customs/Accounting)

Integrations Hub ──ACL──▶ todo sistema externo (incluye CARM/CBSA, ACE/CBP, Amazon SP-API, Wayfair)
AI Platform ──consume vía API/MCP──▶ todos los contextos anteriores
```

**Regla derivada para fases futuras:** si una carpeta bajo `services/` empieza a contener
lógica de dos contextos de esta tabla, es señal de que hay que dividirla (ya está en
`PROJECT_STRUCTURE.md`, se reafirma aquí porque es una decisión de Fase 1).

## 4. Arquitectura de integración núcleo↔satélites

- **Nunca** acceso directo a la base de datos de Twenty desde un servicio satélite (regla ya
  en `CLAUDE.md`). Toda lectura/escritura de datos de Twenty pasa por su API GraphQL/REST con
  un token de API scoped al workspace.
- **Eventos:** Redis/BullMQ para comunicación asíncrona entre satélites, y para consumir
  webhooks salientes de Twenty. Envelope estándar de evento:
  ```
  {
    "workspace_id": "uuid",       // = tenant id, igual al que usa Twenty
    "event_type": "shipment.created",
    "payload": { ... },
    "occurred_at": "ISO-8601",
    "correlation_id": "uuid"      // para trazabilidad entre servicios
  }
  ```
- **Lectura de datos de Twenty desde satélites:** vía GraphQL, con cache local de solo-lectura
  cuando el patrón de acceso lo justifique (a definir por servicio en su fase), nunca como
  fuente de verdad — Twenty sigue siendo dueño de Company/Person/Opportunity.
- **Sistemas externos (customs, marketplaces):** solo a través de `integrations/*`
  (Integrations Hub), nunca un servicio de dominio llama directo a CBSA/CARM o ACE/CBP.

## Alcance de esta fase

**Incluido:** análisis de Twenty y sus puntos de extensión, decisión de multi-tenancy (ADR-002
resuelto), definición de bounded contexts y mapa de contexto, arquitectura de integración
núcleo↔satélites a nivel de patrón (eventos, API, ACL).

**Explícitamente fuera de esta fase:** cualquier código de servicio, cualquier objeto custom
en Twenty, cualquier diagrama de base de datos a nivel de tabla/columna (eso se hace por
bounded context en su fase correspondiente, ej. modelo de datos de Shipment en Fase 5), y la
resolución de la licencia comercial de Twenty (ADR-001, decisión humana/legal, no técnica).

## Criterios de aceptación

- [x] `docs/decisions/002-multi-tenant-strategy.md` actualizado a 🟢 Resuelto con la decisión
      y el razonamiento.
- [x] Análisis de Twenty documentado con sus puntos de extensión soportados (sección 1).
- [x] Bounded contexts identificados, cada uno con agregado raíz, entidades clave, y relación
      con los demás contextos (sección 3), clasificados en core/supporting/generic.
- [x] Arquitectura de integración núcleo↔satélites definida a nivel de patrón (sección 4).
- [x] `docs/00-master-index.md` actualizado reflejando el cierre de esta fase.

## Notas para el agente

- Esta fase no tiene checkpoint humano obligatorio según `docs/00-master-index.md`
  ("Agente puede trabajar solo: Sí"), pero es la base de la que depende todo lo demás — se
  recomienda al operador humano (Carlos) revisar esta decisión de multi-tenancy y el mapa de
  contexto antes de que una fase de dominio (5 en adelante) empiece a definir modelos de
  datos, tal como indica la regla de ADR-002.
- `docs/decisions/001-twenty-license.md` (licencia AGPL/comercial) **sigue pendiente** — no
  bloquea Fases 1-13/15/16 (desarrollo y documentación), pero sí bloquea Fase 14 en producción
  con clientes reales. No se tocó en esta fase porque es una decisión humana, no de agente.
- Próximo paso recomendado: Fases 2-4 (CRM/Customers/Sales) y Fase 5 (Shipment) pueden
  arrancar en paralelo, ya que ambas dependen solo de esta fase — ver `docs/00-master-index.md`
  y `README.md` (sección "Orden recomendado de trabajo").
- Cuando cada fase de dominio (5+) defina su modelo de datos concreto, debe hacerlo dentro de
  los límites del bounded context asignado en la sección 3 de este documento — si una entidad
  no encaja claramente en un contexto existente, es una señal para crear un ADR nuevo antes de
  seguir, no para forzarla en el contexto más cercano.
