# Fase 13 — Security

**Estado:** 🟡 En curso — decisiones tomadas (checkpoint humano levantado explícitamente por
Carlos, ver `docs/decisions/004-human-checkpoint-waiver.md`). Sigue sin implementarse código
ni controles reales.

## Contexto

Depende de: Fase 1 (cerrada). Sin bounded context propio — transversal a todos los demás:
autenticación, secretos, aislamiento entre tenants, auditoría.

## Decisiones

### Autenticación núcleo↔satélites

Cada servicio satélite usa un API token de Twenty **scoped al workspace**, no un token
maestro compartido — límite el blast radius de un token comprometido a un solo tenant.
Rotación: manual al inicio (sin volumen que la justifique automatizada todavía), automatizar
cuando haya más de un puñado de tenants activos — se decide en Fase 14 al definir el entorno
de producción real, no aquí en abstracto.

### Gestión de secretos

`ConnectorCredential.credential_reference` (Fase 11) y las credenciales de cada servicio
satélite (`DATABASE_URL`, tokens de Twenty, credenciales de proveedores externos) viven en
variables de entorno inyectadas por el orquestador de despliegue (Docker Compose `env_file`
en desarrollo/self-host; el mecanismo de secretos del proveedor de hosting en producción,
a definir en Fase 14 según dónde se despliegue) — **nunca en el repositorio**, ni siquiera
cifradas. `.gitignore` de cada app/servicio ya excluye `.env*` (verificado en el scaffold real
de `create-twenty-app`, Fase 2-4/11). No se elige un gestor de secretos dedicado (Vault, AWS
Secrets Manager) todavía — es sobre-ingeniería para el volumen actual (developer único, sin
infraestructura corriendo); se reevalúa cuando exista producción real (Fase 14).

### Aislamiento multi-tenant

Se implementa como test de integración obligatorio en cada servicio satélite, no como
auditoría manual periódica: al implementar cualquier servicio (Fase 5 en adelante), sus tests
(Fase 15) deben incluir al menos un caso que verifique que una query con `workspace_id` A
nunca devuelve filas de `workspace_id` B. Se decide como **requisito de Definition of Done**
de cada fase de dominio, no como una fase de auditoría separada al final.

### Cifrado en tránsito/reposo

TLS siempre entre servicios y hacia clientes — no negociable, sin excepción de entorno
(incluye desarrollo si se expone fuera de `localhost`). mTLS entre servicios satélite: no
todavía — se reevalúa si el modelo de amenaza lo justifica una vez haya tráfico real entre
servicios (sobre-ingeniería prematura ahora mismo). Cifrado a nivel de columna: ningún campo
del modelo actual lo necesita más allá de lo que ya cubre `credential_reference` (que ya es
solo una referencia, no el secreto — ver Fase 11), porque no hay todavía PII sensible más allá
de lo que Twenty ya gestiona con sus propios controles nativos.

### Auditoría

Toda acción que en algún momento requirió o requiere criterio de negocio sensible
(cambios de `status` en `Invoice`/`Payment`, aprobación de `ExtractionResult` en Documents,
cualquier transmisión real a CBSA/CARM/ACE/CBP) se loguea con: quién (referencia a
`Person`/usuario de Twenty), qué acción, timestamp, y el estado anterior/nuevo — log
inmutable (append-only, sin UPDATE/DELETE a nivel de aplicación). No se decide todavía la
tecnología de almacenamiento del log (tabla propia vs. servicio de logging externo) — eso es
detalle de implementación de Fase 14.

### Gestión de vulnerabilidades/dependencias

Dependabot (ya disponible sin costo en GitHub, que es donde vive el repo) sobre cada
`package.json` del monorepo, alertas semanales. No se agrega escaneo de contenedores/imágenes
todavía — no hay imágenes propias corriendo aún (Fase 14 sigue sin implementarse).

### Permisos de Claude Code en este repo

`cargo-one/.claude/settings.json` tenía rutas obsoletas (`core/twenty/**`,
`core/twenty/twenty-apps/**`) que ya no existen desde ADR-003 (`core/twenty-apps/**` es la
ruta real, sin el nivel `twenty/` intermedio) — corregido en esta misma sesión. También
reflejaba el checkpoint viejo de Accounting en el bloque `ask`; actualizado para quitar
`services/accounting-service/**` de ahí (ADR-004) y mantener `services/customs-service/**`,
`integrations/carm-cbsa/**`, `integrations/ace-cbp/**` — esos siguen gateados porque el
requisito de broker licenciado es legal, no un checkpoint de este proyecto.

### Respuesta a incidentes (mínimo viable)

Si una credencial se compromete: revocar el token/secreto en el proveedor correspondiente
(Twenty API token vía su panel de workspace; credenciales de conectores externos vía su
propio panel), rotar, y revisar el log de auditoría del período comprometido para identificar
qué se tocó. No se define un proceso más elaborado (on-call, SLA de respuesta) — sería
sobre-ingeniería para un proyecto de developer único sin clientes reales todavía; se
revisita cuando haya producción con clientes (Fase 14 resuelta).

## Explícitamente fuera de esta fase

- Pentesting o auditoría de seguridad externa — sin código real que auditar todavía.
- Elección de gestor de secretos dedicado y mTLS — decisiones pospuestas explícitamente, no
  omitidas por descuido (ver arriba).
- Cualquier control específico de un servicio individual (ej. rate limiting de un conector) —
  se decide en la fase de ese servicio, no aquí de forma centralizada.

## Criterios de aceptación

- [x] Las ocho áreas originalmente listadas como checklist tienen una decisión explícita
      (algunas son "posponer hasta X", documentado como tal, no implícito).
- [x] `cargo-one/.claude/settings.json` corregido (rutas de ADR-003, checkpoint de ADR-004).
- [ ] Tests de aislamiento multi-tenant implementados como parte del Definition of Done de
      cada servicio satélite, a medida que cada uno se codifique (Fase 5 en adelante).
- [ ] TLS y gestión de secretos vía variables de entorno verificados en la topología real de
      Fase 14 cuando se implemente.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada cuando las decisiones de arriba estén
      siendo seguidas consistentemente por los servicios existentes.

## Notas para el agente

- Checkpoint humano levantado por ADR-004 — estas son decisiones activas, no un checklist de
  preguntas. Si una sesión futura no está de acuerdo con alguna, puede cambiarla, pero debe
  documentar el cambio (nueva ADR o edición de esta con justificación), no simplemente
  ignorarla.
- Varias decisiones aquí se postergan deliberadamente (gestor de secretos dedicado, mTLS,
  proceso de incident response elaborado) por ser sobre-ingeniería para el estado actual del
  proyecto (sin código, sin producción, developer único) — no son huecos, son decisiones de
  "todavía no" explícitas y revisadas junto con Fase 14.
