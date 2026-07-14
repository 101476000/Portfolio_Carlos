# Fase 6 — Ocean

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**: depende de que exista `services/shipment-service/` (Fase 5).

## Contexto

Depende de: Fase 5 (spec/modelo lista, revisión humana pendiente — ver
`docs/phases/05-shipment.md`). Bounded context: **Ocean**, fila 3 de la tabla de
`docs/phases/01-architecture.md` sección 3 — extensión de modo del shared kernel `Shipment`,
vive en el mismo servicio (`services/shipment-service`), no es un servicio nuevo.

Esta fase agrega lo que un embarque marítimo necesita y que `Shipment` (Fase 5) dejó fuera a
propósito: vessel/voyage, puertos operativos, Bill of Lading, y el tipo ISO de contenedor.
Ninguna tabla de Fase 5 se modifica — todo vive en tablas de extensión 1:1 referenciando
`shipment_id`/`cargo_unit_id`, tal como esa fase lo dejó previsto.

## Modelo de datos

### Entidad: `OceanShipmentDetails` (relación 1:1 con `Shipment`, solo cuando `transport_mode = 'ocean'`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `shipment_id` | UUID, único | FK a `Shipment` (Fase 5) — 1:1, no many-to-one |
| `vessel_name` | texto | |
| `vessel_imo_number` | texto, nullable | Identificador IMO del buque, si se conoce |
| `voyage_number` | texto | |
| `carrier_scac_code` | texto, nullable | Standard Carrier Alpha Code — identifica al carrier naviero |
| `port_of_loading` | texto (UN/LOCODE recomendado) | Puerto operativo real de embarque — **distinto** de `Trade Lane.origin_port_or_city` (Fase 2), que es dato comercial/de referencia más amplio (ej. trade lane "China → USA West Coast" vs. booking real "Shanghai → Los Angeles") |
| `port_of_discharge` | texto (UN/LOCODE recomendado) | Puerto operativo real de descarga — mismo razonamiento que `port_of_loading` |
| `bill_of_lading_number` | texto, nullable | Se completa cuando el carrier lo emite, no necesariamente al crear el registro |
| `bill_of_lading_type` | enum | `original`, `seaway`, `telex_release` |
| `etd` / `atd` | fecha, nullable | Estimated/Actual Time of Departure |
| `eta` / `ata` | fecha, nullable | Estimated/Actual Time of Arrival |

### Entidad: `OceanContainerDetails` (relación 1:1 con `CargoUnit`, solo cuando `unit_type = 'container'` dentro de un shipment `ocean`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `cargo_unit_id` | UUID, único | FK a `CargoUnit` (Fase 5) — 1:1 |
| `container_iso_type` | enum | `20GP`, `40GP`, `40HC`, `40RF`, `20OT`, `40OT`, `20FR`, `40FR` (catálogo ISO 6346 más común — no exhaustivo, se amplía si un caso real lo requiere, no se anticipa aquí) |
| `seal_number` | texto, nullable | |
| `tare_weight_kg` | decimal, nullable | Peso del contenedor vacío — distinto del `weight_kg` de `CargoUnit` (Fase 5), que es el peso de la carga |

## Alcance de esta fase

**Incluido:** modelo de datos de `OceanShipmentDetails` y `OceanContainerDetails` como
extensiones 1:1 sobre el modelo de Fase 5, sin modificar sus tablas.

**Explícitamente fuera de esta fase:**
- Integración real con APIs de carriers navieros o tracking de buques (AIS) — eso es un
  conector nuevo bajo `integrations/`, a definir en una fase futura si el negocio lo pide; no
  se inventa aquí.
- Jerarquía Master BL / House BL (embarques consolidados) — esta fase modela un BL por
  shipment. Si el negocio necesita consolidación, es una decisión de modelo que requiere
  input humano explícito (afecta cómo se relacionan `Shipment` entre sí) — no se asume.
- Cálculo de flete, demurrage, detention, o cualquier cifra — Sales (Fase 4) para la
  cotización, Accounting (Fase 10, checkpoint humano) para cualquier cargo real.
- HS codes, descripción arancelaria, o cualquier dato de aduana — Customs (Fase 10-11,
  checkpoint humano obligatorio).
- El código del servicio — igual que Fase 5, esta fase entrega el modelo; la implementación
  ocurre junto con (o después de) la de Fase 5, en el mismo proyecto NestJS.

## Criterios de aceptación

- [x] Modelo de datos de `OceanShipmentDetails` y `OceanContainerDetails` definido, con
      relación 1:1 explícita a las tablas de Fase 5 (no las modifica).
- [x] Distinción documentada entre datos comerciales (Trade Lane, Fase 2) y datos operativos
      (puertos reales, Fase 6) para que no se confundan ni se dupliquen sin razón.
- [ ] Migraciones de estas dos tablas creadas junto con las de Fase 5 en
      `services/shipment-service/`, corriendo contra un schema `logistics_{workspace_uuid}`
      de prueba.
- [ ] Verificado que un `Shipment` con `transport_mode != 'ocean'` nunca tiene un
      `OceanShipmentDetails` asociado (regla de aplicación, no constraint de base — anotar
      cómo se decide validar esto al implementar).
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado.

## Notas para el agente

- No hay checkpoint humano obligatorio listado para esta fase en `docs/00-master-index.md`,
  pero **hereda** el pedido de revisión de modelo de datos de Fase 5 — no tiene sentido que
  Carlos revise `Shipment` sin ver también sus extensiones de modo, ya que juntas forman el
  modelo real que va a usar el negocio. Presentar ambas fases juntas en esa revisión.
- Bounded context involucrado: **Ocean**, downstream de **Shipment** (shared kernel). Ningún
  campo de esta fase debe aparecer en las tablas de Fase 5 ni viceversa — si durante la
  implementación alguien necesita un campo de Ocean disponible para Air o Ground también, es
  señal de que ese campo en realidad pertenece al shared kernel y hay que moverlo a Fase 5
  (con la revisión humana correspondiente antes de mover nada).
- **Patrón a replicar en Fase 7 (Air) y Fase 8 (Ground):** una tabla `<Modo>ShipmentDetails`
  1:1 con `Shipment` y, si aplica, una `<Modo>ContainerDetails`/`<Modo>PackageDetails` 1:1 con
  `CargoUnit` — mismo patrón de extensión que esta fase, para que las tres fases de modo sean
  estructuralmente consistentes entre sí.
