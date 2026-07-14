# Fase 14 — Deployment

**Estado:** 🟡 En curso — topología de desarrollo/self-host propuesta. **Producción con
clientes reales bloqueada** hasta que `docs/decisions/001-twenty-license.md` esté 🟢 Resuelto
(`CLAUDE.md` regla 2) — esto no es negociable ni se puede adelantar con una excepción.

## Contexto

Depende de: Fase 1 (cerrada). Sin bounded context propio — es infraestructura transversal.
Cubre cómo se levanta el sistema completo (Twenty + servicios satélite), no solo un servicio
individual.

**Separación importante que esta fase mantiene:** "se puede desarrollar/probar localmente" y
"se puede desplegar a un cliente real" son cosas distintas. Todo lo de este documento es sobre
lo primero. Lo segundo depende de ADR-001, que es una decisión legal/comercial, no técnica —
ningún contenido de esta fase acelera esa decisión.

## Topología propuesta (desarrollo/self-host, no producción)

| Componente | Cómo se levanta | Notas |
|---|---|---|
| Twenty (núcleo) | Imagen Docker publicada `twentycrm/twenty` + Postgres 16 + Redis, según `packages/twenty-docker/docker-compose.yml` del propio repo de Twenty (verificado contra la v2.9.0 real durante la sesión que resolvió `core/twenty-apps/` — ver ADR-003) | No se vendorea ese archivo en este repo; se referencia la versión pinneada cuando haga falta |
| `sales-extensions-app`, `quotation-app`, `integrations-hub-app` | `yarn twenty dev` (Apps framework), apuntando al Twenty local o self-hosted | Requieren Docker corriendo — ver `core/twenty-apps/README.md` para el bloqueo actual |
| `shipment-service`, `warehouse-service`, `accounting-service`, `customs-service` (dentro de `document-pipeline-service`/Documents), `ai-gateway-service` | NestJS, cada uno con su propia base Postgres (`logistics_{workspace_uuid}` por tenant, ADR-002) | **No comparten base de datos entre sí ni con Twenty** — ver regla de dependencia en `PROJECT_STRUCTURE.md` |
| Redis compartido | Un solo Redis para BullMQ entre los servicios satélite (no el mismo Redis que usa Twenty internamente, para no acoplar su disponibilidad) | Decisión propuesta, no verificada contra una necesidad real de throughput todavía |
| Reverse proxy / ingress | No decidido en detalle, pero el supuesto base ya está resuelto | **Multi-cliente desde el día uno, no un solo tenant** (ADR-005) — Sealion Cargo es el primer tenant, no el único. El self-host debe soportar que cualquier freight forwarder cree su propio workspace en la misma instancia de Twenty (que ya es multi-workspace nativamente) y que cada servicio satélite resuelva `workspace_id` en cada request — el diseño concreto del proxy/ingress (subdominio por tenant vs. path-based, TLS por tenant si aplica) queda para cuando se implemente esta fase |

## Explícitamente fuera de esta fase

- Cualquier despliegue a un cliente real o entorno público — bloqueado por ADR-001, sin
  excepción.
- Kubernetes/orquestación más allá de Docker Compose — `PROJECT_STRUCTURE.md` ya marca
  `infra/k8s/` como "si aplica más adelante", no se adelanta aquí.
- CI/CD real (pipelines ejecutándose) — se propone forma, no se implementa.
- Elección de proveedor de hosting/cloud — decisión de negocio (costo, dónde están los
  clientes, requisitos de residencia de datos) que no le corresponde a un agente.
- Backup/disaster recovery en producción — depende de que exista producción, que depende de
  ADR-001.

## Criterios de aceptación

- [x] Topología de desarrollo/self-host propuesta y justificada.
- [ ] `docker-compose.yml`/`docker-compose.override.yml` real en `infra/docker/` (o donde se
      decida) implementando la topología, verificado levantando el sistema completo en un
      entorno con Docker.
- [ ] `docs/decisions/001-twenty-license.md` resuelto (🟢) — condición para que esta fase
      pueda considerarse cerrada en el sentido de "listo para producción". Sin esto, esta
      fase puede quedar 🟢 solo para desarrollo/self-host interno, nunca para clientes reales.
- [ ] `docs/00-master-index.md` actualizado reflejando ambos estados por separado
      (desarrollo vs. producción) si corresponde, no un solo estado que oculte la distinción.

## Notas para el agente

- El bloqueo de ADR-001 **no es una tecnicalidad que se pueda posponer con un flag de
  configuración** — `CLAUDE.md` regla 2 es explícita: "ningún agente debe asumir que el
  código puede desplegarse públicamente a clientes reales" hasta que esté resuelto. Cualquier
  sesión que quiera avanzar deployment "solo para probar con un cliente piloto" debe
  preguntar al humano primero, no interpretar que un piloto no cuenta como "cliente real".
- No hay bounded context de negocio — esta fase es sobre cómo correr el sistema, no sobre qué
  hace el sistema.
- Depende indirectamente de Fase 13 (Security) para varias decisiones (gestión de secretos en
  el entorno de despliegue, TLS entre servicios) — no tiene sentido cerrar esta fase para
  producción sin que esas decisiones también estén tomadas.
