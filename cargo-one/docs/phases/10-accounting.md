# Fase 10 — Accounting

**Estado:** 🟡 En curso — modelo de datos y reglas de almacenamiento **decididas** (checkpoint
humano levantado explícitamente por Carlos, ver `docs/decisions/004-human-checkpoint-waiver.md`).
Sigue sin implementarse código. **Excepción que no se levanta:** cualquier fórmula real de
impuestos/tasas sigue sin definirse aquí — no es un checkpoint, es que ese dato varía por
jurisdicción/producto y no hay una fuente única y confiable que un agente pueda investigar de
forma genérica (a diferencia de Customs, Fase 11, donde sí hay un proceso oficial documentado).

## Contexto

Depende de: Fase 1 (cerrada), Fase 5 (spec/modelo lista). Bounded context: **Accounting**,
fila 8 de la tabla de `docs/phases/01-architecture.md` sección 3 — servicio satélite propio
(`services/accounting-service`), downstream de Sales (Fase 4: `Quotation` → `Invoice`) y de
Shipment (Fase 5: eventos facturables).

## Modelo de datos

### Agregado raíz: `Invoice`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `invoice_number` | texto, único por tenant | |
| `customer_company_id` | UUID | ACL a Twenty (`Company`) |
| `related_quotation_id` | UUID, nullable | Referencia no forzada al `Quotation` de Fase 4 |
| `related_shipment_id` | UUID, nullable | Referencia no forzada al `Shipment` de Fase 5 |
| `status` | enum | `draft`, `issued`, `paid`, `partially_paid`, `overdue`, `cancelled` |
| `currency` | texto (ISO 4217) | |
| `issue_date` / `due_date` | fecha, nullable | |
| `total_amount` | decimal | Suma de `InvoiceLine.amount` |
| `amount_paid` | decimal | Suma de `Payment.amount` con `status = confirmed` — permite derivar `partially_paid` sin recalcular en cada consulta |

**Transiciones de `status` (decidido):** `draft → issued` (acción manual, al enviar la
factura al cliente) · `issued → partially_paid` (cuando `amount_paid > 0` y `< total_amount`)
· `issued`/`partially_paid` → `paid` (cuando `amount_paid >= total_amount`) ·
`issued`/`partially_paid` → `overdue` (job diario: `due_date` pasado y `amount_paid <
total_amount`) · cualquier estado → `cancelled` (acción manual, con motivo obligatorio en el
código, no modelado como campo separado en esta fase). Esto es lógica de aplicación estándar
de facturación, no una regla contable especializada — se decide aquí porque no depende de
jurisdicción ni de un dato externo.

### Entidad: `InvoiceLine` (relación many-to-one a `Invoice`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `invoice_id` | UUID | FK a `Invoice` |
| `charge_code` | texto | Copia de `QuotationLine.charge_code` (Fase 4) — no referencia viva: una factura emitida no cambia si se edita la cotización que la originó |
| `description` / `quantity` / `unit_price` / `amount` | — | Mismos tipos que `QuotationLine` |
| `account_code` | texto | Ver chart of accounts propuesto abajo — a qué cuenta contable corresponde esta línea |

### Entidad: `Payment` (relación many-to-one a `Invoice`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `invoice_id` | UUID | FK a `Invoice` |
| `amount` / `currency` | — | |
| `method` | enum | `wire_transfer`, `credit_card`, `ach`, `check`, `other` — catálogo cerrado decidido; `other` cubre casos no anticipados sin bloquear el registro |
| `received_at` | timestamp, nullable | |
| `status` | enum | `pending`, `confirmed`, `failed`, `refunded` |

### Entidad: `LedgerEntry` (relación many-to-one a `Invoice`, opcional)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | |
| `entry_type` | enum | `debit`, `credit` |
| `account_code` | texto (ver chart of accounts) | |
| `amount` / `currency` | — | |
| `related_invoice_id` | UUID, nullable | |
| `created_at` | timestamp | |

### Chart of accounts propuesto (decidido como punto de partida, no forzado)

Catálogo simplificado apropiado para un freight forwarder — Carlos puede ajustarlo libremente
sin que eso bloquee el resto del modelo, ya no es checkpoint:

| Código | Cuenta | Tipo |
|---|---|---|
| `4000` | Freight Revenue | Ingreso |
| `4100` | Accessorial Revenue (recargos, almacenaje, etc.) | Ingreso |
| `5000` | Carrier Costs (flete pagado a carriers) | Costo |
| `5100` | Customs Broker Fees | Costo |
| `1100` | Accounts Receivable | Activo |
| `2100` | Accounts Payable | Pasivo |
| `2200` | Tax Payable | Pasivo — **el monto que va aquí no se calcula en esta fase**, ver más abajo |

**Partida doble:** esta fase no implementa una validación automática de que débitos = créditos
por asiento — se deja como regla de aplicación a implementar junto con el servicio, no como
parte del modelo de datos.

## Lo que sigue sin definirse aquí (y por qué)

- **Cálculo de impuestos/tasas** (GST/HST por provincia canadiense, sales tax por estado
  en EE.UU., IVA si aplica a otras jurisdicciones de Sealion Cargo): a diferencia de Customs
  (Fase 11), no hay una única fuente oficial que un agente pueda investigar y aplicar de forma
  genérica — depende de dónde opera cada cliente, qué se factura, y reglas que cambian por
  jurisdicción. Recomendación técnica (no una regla de negocio): integrar un servicio de
  cálculo de impuestos de terceros (ej. Avalara, TaxJar) en vez de hardcodear tasas — eso sí
  es una decisión de implementación razonable de tomar sin bloquear, pero elegir el proveedor
  específico y confirmar cobertura de jurisdicciones queda para cuando se implemente el
  servicio.
- Reglas de crédito (aprobar/rechazar una factura según `credit_terms` de
  `CustomerLogisticsProfile`, Fase 3) — el dato existe desde Fase 3; la regla de negocio de
  qué hacer con él (¿bloquear el shipment? ¿solo alertar?) es una decisión operativa que
  Carlos puede definir cuando quiera, no bloquea el modelo de datos.
- Integración con pasarelas de pago reales, reportes financieros, exportación a sistema
  contable externo — decisiones de implementación, no de esta fase.
- El código del servicio (`services/accounting-service`).

## Criterios de aceptación

- [x] Modelo de datos decidido: `Invoice`/`InvoiceLine`/`Payment`/`LedgerEntry`, transiciones
      de estado de factura, chart of accounts de partida.
- [ ] `services/accounting-service` implementado con las migraciones correspondientes.
- [ ] Proveedor de cálculo de impuestos evaluado/integrado (o decisión explícita de posponerlo
      si el volumen inicial no lo justifica) — sigue siendo una decisión pendiente, ya no
      bloqueante del resto de la fase.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada cuando el código exista y esté
      verificado.

## Notas para el agente

- Checkpoint humano levantado por ADR-004 — se puede avanzar código real sobre este modelo
  sin pausar a pedir revisión, pero seguir sin inventar tasas de impuestos específicas: eso no
  es indecisión burocrática, es que no hay una fuente confiable y genérica que consultar (a
  diferencia de CBSA/CBP en Fase 11, que sí publican el proceso).
- Bounded context involucrado: **Accounting**, downstream de **Sales** y **Shipment**.
  `InvoiceLine.charge_code` copia el valor de `QuotationLine` deliberadamente — no convertir
  en referencia viva.
- Migraciones de base de datos en **producción** siguen siendo checkpoint humano obligatorio
  sin excepción (`CLAUDE.md` regla 3, no tocada por ADR-004) — esto aplica igual en Accounting
  que en cualquier otro servicio.
