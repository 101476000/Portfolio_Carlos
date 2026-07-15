# Fase 4 — Sales

**Estado:** 🟡 En curso — work order especificado, `quotation-app` ya escafoldado en
`core/twenty-apps/quotation-app/` (ver `docs/decisions/003-twenty-apps-scaffolding.md`).
Implementación real pendiente: mismo bloqueante que Fase 2/3 (`yarn install` + instancia de
Twenty vía Docker, no disponibles en este sandbox — ver `core/twenty-apps/README.md`).

## Contexto

Depende de: Fase 1 (cerrada), Fase 2 (spec lista), Fase 3 (spec lista). Bounded context:
**CRM & Sales**, fila 1 de la tabla de contextos en `docs/phases/01-architecture.md` sección 3
— esta fase cubre la **porción Sales**, la última de las tres (Fase 2 = vocabulario CRM,
Fase 3 = perfil de cliente, Fase 4 = cotización).

Twenty ya trae un pipeline de ventas nativo (`Opportunity`, con stages). Esta fase **no
reconstruye ese pipeline** — lo extiende con lo que Twenty no modela: un documento de
cotización formal con líneas de tarifa (`Quotation`), enlazado 1:1 a una `Opportunity`. Es el
artefacto que, al aceptarse, es el puente hacia el dominio logístico (Fase 5 — Shipment):
`docs/phases/01-architecture.md` sección 3 ya señala esta relación ("CRM & Sales —upstream—▶
Shipment").

## Alcance de esta fase

**Incluido:**

| Elemento | Tipo | Dónde vive | Descripción |
|---|---|---|---|
| `Quotation` | Objeto custom nuevo (agregado raíz de esta fase) | Twenty, vía `quotation-app` | Relación 1:1 con `Opportunity` nativo de Twenty (no reemplaza su pipeline, lo complementa) |
| `quotation_number` (API real: `quotationNumber`) | Campo (texto, único) en `Quotation` | ídem | Identificador legible para el cliente (formato a definir en implementación, ej. secuencial por workspace) |
| `status` | Campo (select) en `Quotation` | ídem | Valores UPPER_CASE (convención real de Twenty, ver Fase 2/`CLAUDE.md`): `DRAFT`, `SENT`, `ACCEPTED`, `REJECTED`, `EXPIRED`, `SUPERSEDED` — máquina de estados simple, sin lógica de negocio automática en esta fase |
| `valid_until` (API real: `validUntil`) | Campo (fecha) en `Quotation` | ídem | Vigencia de la cotización |
| `transport_mode` (API real: `transportMode`) | Campo (select) en `Quotation` | ídem | Valores UPPER_CASE: `OCEAN`, `AIR`, `GROUND`, `MULTIMODAL` — anticipa Fases 6-8, no las implementa |
| `trade_lane` (API real: `tradeLane`) | Relación a `Trade Lane` (Fase 2) | ídem | Reutiliza el dato de referencia de Fase 2, no lo duplica |
| `incoterm` | Campo (select, mismo catálogo que Fase 3) en `Quotation` | ídem | Por defecto toma `preferred_incoterms` de `CustomerLogisticsProfile` (Fase 3) si existe, pero es editable por cotización — una cotización puntual puede pactar un INCOTERM distinto al preferido del cliente |
| `QuotationLine` | Objeto custom hijo (relación many-to-one a `Quotation`) | ídem | Campos (API real en camelCase): `chargeCode` (texto), `description`, `unitType` (select, valores UPPER_CASE ej. `PER_CONTAINER`, `PER_KG`, `FLAT`), `quantity`, `unitPrice`, `currency` (ISO 4217), `amount` |
| `total_amount` | Campo calculado/almacenado en `Quotation` | ídem | Suma de `amount` de sus `QuotationLine` — aritmética simple de visualización, no lógica de facturación |

**Explícitamente fuera de esta fase:**
- Cualquier motor de tarifas/rate card automático — en esta fase el usuario ingresa
  `unit_price`/`amount` manualmente en cada `QuotationLine`, no se calcula ni se infiere.
- Cálculo de impuestos, duties, o cualquier cifra relacionada a CAD — nunca vive en Sales,
  vive en Customs (Fase 10-11, checkpoint humano obligatorio).
- Facturación real, términos de pago, o cualquier lógica de Accounting — Fase 10, checkpoint
  humano obligatorio. `total_amount` aquí es informativo para el cliente, no genera ningún
  asiento contable.
- Creación automática de un `Shipment` cuando `status` pasa a `accepted` — Fase 5 (Shipment)
  todavía no existe como servicio. Esta fase deja documentado el contrato de evento que Fase 5
  deberá consumir (ver "Notas para el agente"), pero no lo implementa: no hay para quién
  emitirlo todavía.
- Reconstrucción del pipeline de `Opportunity` (stages, probabilidad, forecasting) — eso ya
  lo provee Twenty nativo, no se toca.

## Criterios de aceptación

- [x] `quotation-app` creado vía `npx create-twenty-app` (scaffold base).
- [ ] `yarn install` completado y `yarn twenty dev`/`yarn twenty remote:add` conectado a una
      instancia real de Twenty — **bloqueante para el resto de esta lista**; ver
      `core/twenty-apps/README.md`.
- [ ] `Quotation` y `QuotationLine` definidos como código dentro de `quotation-app`,
      sincronizados contra esa instancia, con la relación 1:1 a `Opportunity` funcionando.
- [ ] `trade_lane` e `incoterm` reutilizan los objetos/catálogos de Fase 2/3 — verificado que
      no se redefinió ningún catálogo ya existente.
- [ ] `total_amount` se recalcula correctamente al agregar/editar/eliminar una `QuotationLine`
      (verificado con datos de prueba), sin ninguna lógica de impuestos ni facturación mezclada.
- [ ] Nombres de API en `snake_case` inglés, labels en Title Case inglés.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando lo anterior esté
      verificado contra una instancia real, no solo especificado.

## Notas para el agente

- **No hay checkpoint humano obligatorio** para esta fase según `docs/00-master-index.md`,
  pero `total_amount`/`QuotationLine` son *vecinos* de Accounting — igual que en Fase 3, esta
  fase solo modela y muestra el dato, no lo factura ni lo concilia. Cualquier sesión que
  quiera conectar `Quotation` a un flujo de facturación real debe releer `CLAUDE.md` regla 3
  antes de tocar eso.
- El bloqueante de infraestructura de Fase 2 (`yarn install`/Docker) ya se resolvió en un VPS
  real (ver `docs/phases/02-crm.md` y `docs/phases/14-deployment.md`) — `quotation-app`
  todavía no repitió ese proceso ahí, pero el camino ya está probado. Al implementar esta
  fase, seguir la "receta real" documentada en Fase 2 (`dev:add` → completar
  `objectUniversalIdentifier` con `STANDARD_OBJECT_UNIVERSAL_IDENTIFIERS`/el identificador del
  objeto `Quotation` propio de esta app → `name` camelCase → `options` en UPPER_CASE →
  `plan` → `apply`), no reinventar el proceso desde cero.
- **Contrato de evento para Fase 5 (documentado, no implementado):** cuando `Quotation.status`
  pase a `accepted`, el evento a emitir sigue el envelope de
  `docs/phases/01-architecture.md` sección 4:
  ```
  {
    "workspace_id": "uuid",
    "event_type": "quotation.accepted",
    "payload": {
      "quotation_id": "uuid",
      "opportunity_id": "uuid",
      "customer_company_id": "uuid",
      "transport_mode": "ocean|air|ground|multimodal",
      "trade_lane_id": "uuid",
      "incoterm": "string"
    },
    "occurred_at": "ISO-8601",
    "correlation_id": "uuid"
  }
  ```
  No implementar el publisher de este evento hasta que exista un consumidor real (Fase 5) —
  emitir eventos sin consumidor es trabajo especulativo que además nadie podría probar.
- Bounded context involucrado: **CRM & Sales** (porción Sales, cierra el contexto completo
  entre Fase 2, 3 y 4). Con esta fase especificada, las tres porciones del contexto quedan
  consistentes entre sí — no debería hacer falta modificar Fase 2/3 para acomodar Fase 4.
