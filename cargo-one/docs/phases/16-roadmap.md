# Fase 16 — Roadmap

**Estado:** 🟡 En curso — snapshot del estado del proyecto al cierre de esta ronda de
especificación. Se actualiza, no se reescribe desde cero, cada vez que el estado real cambie.

## Contexto

Depende de: todas las fases (documentación). Esta fase no agrega bounded contexts nuevos —
consolida el estado de los 15 anteriores en una sola vista de "qué sigue" para quien retome
el proyecto, humano o agente.

## Estado al cierre de esta ronda

Las 16 fases tienen su work order/modelo de datos especificado. Ninguna tiene código de
servicio implementado todavía. Resumen por nivel de riesgo:

**Sin checkpoint, bloqueadas solo por infraestructura (Fases 2-9):** CRM, Customers, Sales
(bounded context CRM & Sales completo) y Shipment + Ocean + Air + Ground + Warehouse (core
domain completo). Bloqueante común: `yarn install`/Docker no disponibles en el sandbox donde
se especificaron — ver `core/twenty-apps/README.md`. Fase 5 además pide revisión humana del
modelo de datos por ser el core domain (autorevisión de agente ya aplicada, ver
`docs/phases/05-shipment.md`).

**Checkpoint humano parcial (Fase 11):** Documents y conectores genéricos son trabajo normal;
los conectores `carm-cbsa`/`ace-cbp` requieren revisión humana total **y** registro externo
ante CBSA/CBP (dependencia externa, semanas/meses).

**Checkpoint humano total (Fases 10, 13):** Accounting y Security no tienen ni modelo
aprobado — son propuestas explícitamente marcadas como no decididas.

**Bloqueada por decisión legal, no técnica (Fase 14 en producción):** ADR-001 (licencia
comercial Twenty.com) sigue sin resolución humana.

**Transversales sin bounded context propio (Fases 13, 14, 15, 16):** Security, Deployment,
Testing, Roadmap — se maduran junto con las demás, no de una sola vez.

## Orden sugerido de trabajo real (no de especificación — de implementación)

1. **Resolver infraestructura de desarrollo:** `yarn install` + Docker en una máquina de
   Carlos (no en un sandbox de agente) para `core/twenty-apps/`. Sin esto, nada de Fases 2-4
   ni 11 (porción Documents/Integrations Hub genérica) puede pasar de spec a código.
2. **Revisión humana de Fase 5-9** (el core domain) antes de escribir migraciones reales —
   es la que más se propaga si hay un error.
3. **Implementar Fases 2-9 en código**, en el orden de dependencia ya establecido
   (2→3→4 en paralelo a 5→6/7/8→9).
4. **En paralelo, iniciar los trámites de las dependencias externas** (licencia Twenty.com,
   registro CBSA/CBP) — no bloquean 1-3, pero tienen tiempos de calendario largos, vale la
   pena arrancarlos ya.
5. **Resolver Fase 13 (Security)** antes o junto con la primera implementación de un
   conector real (Fase 11) o de Accounting (Fase 10) — varias decisiones de esas dos fases
   dependen de las de Security (gestión de secretos, auditoría).
6. **Fase 10 (Accounting) y Fase 11 (conectores de aduana)** solo después de su revisión
   humana explícita — no antes, sin importar cuánto avance el resto del proyecto.
7. **Fase 12 (AI Platform)** una vez Documents (Fase 11, porción genérica) tenga datos reales
   que indexar — antes de eso, un `EmbeddingIndex` no tiene sobre qué operar.
8. **Fase 14 (Deployment) a producción** solo cuando ADR-001 esté 🟢 — el self-host de
   desarrollo (topología ya propuesta) puede avanzar antes, para producción no.
9. **Fase 15 (Testing)** no es un paso final — se aplica en cada fase de dominio a medida que
   se implementa (ver `docs/phases/15-testing.md`).

## Explícitamente fuera de esta fase

- Fechas de calendario concretas — dependen de cuánto tiempo le dedique Carlos, y de trámites
  externos fuera de control del proyecto. Este documento ordena, no calendariza.
- Priorización de negocio (ej. "Ocean antes que Air porque el primer cliente de Sealion Cargo
  es marítimo") — información que solo tiene Carlos, no se asume.

## Criterios de aceptación

- [x] Snapshot de estado por nivel de riesgo, consistente con `docs/00-master-index.md`.
- [x] Orden de implementación sugerido, distinto del orden de especificación ya completado.
- [ ] Actualizado la próxima vez que una fase cambie de estado de forma significativa (no en
      cada commit menor) — mantenerlo vivo es responsabilidad de quien retome el proyecto,
      igual que `docs/00-master-index.md`.

## Notas para el agente

- Este documento y `docs/00-master-index.md` deben mantenerse consistentes entre sí — si
  divergen, `docs/00-master-index.md` es la fuente de verdad de estado por fase (tabla), y
  este documento es la narrativa de "qué sigue" — actualizar ambos juntos, no uno sin el otro.
- No hay bounded context propio. Esta fase es la única, junto con Fase 1, marcada como
  "documentación" pura en `docs/00-master-index.md` — no se espera código nunca, a diferencia
  de las demás que sí lo tendrán eventualmente.
