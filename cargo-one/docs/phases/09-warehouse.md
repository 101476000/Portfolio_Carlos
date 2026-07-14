# Fase 9 — Warehouse

**Estado:** 🟡 En curso — work order especificado (modelo de datos incluido). Implementación
de código **no iniciada**: `services/warehouse-service/` todavía no existe como proyecto.

## Contexto

Depende de: Fase 5 (spec/modelo lista). Bounded context: **Warehouse**, fila 6 de la tabla de
`docs/phases/01-architecture.md` sección 3 — a diferencia de Fases 6-8 (Ocean/Air/Ground, que
extienden `shipment-service`), Warehouse **es un servicio satélite propio**
(`services/warehouse-service`), con su propia base Postgres (schema
`logistics_{workspace_uuid}`, mismo patrón de `docs/decisions/002-multi-tenant-strategy.md`).

`docs/phases/01-architecture.md` describe esta relación como "downstream y a la vez upstream
de Shipment" — es la primera fase donde eso se vuelve concreto:
- **Downstream:** un shipment puede terminar en un almacén (la carga llega y se guarda).
- **Upstream:** la carga almacenada puede salir generando un shipment nuevo (la salida de
  almacén dispara un booking, no al revés).

Como ninguna de las dos direcciones tiene todavía un consumidor real implementado (ni
`shipment-service` ni `warehouse-service` existen como código), esta fase documenta ambos
contratos de evento sin implementar los publishers — mismo criterio aplicado en Fases 4 y 5.

## Modelo de datos

### Entidad: `Facility` (dato maestro del tenant, no depende de un shipment)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `name` | texto | |
| `code` | texto, único por tenant | Identificador corto (ej. para referenciar en reportes) |
| `address` | texto | Texto libre en esta fase — mismo criterio que `origin_address`/`destination_address` en Fase 8, no se modela un address book estructurado sin caso de uso confirmado |

### Agregado raíz: `WarehouseReceipt`

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `receipt_number` | texto, único por tenant | |
| `facility_id` | UUID | FK a `Facility` |
| `customer_company_id` | UUID | Referencia a `Company` de Twenty (ACL, no se duplica el dato — mismo patrón que `Shipment.customer_company_id` en Fase 5) |
| `source_shipment_id` | UUID, nullable | Referencia **no forzada** (sin FK de base de datos — es otro servicio, otra base) al `Shipment` de `shipment-service` que trajo esta carga, si aplica. Nullable porque un cliente puede entregar carga directo al almacén sin un shipment trackeado |
| `status` | enum | `expected`, `received`, `in_storage`, `released` |
| `received_at` | timestamp, nullable | |

### Entidad: `StorageUnit` (relación many-to-one a `WarehouseReceipt`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `warehouse_receipt_id` | UUID | FK a `WarehouseReceipt` |
| `source_cargo_unit_id` | UUID, nullable | Referencia no forzada al `CargoUnit` de `shipment-service`, si esta unidad vino de un shipment trackeado — trazabilidad, no integridad referencial real (son bases distintas) |
| `unit_type` | enum | `container`, `package`, `pallet`, `loose_cargo` — mismo catálogo que `CargoUnit` (Fase 5), no se reinventa |
| `description` | texto | Copia local, no referencia viva — el almacén necesita saber qué tiene aunque la carga nunca haya pasado por `shipment-service` |
| `weight_kg` / `volume_cbm` | decimal | |
| `location_in_facility` | texto, nullable | Ubicación física dentro del almacén (rack/bin) — texto libre, no un modelo de slotting |
| `status` | enum | `in_storage`, `reserved_for_release`, `released` |

### Entidad: `InboundOrder` (relación many-to-one a `Facility`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `facility_id` | UUID | FK a `Facility` |
| `customer_company_id` | UUID | ACL a Twenty |
| `related_shipment_id` | UUID, nullable | Igual que `WarehouseReceipt.source_shipment_id` — referencia no forzada |
| `expected_arrival_date` | fecha, nullable | |
| `status` | enum | `expected`, `received`, `cancelled` — al pasar a `received`, se espera que exista un `WarehouseReceipt` asociado (regla de aplicación, no constraint) |

### Entidad: `OutboundOrder` (relación many-to-one a `Facility`)

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK |
| `workspace_id` | UUID | Tenant |
| `facility_id` | UUID | FK a `Facility` |
| `customer_company_id` | UUID | ACL a Twenty |
| `requested_date` | fecha, nullable | |
| `status` | enum | `requested`, `picked`, `released` |
| `resulting_shipment_id` | UUID, nullable | Se completa si la salida genera un shipment nuevo (ver contrato de evento abajo) — nullable porque una salida puede ser un retiro directo del cliente sin transporte gestionado por Cargo One |

**Relación `OutboundOrder` ↔ `StorageUnit`:** tabla de asociación simple
(`outbound_order_id`, `storage_unit_id`) — qué unidades específicas cubre cada salida. No se
modela como campo en `StorageUnit` porque una unidad podría teóricamente considerarse para
más de una orden antes de confirmarse cuál la cubre finalmente (evita bloquear el dato antes
de tiempo).

## Integración

- **Consumo opcional, no automático en esta fase:** `WarehouseReceipt.source_shipment_id` e
  `InboundOrder.related_shipment_id` se completan manualmente o por una automatización futura
  — esta fase **no** implementa un listener que reaccione automáticamente a
  `shipment.milestone.recorded` (Fase 5), porque no todo shipment termina en un almacén y
  decidir cuáles sí es una regla de negocio que no está definida — no se asume.
- **Emite** (contrato documentado, publisher no implementado hasta que `warehouse-service` y
  `shipment-service` existan como código): `warehouse.outbound.released`, mismo envelope de
  `docs/phases/01-architecture.md` sección 4:
  ```
  {
    "workspace_id": "uuid",
    "event_type": "warehouse.outbound.released",
    "payload": {
      "outbound_order_id": "uuid",
      "facility_id": "uuid",
      "customer_company_id": "uuid",
      "storage_unit_ids": ["uuid", "..."]
    },
    "occurred_at": "ISO-8601",
    "correlation_id": "uuid"
  }
  ```
  Cuando Fase 5 se implemente de verdad, este es el segundo consumidor candidato (junto con
  `quotation.accepted`) para crear un `Shipment` — en este caso, uno de origen `warehouse`
  en vez de `sales`.
- **Lee** `Company` de Twenty vía API GraphQL con token scoped al workspace, igual que el
  resto de servicios satélite — nunca acceso directo a su base de datos.

## Alcance de esta fase

**Incluido:** modelo de datos completo de `Facility`/`WarehouseReceipt`/`StorageUnit`/
`InboundOrder`/`OutboundOrder`, contrato de evento saliente `warehouse.outbound.released`.

**Explícitamente fuera de esta fase:**
- Cualquier automatización que conecte shipments y warehouse operations automáticamente
  (ver "Integración" arriba) — se deja como campo nullable completable manualmente hasta que
  el negocio confirme la regla.
- Facturación por almacenaje (storage fees, demurrage en almacén) — Accounting (Fase 10,
  checkpoint humano obligatorio).
- Cualquier dato o cálculo de aduana sobre mercancía en almacén (bonded warehouse, in-bond) —
  Customs (Fase 10-11, checkpoint humano obligatorio). Esta fase no asume que los almacenes
  de Cargo One son bonded/in-bond ni modela nada específico de ese régimen sin confirmación
  del negocio.
- Modelo de slotting/ubicación física detallado — `location_in_facility` es texto libre,
  no un sistema de racks/bins estructurado.
- El código del servicio (`services/warehouse-service/` como proyecto NestJS).

## Criterios de aceptación

- [x] Modelo de datos completo definido con las cinco entidades y su relación con `Shipment`
      vía referencias no forzadas (cross-service, sin FK real).
- [x] Contrato de evento `warehouse.outbound.released` documentado como segundo posible
      origen de creación de `Shipment` (junto a `quotation.accepted` de Fase 4).
- [ ] `services/warehouse-service/` creado como proyecto NestJS con las migraciones
      correspondientes, corriendo contra un schema `logistics_{workspace_uuid}` de prueba
      propio (base de datos separada de `shipment-service`, no compartida).
- [ ] Endpoint o listener que consume `warehouse.outbound.released` desde `shipment-service`
      (cuando ambos existan) verificado con un evento de prueba.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando el código exista y esté
      verificado.

## Notas para el agente

- No es checkpoint humano obligatorio, pero al ser un servicio nuevo (no una extensión de
  `shipment-service`) se recomienda la misma cortesía que con Fase 5: que Carlos vea el
  modelo antes de que se escriban migraciones reales, aunque `docs/00-master-index.md` no lo
  marque como obligatorio para esta fase específicamente.
- Bounded context involucrado: **Warehouse**, con relación bidireccional a **Shipment**
  (downstream y upstream, ver "Contexto"). Ninguna tabla de esta fase vive en la base de
  datos de `shipment-service` ni viceversa — toda referencia cruzada es un UUID suelto, sin
  integridad referencial de base de datos, exactamente como con las referencias a Twenty
  (`customer_company_id`) en Fase 5.
- Si una sesión futura decide implementar la automatización que esta fase dejó fuera
  (conectar `shipment.milestone.recorded` a la creación automática de `InboundOrder`), debe
  primero confirmar con el humano la regla de negocio ("¿cuándo un shipment implica
  almacenaje?") — no inferirla del modelo de datos.
