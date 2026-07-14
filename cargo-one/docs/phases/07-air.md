# Fase 7 — Air

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**: depende de que exista `services/shipment-service/` (Fase 5).

## Contexto

Depende de: Fase 5 (spec/modelo lista, revisión humana pendiente). Bounded context: **Air**,
fila 4 de la tabla de `docs/phases/01-architecture.md` sección 3 — extensión de modo del
shared kernel `Shipment`, mismo patrón que Fase 6 (Ocean): tablas 1:1 sobre `shipment_id`/
`cargo_unit_id`, sin modificar el modelo base.

## Modelo de datos

### Entidad: `AirShipmentDetails` (relación 1:1 con `Shipment`, solo cuando `transport_mode = 'air'`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID, único | FK a `Shipment` (Fase 5) — 1:1 |
| `airline_name` | texto | |
| `airline_iata_code` | texto, nullable | Código IATA de dos letras de la aerolínea |
| `flight_number` | texto, nullable | |
| `awb_number` | texto, nullable | Air Waybill — se completa cuando la aerolínea lo emite |
| `awb_type` | enum | `master`, `house` |
| `airport_of_departure` / `airport_of_destination` | texto (código IATA de 3 letras recomendado) | Aeropuerto operativo real — mismo razonamiento que `port_of_loading`/`port_of_discharge` en Fase 6: distinto del dato comercial de `Trade Lane` (Fase 2) |
| `etd` / `atd` | fecha, nullable | Estimated/Actual Time of Departure |
| `eta` / `ata` | fecha, nullable | Estimated/Actual Time of Arrival |

### Entidad: `AirCargoUnitDetails` (relación 1:1 con `CargoUnit`, solo dentro de un shipment `air`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `cargo_unit_id` | UUID, único | FK a `CargoUnit` (Fase 5) — 1:1 |
| `uld_type` | texto, nullable | Unit Load Device (ej. `AKE`, `PMC`, `PAG`) — nullable porque no toda carga aérea va en ULD (puede ser bulk/loose) |
| `uld_number` | texto, nullable | |

## Alcance de esta fase

**Incluido:** modelo de datos de `AirShipmentDetails` y `AirCargoUnitDetails`, siguiendo
exactamente el mismo patrón de extensión que Fase 6 (Ocean) sobre el modelo de Fase 5.

**Explícitamente fuera de esta fase:**
- Integración real con APIs de aerolíneas o tracking de vuelos — conector futuro bajo
  `integrations/` si el negocio lo pide, no se inventa aquí.
- Jerarquía Master AWB / House AWB más allá del campo `awb_type` — modelar la relación
  completa (un Master AWB agrupando varios House AWB de distintos shippers) es una decisión
  de consolidación que, igual que en Fase 6, requiere input humano explícito antes de
  asumirla.
- Cálculo de flete aéreo, chargeable weight, o cualquier cifra — Sales (Fase 4)/Accounting
  (Fase 10, checkpoint humano). Esta fase no calcula `chargeable_weight` (max entre peso real
  y volumétrico) porque eso alimenta una tarifa, y las reglas de conversión volumétrica
  varían por acuerdo comercial — no se asume una fórmula sin que el negocio la confirme.
- HS codes, descripción arancelaria, o cualquier dato de aduana — Customs (Fase 10-11,
  checkpoint humano obligatorio).
- El código del servicio — se implementa junto con Fase 5/6 en el mismo proyecto NestJS.

## Criterios de aceptación

- [x] Modelo de datos de `AirShipmentDetails` y `AirCargoUnitDetails` definido, con relación
      1:1 explícita a las tablas de Fase 5.
- [x] Distinción documentada entre aeropuerto operativo real y `Trade Lane` comercial
      (mismo patrón que Fase 6, para consistencia entre fases de modo).
- [ ] Migraciones creadas junto con las de Fase 5/6 en `services/shipment-service/`.
- [ ] Verificado que un `Shipment` con `transport_mode != 'air'` nunca tiene un
      `AirShipmentDetails` asociado (regla de aplicación).
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado.

## Notas para el agente

- Hereda el pedido de revisión de modelo de datos de Fase 5, igual que Fase 6 — presentar
  las tres fases de modo (6, 7, 8) junto con Fase 5 en la misma revisión humana, no por
  separado.
- Bounded context involucrado: **Air**, downstream de **Shipment**. Mismo criterio que
  Fase 6: si un campo de esta fase resulta necesario también en Ocean o Ground, es señal de
  que pertenece al shared kernel (Fase 5), no aquí.
- Sigue el patrón de extensión establecido en Fase 6 — cualquier divergencia de ese patrón
  (nombres de campo, estructura de tablas) debe justificarse explícitamente, no improvisarse.
