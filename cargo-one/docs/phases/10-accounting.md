# Fase 10 — Accounting

**Estado:** 🟡 En curso — **propuesta de modelo de datos únicamente. REQUIERE REVISIÓN
HUMANA** antes de considerarse siquiera un borrador utilizable (`CLAUDE.md` regla 3:
"Cualquier lógica de Accounting/Billing/Payments"). Nada de este documento debe leerse como
decidido.

## Contexto

Depende de: Fase 1 (cerrada), Fase 5 (spec/modelo lista). Bounded context: **Accounting**,
fila 8 de la tabla de `docs/phases/01-architecture.md` sección 3 — servicio satélite propio
(`services/accounting-service`), downstream de Sales (Fase 4: `Quotation` → `Invoice`) y de
Shipment (Fase 5: eventos facturables).

**Por qué esta fase es distinta a todas las anteriores:** de Fase 2 a 9, este agente pudo
proponer un modelo completo y dejarlo en 🟡 esperando solo infraestructura (Docker/yarn) o
una revisión de calidad. Acá el checkpoint no es de infraestructura ni de calidad — es una
regla explícita de `CLAUDE.md` que **no se negocia**: ningún cálculo de facturación, ninguna
regla contable, ningún flujo de pago se implementa ni se aprueba sin que Carlos lo revise
primero. Esta fase entrega **forma de almacenamiento**, no lógica de negocio.

## Modelo de datos (propuesto, no decidido)

### Agregado raíz: `Invoice`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `invoice_number` | texto, único por tenant | |
| `customer_company_id` | UUID | ACL a Twenty (`Company`) |
| `related_quotation_id` | UUID, nullable | Referencia no forzada al `Quotation` de Fase 4 |
| `related_shipment_id` | UUID, nullable | Referencia no forzada al `Shipment` de Fase 5 |
| `status` | enum | `draft`, `issued`, `paid`, `overdue`, `cancelled` — máquina de estados de **almacenamiento**, no de negocio: qué transición dispara qué (ej. cuándo algo pasa a `overdue`) es una regla que no se define aquí |
| `currency` | texto (ISO 4217) | |
| `issue_date` / `due_date` | fecha, nullable | |
| `total_amount` | decimal | Suma de `InvoiceLine.amount` — aritmética de visualización, no un cálculo contable |

### Entidad: `InvoiceLine` (relación many-to-one a `Invoice`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `invoice_id` | UUID | FK a `Invoice` |
| `charge_code` | texto | Mismo concepto que `QuotationLine.charge_code` (Fase 4) — **copia**, no referencia viva: una factura no debe cambiar si se edita la cotización que la originó |
| `description` / `quantity` / `unit_price` / `amount` | — | Mismos tipos que `QuotationLine` |

### Entidad: `Payment` (relación many-to-one a `Invoice`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `invoice_id` | UUID | FK a `Invoice` |
| `amount` / `currency` | — | |
| `method` | texto libre | Sin catálogo cerrado — definir métodos reales de pago es una decisión de negocio, no de esta fase |
| `received_at` | timestamp, nullable | |
| `status` | enum | `pending`, `confirmed`, `failed`, `refunded` |

### Entidad: `LedgerEntry` (relación many-to-one a `Invoice`, opcional)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | |
| `entry_type` | enum | `debit`, `credit` |
| `account` | **texto libre, sin catálogo** | Un chart of accounts real es una decisión contable que le corresponde a un contador o al propio Carlos — este documento no inventa uno |
| `amount` / `currency` | — | |
| `related_invoice_id` | UUID, nullable | |
| `created_at` | timestamp | |

**Nota deliberada sobre `LedgerEntry`:** se incluye como placeholder de que *algún día* puede
hacer falta un libro mayor real, pero esta fase no define partida doble, no valida que
débitos = créditos, no define un chart of accounts, y no genera `LedgerEntry` automáticamente
desde ningún evento. Es forma vacía, no contabilidad funcionando.

## Explícitamente fuera de esta fase (y de cualquier implementación sin revisión humana)

- Cualquier fórmula de cálculo: impuestos, conversión de moneda, recargos, descuentos,
  intereses por mora.
- Reconocimiento de ingresos, cierre contable, o cualquier regla de "cuándo" registrar algo.
- Integración con pasarelas de pago reales.
- Reglas de crédito (aprobar/rechazar una factura según `credit_terms` de
  `CustomerLogisticsProfile`, Fase 3) — el dato existe desde Fase 3, la regla de qué hacer
  con él no se infiere aquí.
- Reportes financieros o exportación a un sistema contable externo.
- El código del servicio (`services/accounting-service`).

## Criterios de aceptación

- [ ] **Revisión humana explícita de este documento completo** (Carlos) — sin esto, ningún
      otro criterio de esta fase puede marcarse como cumplido, ni siquiera el modelo de datos.
- [ ] Chart of accounts real (si se decide usar `LedgerEntry`) definido por el humano, no
      inferido.
- [ ] Reglas de negocio de facturación (transiciones de `status`, cuándo se considera
      `overdue`, etc.) confirmadas por el humano antes de codificarse.
- [ ] `services/accounting-service` implementado y con las migraciones correspondientes —
      **solo después** de los dos puntos anteriores.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo con evidencia de revisión
      humana explícita, no solo código funcionando.

## Notas para el agente

- **REQUIERE REVISIÓN HUMANA** (`CLAUDE.md` regla 3) — esta fase, a diferencia de todas las
  anteriores, no se puede avanzar a 🟢 ni "por defecto" ni por criterio del agente. Cualquier
  sesión que retome esto y sienta la tentación de implementar una regla de cálculo "razonable"
  (ej. "seguramente el IVA es X%") debe detenerse y preguntar — un error acá no es un bug, es
  dinero real de un cliente real.
- Bounded context involucrado: **Accounting**, downstream de **Sales** y **Shipment**.
  `InvoiceLine.charge_code` copia el valor de `QuotationLine` deliberadamente (no referencia
  viva) — si una sesión futura "optimiza" esto a una referencia compartida, rompe la garantía
  de que una factura emitida no cambia si se edita una cotización vieja.
- Este documento es una **propuesta de forma de almacenamiento**, escrita por un agente sin
  autoridad para tomar decisiones contables. No es un plan de implementación aprobado.
