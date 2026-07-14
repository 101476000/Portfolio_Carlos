# Fase 8 — Ground

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**: depende de que exista `services/shipment-service/` (Fase 5).

## Contexto

Depende de: Fase 5 (spec/modelo lista, revisión humana pendiente). Bounded context:
**Ground**, fila 5 de la tabla de `docs/phases/01-architecture.md` sección 3 — extensión de
modo del shared kernel `Shipment`, mismo patrón que Fases 6 (Ocean) y 7 (Air).

## Modelo de datos

### Entidad: `GroundShipmentDetails` (relación 1:1 con `Shipment`, solo cuando `transport_mode = 'ground'`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID, único | FK a `Shipment` (Fase 5) — 1:1 |
| `carrier_name` | texto | |
| `trailer_type` | enum | `dry_van`, `reefer`, `flatbed`, `other` — a nivel de shipment, no de unidad de carga (ver nota abajo) |
| `trailer_number` | texto, nullable | |
| `bill_of_lading_number` | texto, nullable | BOL terrestre — se completa cuando el carrier lo emite |
| `origin_address` / `destination_address` | texto | Dirección operativa real de pickup/delivery — texto libre en esta fase, **no** un modelo de direcciones estructurado (calle/ciudad/código postal por campo); se define solo si una fase futura lo necesita, no se anticipa aquí |
| `etd` / `atd` | fecha, nullable | Estimated/Actual Time of Departure (pickup) |
| `eta` / `ata` | fecha, nullable | Estimated/Actual Time of Arrival (delivery) |

**Por qué no hay `GroundCargoUnitDetails`:** a diferencia de Ocean (tipo ISO de contenedor) y
Air (tipo de ULD), el transporte terrestre en el alcance de esta fase no necesita metadata
adicional por unidad de carga — los campos genéricos de `CargoUnit` (Fase 5) ya alcanzan. El
dato específico de modo que sí existe (`trailer_type`) es del shipment completo, no por
unidad, así que vive en `GroundShipmentDetails`. No se crea una tabla vacía solo para
mantener simetría con Fase 6/7 — sería una abstracción sin uso real.

## Alcance de esta fase

**Incluido:** modelo de datos de `GroundShipmentDetails` únicamente (ver justificación arriba
de por qué no hace falta una tabla de extensión de `CargoUnit`).

**Explícitamente fuera de esta fase:**
- Integración real con APIs de carriers terrestres o tracking GPS — conector futuro bajo
  `integrations/` si el negocio lo pide.
- Modelo estructurado de direcciones (geocoding, validación postal) — se usa texto libre
  hasta que una fase futura (o un ADR) justifique la inversión.
- Cálculo de flete terrestre o cualquier cifra — Sales (Fase 4)/Accounting (Fase 10,
  checkpoint humano).
- HS codes, descripción arancelaria, o cualquier dato de aduana — Customs (Fase 10-11,
  checkpoint humano obligatorio). Esto aplica incluso a cruces fronterizos terrestres
  (ej. México-USA-Canadá) — ningún dato ni regla de aduana se infiere en esta fase.
- El código del servicio — se implementa junto con Fase 5/6/7 en el mismo proyecto NestJS.

## Criterios de aceptación

- [x] Modelo de datos de `GroundShipmentDetails` definido, con relación 1:1 explícita a
      `Shipment` (Fase 5), y justificación explícita de por qué no hay tabla de extensión de
      `CargoUnit` (a diferencia de Fase 6/7).
- [ ] Migraciones creadas junto con las de Fase 5/6/7 en `services/shipment-service/`.
- [ ] Verificado que un `Shipment` con `transport_mode != 'ground'` nunca tiene un
      `GroundShipmentDetails` asociado (regla de aplicación).
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado.

## Notas para el agente

- Hereda el pedido de revisión de modelo de datos de Fase 5 — presentar junto con Fases 5, 6
  y 7 en la misma revisión humana.
- Bounded context involucrado: **Ground**, downstream de **Shipment**. Mismo criterio que
  Fase 6/7: un campo que resulte necesario en más de un modo pertenece al shared kernel.
- Si en el futuro aparece un dato real por unidad de carga específico de transporte
  terrestre (ej. requisitos de un tipo de pallet particular), crear `GroundCargoUnitDetails`
  en ese momento siguiendo el mismo patrón de Fase 6/7 — no es una omisión de esta fase, es
  una decisión deliberada de no construir una tabla sin caso de uso confirmado.
