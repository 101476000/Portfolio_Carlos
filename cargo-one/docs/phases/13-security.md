# Fase 13 — Security

**Estado:** 🟡 En curso — **propuesta de áreas a cubrir, no plan aprobado. REQUIERE REVISIÓN
HUMANA** en su totalidad (`docs/00-master-index.md` la marca "No — checkpoint humano
obligatorio", sin excepción parcial como Fase 11).

## Contexto

Depende de: Fase 1 (cerrada). A diferencia de las demás fases, Security no tiene un bounded
context propio en `docs/phases/01-architecture.md` — es transversal: toca autenticación,
secretos, aislamiento entre tenants, y auditoría de todos los contextos anteriores. Por eso
este documento no es un modelo de datos sino un **checklist de áreas** que necesitan una
decisión humana antes de que cualquier agente las implemente.

**Por qué no hay modelo de datos ni decisiones aquí:** a diferencia de Accounting (Fase 10),
donde al menos la *forma* de almacenamiento se puede proponer sin decidir la lógica, en
Security incluso proponer una forma (ej. "así se guardan las API keys") ya es una decisión de
seguridad que un agente no debería tomar unilateralmente. Este documento lista preguntas, no
respuestas.

## Áreas a cubrir (cada una requiere decisión humana antes de implementarse)

| Área | Qué hay que decidir | Por qué no se decide en este documento |
|---|---|---|
| Autenticación núcleo↔satélites | ¿API tokens de Twenty por servicio? ¿rotación? ¿scope por workspace o más granular? | Afecta el blast radius de un token comprometido — decisión de riesgo, no técnica |
| Gestión de secretos | Dónde viven `ConnectorCredential.credential_reference` (Fase 11) y las credenciales de cada servicio satélite (Vault, AWS Secrets Manager, variables de entorno cifradas, otro) | Elegir mal esto compromete todos los conectores, incluidos los de aduana |
| Aislamiento multi-tenant | Verificación activa (no solo diseño) de que el patrón schema-per-tenant de ADR-002 realmente aísla — ¿tests automatizados de fuga entre tenants? ¿auditoría periódica? | ADR-002 define el diseño; esta fase debería definir cómo se **verifica** que el diseño se respeta en la práctica |
| Cifrado en tránsito/reposo | TLS entre servicios (¿siempre? ¿mTLS?), cifrado de campos sensibles en base de datos (ej. ¿algo más que `credential_reference` necesita cifrado a nivel de columna?) | Depende de dónde y cómo se despliega (Fase 14), que tampoco está decidido |
| Auditoría | Qué acciones quedan logueadas de forma inmutable — especialmente las que ya son checkpoint humano (Accounting, Customs) necesitan rastro de quién aprobó qué | Sin esto, "REQUIERE REVISIÓN HUMANA" en otras fases no es verificable después de los hechos |
| Gestión de vulnerabilidades/dependencias | ¿Escaneo automático de dependencias (npm audit, Dependabot o similar)? ¿con qué frecuencia? | Decisión operativa que requiere que Carlos defina cuánto tiempo puede dedicarle, siendo developer único |
| Permisos de Claude Code en este repo | `cargo-one/.claude/settings.json` ya existe en el scaffold — revisar qué permisos tiene configurados y si son apropiados ahora que hay 12 fases especificadas y (eventualmente) servicios reales corriendo | Relevante porque el propio flujo de trabajo del proyecto depende de agentes con acceso al repo |
| Respuesta a incidentes | Aunque sea mínimo (¿a quién se notifica? ¿cómo se revoca un token comprometido?), Sealion Cargo maneja datos de aduana y facturación de clientes reales | No se puede improvisar en el momento del incidente |

## Explícitamente fuera de esta fase

- Cualquier decisión concreta de las listadas arriba — este documento pregunta, no responde.
- Implementación de cualquier control de seguridad sin que su decisión correspondiente haya
  sido tomada por Carlos.
- Pentesting o auditoría de seguridad externa — se puede planear una vez haya código real que
  auditar, no sobre un proyecto que todavía es solo especificación.

## Criterios de aceptación

- [ ] **Revisión humana de cada área de la tabla**, con una decisión explícita (aunque sea
      "esto se pospone hasta X") para cada una — ninguna puede quedar implícita.
- [ ] Las decisiones tomadas se documentan como ADRs numeradas en `docs/decisions/` (no en
      este archivo — este archivo es el checklist, no el registro de decisiones).
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada solo cuando **todas** las áreas
      tengan una decisión humana documentada, no cuando existan controles implementados sin
      esa decisión previa.

## Notas para el agente

- **REQUIERE REVISIÓN HUMANA total** — a diferencia de Fase 11 (parcial) o incluso Fase 10
  (donde al menos la forma de almacenamiento se pudo proponer), aquí ni siquiera se propone
  una forma. Cualquier sesión que sienta la tentación de "solo dejar un default razonable"
  para alguna de estas áreas debe resistirla y preguntar en su lugar.
- No hay bounded context de negocio involucrado — esto es transversal a todos. Por eso el
  criterio "confirma que no estás duplicando lógica que ya vive en otro contexto"
  (`CLAUDE.md`, sección "Antes de empezar cualquier tarea") no aplica de la misma forma: acá
  el riesgo no es duplicar lógica, es tomar una decisión de seguridad sin autoridad para
  hacerlo.
- Cuando Carlos resuelva estas áreas, lo natural es que cada decisión se convierta en su
  propia ADR (`docs/decisions/00X-*.md`) — este documento debería terminar siendo un índice
  de referencias a esas ADRs, no crecer indefinidamente él mismo.
