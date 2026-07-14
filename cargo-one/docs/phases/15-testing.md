# Fase 15 — Testing

**Estado:** 🟡 En curso — estrategia propuesta. No hay código de servicio todavía sobre el
cual escribir tests reales.

## Contexto

Depende de: todas las fases anteriores relevantes (`docs/00-master-index.md`) — en la
práctica, esta fase madura junto con cada servicio, no de una sola vez al final. Sin bounded
context propio: es transversal, igual que Security y Deployment.

## Estrategia propuesta

| Capa | Qué se prueba | Herramienta | Notas |
|---|---|---|---|
| Apps de Twenty (`sales-extensions-app`, `quotation-app`, `integrations-hub-app`) | Lógica de la app, config de objetos custom | `vitest` (ya configurado por `create-twenty-app` — verificado en el scaffold real de Fase 2-4/11) | Sigue el patrón que Twenty ya trae, no se reinventa |
| Servicios satélite (NestJS) | Unit tests de lógica de dominio, tests de integración contra Postgres de prueba | Jest (estándar NestJS, consistente con `CLAUDE.md`) | Cada servicio corre sus tests contra su propio schema `logistics_{workspace_uuid}` de prueba, aislado del de desarrollo |
| Contratos de eventos | Que el payload emitido por un productor (ej. `quotation.accepted`, Fase 4) coincida con lo que el consumidor espera (Fase 5) | Tests de contrato simples (schema validation del envelope de `docs/phases/01-architecture.md` sección 4) | Importante porque productor y consumidor están en servicios/repos distintos y pueden evolucionar en sesiones de agente diferentes sin memoria compartida |
| Aislamiento multi-tenant | Que un tenant nunca pueda leer/escribir datos de otro | Tests de integración específicos, coordinados con Fase 13 (Security) | No es un "nice to have" — es la verificación de que ADR-002 se cumple en la práctica |
| Flujos con checkpoint humano (Accounting, Customs) | Tests técnicos sí (que el código haga lo que dice el código), pero **no** tests que asuman que una regla de negocio no revisada es correcta | — | Un test verde sobre una fórmula de impuestos no confirmada no es evidencia de que la fórmula esté bien — sigue haciendo falta la revisión humana de Fase 10/11 |
| End-to-end | Flujo completo Quotation → Shipment → (eventualmente) Invoice, contra un entorno de desarrollo completo levantado con la topología de Fase 14 | Por definir la herramienta (Playwright ya está preinstalado en algunos entornos de agente, per las notas del entorno — evaluar cuando haya UI real que probar) | No se implementa hasta que haya suficientes servicios reales para que un e2e tenga sentido |

## Explícitamente fuera de esta fase

- Tests reales — no hay código de servicio contra el cual escribirlos todavía (ver estado de
  cada fase en `docs/00-master-index.md`).
- Cobertura mínima obligatoria por número — definir un porcentaje arbitrario sin código real
  que lo justifique es prematuro.
- Testing de carga/performance — prematuro sin un sistema real corriendo ni datos de uso
  esperado.

## Criterios de aceptación

- [x] Estrategia por capa definida, alineada con las herramientas que cada parte del stack ya
      trae (vitest para Apps de Twenty, Jest para NestJS).
- [ ] Cada fase de dominio (2-12), al implementarse, incluye sus propios tests siguiendo esta
      estrategia — no se centraliza todo el testing al final del proyecto.
- [ ] Tests de contrato de eventos implementados a medida que existan productor y consumidor
      reales de cada evento documentado (`quotation.accepted`, `warehouse.outbound.released`,
      etc.).
- [ ] Tests de aislamiento multi-tenant implementados junto con el primer servicio satélite
      que tenga datos reales que aislar.
- [ ] `docs/00-master-index.md` actualizado a 🟢 Cerrada cuando la estrategia esté siendo
      seguida consistentemente por los servicios existentes, no como un hito único.

## Notas para el agente

- Esta fase es más una **convención a seguir** que un entregable único — cada sesión que
  implemente código de una fase de dominio debe escribir sus tests seguido de esta estrategia,
  no dejarlo para "la fase de Testing" al final.
- Sin bounded context de negocio propio. La única regla dura: ningún test debe encubrir la
  falta de revisión humana en Accounting/Customs haciendo parecer "verificado" algo que solo
  está probado técnicamente, no aprobado por el negocio.
