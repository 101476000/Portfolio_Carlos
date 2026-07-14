# Fase 3 — Customers

**Estado:** 🟡 En curso — work order especificado, implementación real pendiente (mismo
bloqueante que Fase 2: requiere `core/twenty/` con Twenty configurado).

## Contexto

Depende de: Fase 1 (cerrada), Fase 2 (spec lista, ver dependencia real en "Notas para el
agente"). Bounded context: **CRM & Sales**, fila 1 de la tabla de contextos en
`docs/phases/01-architecture.md` sección 3.

Fase 2 define el vocabulario base (roles de Company/Person, Trade Lane). Esta fase construye
sobre eso el **perfil operativo del cliente** — el value object `CustomerLogisticsProfile`
mencionado en `docs/phases/01-architecture.md` sección 3 — que es lo que Fase 4 (Sales) y
Fase 5 (Shipment) necesitan leer para cotizar y operar embarques sin tener que preguntarle al
usuario los mismos datos cada vez (INCOTERMS que prefiere el cliente, términos de crédito,
lanes que usa habitualmente).

**Por qué es una fase separada de Fase 2:** `company_role`/`Trade Lane` (Fase 2) son
vocabulario compartido por todo el CRM — también lo usa Sales, Shipment, Accounting. El
perfil de cliente en cambio es específico de la relación comercial con un `Company` que tiene
rol `shipper` o `consignee`, y algunos de sus campos (crédito, compliance) son sensibles —
tiene sentido aislarlo en su propio objeto con su propia fase, en vez de sobrecargar el
Company nativo de Twenty con campos que no aplican a un `carrier` o `vendor`.

## Alcance de esta fase

**Incluido:**

| Elemento | Tipo | Dónde vive | Descripción |
|---|---|---|---|
| `CustomerLogisticsProfile` | Objeto custom nuevo | Twenty, vía `sales-extensions-app` | Relación 1:1 con `Company` (solo aplica a Companies con `company_role` incluyendo `shipper` o `consignee`, definido en Fase 2) |
| `preferred_incoterms` | Campo (select) en `CustomerLogisticsProfile` | ídem | Catálogo cerrado: EXW, FCA, FOB, CIF, CPT, CIP, DAP, DPU, DDP (Incoterms 2020) — dato declarativo, no dispara ningún cálculo en esta fase |
| `credit_terms` | Value object embebido: `payment_terms_days` (entero), `credit_limit` (decimal), `currency` (ISO 4217) | ídem | Solo almacenamiento estructurado — **ninguna lógica de aprobación de crédito ni de facturación se implementa aquí**, eso es Fase 10 (Accounting), que tiene checkpoint humano obligatorio |
| `preferred_trade_lanes` | Relación many-to-many a `Trade Lane` (Fase 2) | ídem | Reutiliza el objeto de referencia de Fase 2, no lo duplica |
| `customs_broker_of_record` | Relación a `Company` con rol `customs_broker` | ídem | Solo referencia al broker asignado — ninguna lógica de aduana vive aquí (Fase 10/11) |

**Explícitamente fuera de esta fase:**
- Cualquier lógica de screening de partes denegadas ("denied party screening") o validación
  de compliance real. `CLAUDE.md` regla 4 prohíbe inventar reglas de compliance no
  documentadas — si el negocio necesita esto, es un ADR nuevo con input humano explícito,
  no una inferencia de esta fase. Esta fase no crea ni siquiera un campo placeholder para
  esto, precisamente para no sugerir que existe una verificación que no existe.
- Cálculo o aprobación de crédito — solo se almacena el dato declarado (Fase 10, checkpoint
  humano).
- Cualquier lógica de duties/taxes/CAD — nunca vive en CRM, vive en Customs (Fase 10-11,
  checkpoint humano).
- `Quotation` y pipeline de ventas — Fase 4.

## Criterios de aceptación

- [ ] Fase 2 verificada contra una instancia real de Twenty (no solo especificada) — este
      objeto depende de que `Trade Lane` y `company_role` ya existan de verdad.
- [ ] `CustomerLogisticsProfile` implementado en `sales-extensions-app` con los campos de la
      tabla de alcance, relación 1:1 a `Company` funcionando.
- [ ] Ningún campo de esta fase implementa lógica de crédito, compliance o aduana — solo
      almacenamiento estructurado, verificado por revisión de código antes de cerrar la fase.
- [ ] Nombres de API en `snake_case` inglés, labels en Title Case inglés.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando lo anterior esté
      verificado contra una instancia real.

## Notas para el agente

- **No hay checkpoint humano obligatorio** para esta fase según `docs/00-master-index.md`,
  pero dos de sus campos (`credit_terms`, `customs_broker_of_record`) son *vecinos* de dominios
  que sí lo tienen (Accounting, Customs). Esta fase solo modela el dato; cualquier sesión
  futura que quiera agregarle lógica sobre esos campos debe releer `CLAUDE.md` regla 3 antes
  de hacerlo, no asumir que por estar "cerca" del dato ya tiene permiso de operar sobre él.
- Mismo bloqueante que Fase 2: no existe `core/twenty/` en este repo todavía. No marcar 🟢
  hasta que Fase 2 esté realmente implementada y verificada (no solo su spec) y este objeto
  se haya creado contra esa instancia real.
- Bounded context involucrado: **CRM & Sales** (porción Customers). Si en el futuro un campo
  de `CustomerLogisticsProfile` empieza a necesitar lógica de negocio real (no solo
  almacenamiento), es señal de que esa lógica pertenece a otro bounded context (Accounting o
  Customs) y debe moverse ahí, no crecer dentro de CRM.
