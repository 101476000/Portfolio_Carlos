# Fase 2 — CRM

**Estado:** 🟢 Cerrada — los 6 elementos de la tabla de alcance (`company_role`, `tax_id`,
`industry_vertical`, `contact_role`, `Trade Lane`, `OrganizationProfile`) están implementados
y verificados contra la instancia real de Twenty self-hosted (VPS de desarrollo, Fase 14), no
solo especificados. Único punto que queda deliberadamente abierto: la enforcement real de
`OrganizationProfile` como singleton (ver "Notas para el agente" — no bloquea el cierre,
requiere un `logicFunction` que es trabajo de una sesión futura). Ver
`docs/decisions/003-twenty-apps-scaffolding.md`, `docs/phases/14-deployment.md`.

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
| `Trade Lane` (✅ implementado) | Objeto custom nuevo (dato de referencia) | ídem | Campos (API real): `originCountry`, `originPortOrCity`, `destinationCountry`, `destinationPortOrCity`. Vínculo con `Company` **corregido**: el SDK no tiene `MANY_TO_MANY` directo (solo `MANY_TO_ONE`/`ONE_TO_MANY`) — se implementó vía objeto pivote `CompanyTradeLane` con dos relaciones `MANY_TO_ONE` (a `Company` y a `Trade Lane`), cada una con su recíproco `ONE_TO_MANY`. Ver "Notas para el agente" para la receta completa |
| `OrganizationProfile` (✅ implementado) | Objeto custom nuevo, **singleton por workspace** (una sola fila por tenant — enforcement real pendiente, ver "Notas para el agente") | ídem | Configuración de la propia empresa del tenant — ver tabla de campos abajo |

### `OrganizationProfile` — la empresa del tenant, no la de sus clientes

| Campo | Tipo | Notas |
|---|---|---|
| `legal_name` (API real: `legalName`) | texto | Razón social del tenant (ej. "Sealion Cargo Inc.") |
| `trade_name` (API real: `tradeName`) | texto, nullable | Nombre comercial si difiere del legal |
| `tax_id` (API real: `taxId`) | texto | Identificador fiscal/de negocio del tenant — mismo criterio que `Company.taxId`: sin validación por país en esta fase |
| `default_currency` (API real: `defaultCurrency`) | texto (ISO 4217) | Moneda por defecto para `Quotation`/`Invoice` de este tenant — evita que cada cotización tenga que preguntarlo |
| `default_incoterms` (API real: `defaultIncoterms`) | select, valores `EXW`/`FCA`/`FOB`/`CIF`/`CPT`/`CIP`/`DAP`/`DPU`/`DDP` (mismo catálogo que Fase 3) | Default de la empresa, distinto del `preferred_incoterms` por cliente de `CustomerLogisticsProfile` (Fase 3) — ese es por cliente del tenant, este es el default general del tenant cuando no hay uno más específico |
| `primary_operating_countries` (API real: `primaryOperatingCountries`) | texto (códigos ISO 3166-1 alpha-2 separados por coma) | En qué países opera este tenant — informativo, no dispara reglas de compliance por sí solo. Simplificación deliberada: texto libre en vez de un multi-select con ~250 países, ajustar si hace falta una UI más estructurada |
| `logo_reference` (API real: `logoReference`) | texto, nullable | Referencia al logo del tenant para documentos/cotizaciones/facturas generados por la plataforma (branding propio, no el de Cargo One) |
| `customs_filing_mode` (API real: `customsFilingMode`) | select, valores `OWN_LICENSED_BROKER`/`PARTNER_BROKER`/`NOT_CONFIGURED` | **Reemplaza** el enfoque anterior de tratar esto como una pregunta única para todo el sistema (ver corrección en Fase 11): cada tenant configura lo suyo aquí |
| `customs_broker_of_record` (API real: `customsBrokerOfRecord`) | relación `MANY_TO_ONE` a `Company` | Referencia a la `Company` (rol `customs_broker`, puede ser una Company del propio tenant si tiene broker in-house) que presenta CADs/entries en su nombre. Reciprocada por `organizationProfilesAsBroker` (`ONE_TO_MANY`) en `Company` |

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
      desarrollo (Fase 14), no en el sandbox de agente originalmente.
- [x] `company_role`, `tax_id`, `industry_vertical`, `contact_role` creados y verificados
      contra el servidor real (`plan` limpio + confirmado visualmente en Configuración →
      Modelo de datos).
- [x] Nombres de API en camelCase alfanumérico, valores de opciones en UPPER_CASE (convención
      real del servidor, corregida en `CLAUDE.md` durante esta implementación — no era
      snake_case como se había asumido originalmente), labels en Title Case.
- [x] `Trade Lane` creado (4 campos propios + objeto pivote `CompanyTradeLane` con relación
      a `Company`), verificado en Configuración → Modelo de datos de la instancia real
      (los 4 campos de relación confirmados presentes, no solo por el diff de `plan`).
- [x] `OrganizationProfile` creado con sus 9 campos (incluida la relación `MANY_TO_ONE` a
      `Company`), verificado en Configuración → Modelo de datos — ambos lados de la relación
      (`customsBrokerOfRecord` en `OrganizationProfile`, `organizationProfilesAsBroker` en
      `Company`) confirmados presentes.
- [ ] `OrganizationProfile` verificado como singleton real (no permite dos filas en el mismo
      workspace) — **deliberadamente no resuelto en esta ronda**: el Apps SDK no expone una
      opción de "objeto singleton" a nivel de metadata; enforcement real requeriría un
      `logicFunction` (validación server-side) que es una unidad de trabajo aparte — no
      bloquea el cierre de esta fase, queda anotado para una sesión futura.
- [x] No se duplicó ningún campo que ya exista nativo en Twenty (Company/Person) — confirmado
      contra el esquema real antes de crear cada campo nuevo.
- [x] `docs/00-master-index.md` actualizado a 🟢 Cerrada — verificado contra una instancia
      real, no solo especificado.

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
- **Receta real para relaciones (probada de punta a punta con `Trade Lane` ↔ `Company`):**
  - El SDK **no tiene `MANY_TO_MANY`** — el enum `RelationType` solo tiene `MANY_TO_ONE` y
    `ONE_TO_MANY` (confirmado en `node_modules/twenty-sdk/dist/define/index.d.ts`). Para una
    relación many-to-many real hace falta un **objeto pivote** con dos relaciones
    `MANY_TO_ONE` (una a cada objeto que se quiere vincular).
  - Cada campo `RELATION` necesita `relationTargetObjectMetadataUniversalIdentifier` (el
    objeto al que apunta) y `relationTargetFieldMetadataUniversalIdentifier` (el campo
    recíproco del otro lado) — **los dos campos de un par recíproco se referencian
    mutuamente**, así que hay que generar los UUIDs de ambos lados *antes* de escribir
    cualquiera de los dos archivos. El emparejamiento correcto es: el campo A
    (`MANY_TO_ONE` hacia objeto X) reciproca con el campo B que **vive en X** — no con
    cualquier otro campo. Cruzar el emparejamiento produce una relación lógicamente rota
    (se detectó y corrigió este error concreto en esta sesión antes de aplicar).
  - Para un objeto **estándar** de Twenty (Company, Person), el campo de relación va en un
    archivo nuevo en `src/fields/`, igual que cualquier otro campo custom sobre ese objeto.
  - Para un objeto **propio** (ej. `Trade Lane`, `CompanyTradeLane`), el campo va **inline**
    en el array `fields: [...]` del propio archivo `src/objects/<objeto>.ts` — no como
    archivo separado.
  - `yarn twenty plan` puede mostrar **menos entradas de las esperadas** para un par de
    campos recíprocos (ej. mostrar solo 1 de 2) — esto se verificó empíricamente que es
    comportamiento normal del diff (probablemente trata el par como una sola relación
    física a efectos de visualización), no un bug que vaya a dejar la relación a medio
    crear. **No asumir esto sin verificar**: después de `apply`, confirmar en
    Configuración → Modelo de datos de Twenty que ambos lados existen de verdad, en vez de
    confiar solo en el conteo del `plan`.
  - **La misma omisión pasa con campos simples (no solo relaciones) cuando el objeto entero
    es nuevo:** al crear `OrganizationProfile` con 8 campos `TEXT`/`SELECT` propios más 1
    `RELATION`, el diff de `plan` no listó ninguno de los 8 campos simples individualmente
    (solo el `objectMetadata`, la relación, y el scaffolding estándar) — mismo patrón,
    mismo verificado-como-no-bug: los 9 campos existían de verdad al confirmar en la UI
    después de `apply`. Regla general: **si el objeto es nuevo en esta misma sesión de
    `plan`/`apply`, no confiar en que el diff liste cada campo propio individualmente** —
    verificar el archivo fuente (`cat` del objeto) está bien escrito, y verificar el
    resultado final en la UI, no el conteo de líneas del diff.
  - Antes de confiar en un análisis de "por qué el diff se ve así" basado en código fuente
    de Twenty: **este VPS no tiene el código fuente de `twenty-server` ni el monorepo
    completo de Twenty** (ver ADR-003) — solo `node_modules/twenty-sdk/dist/*` (compilado).
    Un reporte que cite archivos `.ts` de `packages/twenty-server/src/...` con números de
    línea específicos no puede ser una lectura real de este servidor — pasó una vez en esta
    sesión y resultó ser información reconstruida de memoria, no evidencia real. Ante duda
    sobre comportamiento del servidor, verificar empíricamente contra la UI real, no confiar
    en una "investigación" de código que no puede haberse leído desde acá.
  - Un objeto pivote técnico (solo de vinculación, no algo que el usuario navegue
    directamente) no necesita vista/navegación/layout automáticos — declinar esa opción del
    wizard (`n`) para no ensuciar la navegación principal, consistente con el principio de
    UX de `CLAUDE.md`.
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
