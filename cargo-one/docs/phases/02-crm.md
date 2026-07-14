# Fase 2 — CRM

**Estado:** 🟡 En curso — work order especificado, `sales-extensions-app` escafoldado en
`core/twenty-apps/sales-extensions-app/`. Implementación real pendiente: falta `yarn install`
(bloqueado en este sandbox por red) y una instancia de Twenty corriendo vía Docker (daemon no
disponible en este sandbox). Ver `docs/decisions/003-twenty-apps-scaffolding.md` y
`core/twenty-apps/README.md`.

## Contexto

Depende de: Fase 1 (cerrada). Bounded context: **CRM & Sales**, fila 1 de la tabla de
contextos en `docs/phases/01-architecture.md` sección 3 — vive dentro de Twenty (objetos
nativos Company/Person/Opportunity + objetos/campos custom), no en un servicio satélite propio.

Esta fase cubre **solo la porción CRM** de ese contexto: el vocabulario compartido de roles y
datos de referencia que Fase 3 (Customers) y Fase 4 (Sales) van a consumir. No define todavía
el perfil operativo del cliente (eso es Fase 3: `CustomerLogisticsProfile`, INCOTERMS, términos
de crédito) ni la cotización/pipeline de ventas (eso es Fase 4: `Quotation`).

**Por qué existe esta fase separada de Fase 3/4:** un freight forwarder necesita distinguir,
desde el CRM base, qué rol de negocio cumple cada Company/Person (¿es el shipper, el consignee,
un agente/partner, un carrier, un proveedor?) y en qué trade lanes opera, antes de poder
construir perfiles de cliente o cotizaciones sobre esos datos. Si esto se mezclara con Fase 3
o 4, esas fases tendrían que redefinir el mismo vocabulario — señal de bounded context
duplicado que `CLAUDE.md` pide evitar.

**Aclaración de producto agregada en esta ronda (ver `docs/decisions/005-multi-tenant-product-clarification.md`):**
Cargo One es multi-tenant desde el día uno — cualquier freight forwarder que contrate el
servicio crea su propio workspace en Twenty y necesita configurar los datos de **su propia
empresa** (no los de sus clientes, eso ya lo cubre esta fase con `company_role`/`Trade Lane`
y Fase 3 con `CustomerLogisticsProfile`). Ese pedazo faltaba — se agrega abajo como
`OrganizationProfile`.

## Alcance de esta fase

**Incluido:**

| Elemento | Tipo | Dónde vive | Descripción |
|---|---|---|---|
| `company_role` | Campo custom (multi-select) en `Company` | Twenty, vía `sales-extensions-app` (Apps framework) | Valores: `shipper`, `consignee`, `carrier`, `customs_broker`, `partner_agent`, `vendor`. Una Company puede tener más de un rol (ej. un cliente que también es agente en otro país) |
| `tax_id` | Campo custom (texto) en `Company` | ídem | Identificador fiscal del país de operación (formato libre en esta fase — validación por país queda fuera, no se inventa regla de compliance sin documentarla) |
| `industry_vertical` | Campo custom (select) en `Company` | ídem | Clasificación comercial simple (ej. `retail`, `manufacturing`, `automotive`) para reporting, no para lógica de negocio |
| `contact_role` | Campo custom (multi-select) en `Person` | ídem | Valores: `primary`, `operations`, `billing`, `customs` — de qué trata cada contacto dentro de una Company |
| `Trade Lane` | Objeto custom nuevo (dato de referencia) | ídem | Campos: `origin_country`, `origin_port_or_city`, `destination_country`, `destination_port_or_city`. Relación many-to-many con `Company` (`primary_trade_lanes`) |
| `OrganizationProfile` | Objeto custom nuevo, **singleton por workspace** (una sola fila por tenant) | ídem | Configuración de la propia empresa del tenant — ver tabla de campos abajo |

### `OrganizationProfile` — la empresa del tenant, no la de sus clientes

| Campo | Tipo | Notas |
|---|---|---|
| `legal_name` | texto | Razón social del tenant (ej. "Sealion Cargo Inc.") |
| `trade_name` | texto, nullable | Nombre comercial si difiere del legal |
| `tax_id` / `business_number` | texto | Identificador fiscal/de negocio del tenant — mismo criterio que `Company.tax_id`: sin validación por país en esta fase |
| `default_currency` | texto (ISO 4217) | Moneda por defecto para `Quotation`/`Invoice` de este tenant — evita que cada cotización tenga que preguntarlo |
| `default_incoterms` | texto (mismo catálogo de Fase 3) | Default de la empresa, distinto del `preferred_incoterms` por cliente de `CustomerLogisticsProfile` (Fase 3) — ese es por cliente del tenant, este es el default general del tenant cuando no hay uno más específico |
| `primary_operating_countries` | texto[] (ISO 3166-1 alpha-2) | En qué países opera este tenant — informativo, no dispara reglas de compliance por sí solo |
| `logo_reference` | texto, nullable | Referencia al logo del tenant para documentos/cotizaciones/facturas generados por la plataforma (branding propio, no el de Cargo One) |
| `customs_filing_mode` | enum | `own_licensed_broker`, `partner_broker`, `not_configured` — **reemplaza** el enfoque anterior de tratar esto como una pregunta única para todo el sistema (ver corrección en Fase 11): cada tenant configura lo suyo aquí |
| `customs_broker_of_record_company_id` | UUID, nullable | Si `customs_filing_mode != not_configured`, referencia a la `Company` (rol `customs_broker`, puede ser una Company del propio tenant si tiene broker in-house) que presenta CADs/entries en su nombre |

**Por qué singleton y no una fila más de `Company`:** el tenant no es "un cliente más" del
CRM — es el dueño del workspace. Modelarlo como una fila de `Company` con un rol especial
generaría ambigüedad (¿puede el sistema mostrarle al tenant su propia empresa en la lista de
clientes? ¿puede alguien accidentalmente asignarle `company_role = shipper`?). Un objeto
singleton separado evita esa confusión desde el modelo de datos, no solo por convención.

**Onboarding de un tenant nuevo:** la creación del workspace en sí (signup, invitar
miembros) es 100% nativa de Twenty — no se reconstruye. Lo único nuevo de este proyecto es
que, al crear el workspace, la UI debe guiar al tenant a completar su `OrganizationProfile`
antes de dejarlo operar CRM/Sales/Shipment con normalidad — el flujo de ese guiado (wizard,
checklist, o simplemente un campo obligatorio bloqueante) es detalle de implementación, no se
decide en esta fase.

Todo lo anterior se define como código versionado dentro de `sales-extensions-app`
(`core/twenty-apps/sales-extensions-app/`, ya escafoldado con `create-twenty-app` — ver
ADR-003) usando el Apps framework — no directamente por UI vía el metadata engine — para que
la definición sea reproducible entre entornos (dev/staging/prod), consistente con que el
proyecto lo mantiene un developer único vía agentes que no comparten memoria entre sesiones.

**Explícitamente fuera de esta fase:**
- `CustomerLogisticsProfile` (INCOTERMS preferido, términos de crédito, compliance) → Fase 3.
- `Quotation` y cualquier lógica de pipeline de ventas → Fase 4.
- Cualquier validación real de `tax_id` por país o regla de screening de partes denegadas →
  no se inventa aquí (regla 4 de `CLAUDE.md`); si se necesita, se pregunta al humano y se
  documenta como ADR antes de implementarla.
- Cualquier servicio satélite — esta fase vive 100% dentro de Twenty.

## Criterios de aceptación

- [x] `sales-extensions-app` creado vía `npx create-twenty-app` (scaffold base).
- [ ] `yarn install` completado y `yarn twenty dev`/`yarn twenty remote:add` conectado a una
      instancia real de Twenty — **bloqueante para el resto de esta lista**; ver
      `core/twenty-apps/README.md` para el estado exacto del bloqueo y los comandos.
- [ ] Campos/objeto de la tabla de alcance definidos como código dentro de
      `sales-extensions-app` y sincronizados contra esa instancia.
- [ ] Nombres de API en `snake_case` inglés, labels en Title Case inglés (convención de
      `CLAUDE.md`).
- [ ] `Trade Lane` expuesto correctamente en GraphQL/REST (verificado con una consulta de
      prueba contra el workspace de desarrollo).
- [ ] `OrganizationProfile` verificado como singleton real (no permite dos filas en el mismo
      workspace) — regla de aplicación, documentar cómo se hizo cumplir al implementar.
- [ ] No se duplicó ningún campo que ya exista nativo en Twenty (Company/Person) — confirmado
      contra el esquema real antes de crear campos nuevos.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando lo anterior esté
      verificado contra una instancia real, no solo especificado.

## Notas para el agente

- **No hay checkpoint humano obligatorio** para esta fase (ver `docs/00-master-index.md`).
- **Bloqueante real (actualizado):** ya no es "clonar Twenty" — se determinó (ADR-003) que
  eso ni siquiera es necesario. El bloqueante real es (1) `yarn install` en
  `sales-extensions-app`, que falló en este sandbox porque corepack no puede descargar el
  binario de `yarn@4.13.0` a través del proxy saliente del entorno, y (2) no hay Docker
  daemon disponible en este sandbox para levantar un Twenty local (`yarn twenty dev`) o
  conectar a uno self-hosted. Una sesión futura con red completa y Docker debe correr
  `yarn install` y luego `yarn twenty dev` / `yarn twenty remote:add` desde
  `core/twenty-apps/sales-extensions-app/` antes de poder ejecutar el resto de los criterios
  de aceptación. Hasta entonces, esta fase queda en 🟡 — no marcar 🟢 sin instancia real.
- No se toca `packages/twenty-server` ni `packages/twenty-front` bajo ninguna circunstancia
  (regla 1 de `CLAUDE.md`) — todo lo de esta fase pasa por `sales-extensions-app`.
- Bounded context involucrado: **CRM & Sales** (porción CRM). Antes de empezar, confirmar
  contra `docs/phases/01-architecture.md` sección 3 que ningún campo propuesto pertenece en
  realidad a Fase 3 (perfil de cliente) o Fase 4 (ventas) — si hay duda, preguntar al humano
  antes de crear el campo, no asumir.
- `OrganizationProfile` es la corrección de un gap de producto real, no una anticipación
  especulativa (ver ADR-005): sin esto, cada fase que necesitara "el default de moneda/
  INCOTERMS/broker del tenant" hubiera terminado hardcodeando el caso de Sealion Cargo. No
  confundirlo con `CustomerLogisticsProfile` (Fase 3) — ese es sobre los clientes del tenant,
  este es sobre el tenant mismo.
