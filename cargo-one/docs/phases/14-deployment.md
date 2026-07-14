# Fase 14 — Deployment

**Estado:** 🟡 En curso — topología de desarrollo/self-host **verificada en un VPS real**
(Hostinger KVM 2, plantilla de un clic para Twenty). **Producción con clientes reales sigue
bloqueada** hasta que `docs/decisions/001-twenty-license.md` esté 🟢 Resuelto (`CLAUDE.md`
regla 2) — esto no es negociable ni se puede adelantar con una excepción, y un VPS real
corriendo no cambia esta regla: sigue siendo entorno de desarrollo/pruebas hasta que ADR-001
se resuelva.

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
| Reverse proxy / ingress | **Resuelto y verificado:** Traefik | Confirmado corriendo en el VPS de desarrollo — `network_mode: host`, escucha 80/443, TLS automático vía Let's Encrypt (HTTP challenge), redirect HTTP→HTTPS forzado, `providers.docker.exposedbydefault=false` (nada se expone sin label explícita, default seguro). Un contenedor de Twenty se expone agregando labels `traefik.enable=true` + `traefik.http.routers.<nombre>.rule=Host(...)` — mismo patrón a seguir para exponer servicios satélite cuando corresponda |

**Multi-cliente desde el día uno, no un solo tenant** (ADR-005) — Sealion Cargo es el primer
tenant, no el único. El self-host soporta que cualquier freight forwarder cree su propio
workspace en la misma instancia de Twenty (que ya es multi-workspace nativamente); el diseño
de subdominio-por-tenant para los servicios satélite (si hace falta exponerlos directo, más
allá de que Twenty los llame internamente) queda para cuando existan esos servicios.

## VPS de desarrollo actual (no es el entorno de producción de ADR-001)

Aprovisionado en Hostinger (plan KVM 2 — 2 vCPU/8GB RAM/100GB NVMe), usando su plantilla de
instalación de un clic para Twenty, que resultó estar bien construida — se verificó línea por
línea contra lo esperado:

- `twentycrm/twenty:latest` (v2.21.0 al verificar) — server + worker, cada uno con las
  variables de entorno correctas (`DISABLE_DB_MIGRATIONS=true` solo en el worker, igual que el
  `docker-compose.yml` oficial investigado en ADR-003).
- `postgres:16-alpine` y `redis:7-alpine` — **sin puertos publicados al host**, solo
  accesibles desde la red interna de Docker. Correcto, nada de base de datos expuesta a
  internet.
- Secretos (`APP_SECRET`, contraseña de Postgres) generados aleatoriamente por la plantilla,
  no defaults inseguros — verificado, no son valores como "postgres"/"changeme".
- Dominio real con TLS válido asignado por Hostinger (formato
  `<nombre-instancia>.<servidor>.hstgr.cloud`) — no se documenta el valor exacto acá a
  propósito (evitar publicar el hostname real de un servidor de desarrollo en el repo); vive
  en las notas operativas de Carlos, no en este archivo.
- Healthcheck (`/healthz`) responde 200 — instancia sana.

**Pendiente en este VPS (no de esta fase, es trabajo de Fase 2-4):** Node.js no está instalado
a nivel de host (solo dentro de los contenedores) — hace falta para correr `yarn install` en
`core/twenty-apps/*` y conectar esas apps a esta instancia vía
`yarn twenty remote:add --url <URL de esta instancia>`. Tampoco existe todavía ningún
workspace creado en la instancia — el primer signup será el de Sealion Cargo como primer
tenant (ver ADR-005, `OrganizationProfile` en Fase 2).

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
- [x] Verificada contra un VPS real (Hostinger KVM 2) — Twenty + Postgres + Redis + Traefik
      corriendo y saludable, secretos no-default, DB/Redis sin exponer al exterior.
- [ ] `docker-compose.yml`/`docker-compose.override.yml` de los servicios satélite (todavía no
      existen como código) agregado a este mismo VPS o a `infra/docker/` cuando se implementen.
- [ ] `docs/decisions/001-twenty-license.md` resuelto (🟢) — condición para que esta fase
      pueda considerarse cerrada en el sentido de "listo para producción". Sin esto, esta
      fase puede quedar 🟢 solo para desarrollo/self-host interno, nunca para clientes reales.
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
- **Ningún valor real de credenciales de este VPS (IP, contraseñas, secretos generados) se
  documenta en este archivo ni en ningún otro del repo** — se mencionan como "existen y son
  válidos" cuando hace falta, nunca su valor. Esto no es solo la regla general de Fase 13
  (nunca secretos en el repo) — es además la única instancia real y accesible por internet que
  existe del proyecto hasta ahora, así que el costo de una filtración acá es real, no
  hipotético.
