# Fase 2 — CRM

**Estado:** 🟡 En curso — **bloqueante de infraestructura resuelto**: `sales-extensions-app`
tiene `yarn install` completo y está conectado (`remote:add`, auth por API key válida) a una
instancia real de Twenty self-hosted (VPS de desarrollo, Fase 14). Falta implementar los
campos/objetos de la tabla de alcance como código y sincronizarlos — eso es lo único que
queda para cerrar esta fase. Ver `docs/decisions/003-twenty-apps-scaffolding.md`,
`docs/phases/14-deployment.md`, y `core/twenty-apps/README.md`.

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
| `company_role` (API real: `companyRole`, ✅ implementado) | Campo custom (multi-select) en `Company` | Twenty, vía `sales-extensions-app` (Apps framework) | Valores reales (UPPER_CASE, validado contra el servidor): `SHIPPER`, `CONSIGNEE`, `CARRIER`, `CUSTOMS_BROKER`, `PARTNER_AGENT`, `VENDOR`. Una Company puede tener más de un rol (ej. un cliente que también es agente en otro país) |
| `tax_id` (API real: `taxId`) | Campo custom (texto) en `Company` | ídem | Identificador fiscal del país de operación (formato libre en esta fase — validación por país queda fuera, no se inventa regla de compliance sin documentarla) |
| `industry_vertical` (API real: `industryVertical`) | Campo custom (select) en `Company` | ídem | Valores UPPER_CASE: `RETAIL`, `MANUFACTURING`, `AUTOMOTIVE` — clasificación comercial simple para reporting, no para lógica de negocio |
| `contact_role` (API real: `contactRole`) | Campo custom (multi-select) en `Person` | ídem | Valores UPPER_CASE: `PRIMARY`, `OPERATIONS`, `BILLING`, `CUSTOMS` — de qué trata cada contacto dentro de una Company |
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
| `customs_filing_mode` (API real: `customsFilingMode`) | enum | Valores UPPER_CASE: `OWN_LICENSED_BROKER`, `PARTNER_BROKER`, `NOT_CONFIGURED` — **reemplaza** el enfoque anterior de tratar esto como una pregunta única para todo el sistema (ver corrección en Fase 11): cada tenant configura lo suyo aquí |
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
- [x] `yarn install` completado y `yarn twenty remote:add` conectado a una instancia real de
      Twenty self-hosted (`remote:status` → `Auth: api-key (valid)`) — verificado en el VPS de
      desarrollo (Fase 14), no en el sandbox de agente donde se especificó originalmente.
- [ ] Campos/objeto de la tabla de alcance definidos como código dentro de
      `sales-extensions-app` (usar `yarn twenty dev:add <entityType>` para escafoldar cada
      objeto/campo, luego `yarn twenty plan`/`apply` para sincronizar contra el remote) —
      **este es el único criterio pendiente para cerrar la fase.**
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
- **Bloqueante de infraestructura: resuelto.** El sandbox de agente nunca pudo completar
  `yarn install` (corepack bloqueado por el proxy de red) ni tenía Docker disponible — se
  resolvió aprovisionando un VPS real (Hostinger, ver Fase 14) donde sí hay red completa y
  Docker corriendo. `sales-extensions-app` ya corrió `yarn install` ahí y está conectado
  (`remote:add --as production`) a la instancia real self-hosted con una API key válida.
  Sesiones futuras que trabajen esta fase deben operar sobre ese VPS (vía SSH o el terminal
  del panel de Hostinger), no asumir que hace falta repetir el diagnóstico de infraestructura.
- **Comandos reales del CLI `twenty`** (confirmados con `yarn twenty --help` contra la
  versión instalada, no asumidos de memoria): `dev:add <entityType>` para escafoldar un
  objeto/campo nuevo (tipos válidos: `object|field|logicFunction|frontComponent|role|skill|
  agent|connectionProvider|view|viewField|navigationMenuItem|pageLayout|pageLayoutTab|
  commandMenuItem`), `plan` para previsualizar el cambio de metadata antes de aplicarlo, y
  `apply` para aplicarlo contra el remote activo. `remote:add` NO acepta un flag
  `--authentication-method` (error real encontrado en esta sesión) — solo `--as`, `--url`,
  `--api-key`, `--local`.
- **Receta real para crear un campo `SELECT`/`MULTI_SELECT` (probada de punta a punta con
  `company_role`, no teórica):**
  1. `yarn twenty dev:add field` (wizard interactivo) — pero el wizard **no pide las
     opciones** y deja `objectUniversalIdentifier: 'fill-later'` como placeholder sin
     resolver. No confiar en que el wizard termina el trabajo solo.
  2. Editar el archivo generado en `src/fields/<nombre>.ts` a mano:
     - Import: `import { defineField, FieldType, STANDARD_OBJECT_UNIVERSAL_IDENTIFIERS } from 'twenty-sdk/define';`
     - `objectUniversalIdentifier: STANDARD_OBJECT_UNIVERSAL_IDENTIFIERS.company.universalIdentifier`
       (u otro objeto estándar — no hardcodear el UUID a mano, usar esta constante).
     - `name` en camelCase alfanumérico puro (`companyRole`, no `company-role` ni
       `company_role`) — el servidor rechaza guiones/underscores acá.
     - `options: [{ value: 'SHIPPER', label: 'Shipper', position: 0, color: 'blue' }, ...]`
       — cada opción es un `FieldMetadataComplexOption`: `value` (UPPER_CASE snake_case),
       `label`, `position` (entero), `color` (`TagColor`: `red|ruby|crimson|tomato|orange|
       amber|yellow|lime|grass|green|jade|mint|turquoise|cyan|sky|blue|iris|violet|purple|
       plum|pink|bronze|gold|brown|gray`), `id` opcional.
  3. `yarn twenty dev:typecheck` para validar TypeScript.
  4. `yarn twenty plan` para validar contra el servidor real **antes** de aplicar — acá
     aparecen los errores de convención (`name` inválido, `value` en minúscula) que el
     typecheck no detecta porque son reglas del servidor, no del tipo TypeScript.
  5. `yarn twenty apply` recién cuando `plan` da 0 errores.
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
