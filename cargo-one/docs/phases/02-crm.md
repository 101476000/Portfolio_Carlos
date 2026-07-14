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

## Alcance de esta fase

**Incluido:**

| Elemento | Tipo | Dónde vive | Descripción |
|---|---|---|---|
| `company_role` | Campo custom (multi-select) en `Company` | Twenty, vía `sales-extensions-app` (Apps framework) | Valores: `shipper`, `consignee`, `carrier`, `customs_broker`, `partner_agent`, `vendor`. Una Company puede tener más de un rol (ej. un cliente que también es agente en otro país) |
| `tax_id` | Campo custom (texto) en `Company` | ídem | Identificador fiscal del país de operación (formato libre en esta fase — validación por país queda fuera, no se inventa regla de compliance sin documentarla) |
| `industry_vertical` | Campo custom (select) en `Company` | ídem | Clasificación comercial simple (ej. `retail`, `manufacturing`, `automotive`) para reporting, no para lógica de negocio |
| `contact_role` | Campo custom (multi-select) en `Person` | ídem | Valores: `primary`, `operations`, `billing`, `customs` — de qué trata cada contacto dentro de una Company |
| `Trade Lane` | Objeto custom nuevo (dato de referencia) | ídem | Campos: `origin_country`, `origin_port_or_city`, `destination_country`, `destination_port_or_city`. Relación many-to-many con `Company` (`primary_trade_lanes`) |

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
