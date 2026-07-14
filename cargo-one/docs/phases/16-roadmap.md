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

**Actualización importante (misma ronda, después del snapshot inicial):** Carlos levantó
explícitamente el checkpoint humano de Accounting y Security
(`docs/decisions/004-human-checkpoint-waiver.md`), y pidió que los conectores de aduana se
fundamenten en investigación de fuentes oficiales en vez de bloquearse en seco. Eso cambió el
mapa de riesgo de abajo respecto a la primera versión de este documento.

**Bloqueadas solo por infraestructura, sin checkpoint (Fases 2-9):** CRM, Customers, Sales
(bounded context CRM & Sales completo) y Shipment + Ocean + Air + Ground + Warehouse (core
domain completo). Bloqueante común: `yarn install`/Docker no disponibles en el sandbox donde
se especificaron — ver `core/twenty-apps/README.md`. Carlos indicó que no va a resolverlo en
su máquina local; pendiente definir un entorno alternativo (ej. VPS) con acceso para una
sesión de agente. Fase 5 además tiene una recomendación (no checkpoint) de revisión humana del
modelo de datos por ser el core domain (autorevisión de agente ya aplicada, ver
`docs/phases/05-shipment.md`).

**Diseño decidido, solo bloqueadas por requisito legal externo (Fase 11 — Documents, Customs
e Integrations Hub):** Documents, el modelo de `CustomsDeclaration` (Customs ya no vive solo
implícito en el nombre de la fase — se agregó su modelo completo en esta ronda) y los
conectores genéricos son trabajo normal. Los conectores `carm-cbsa`/`ace-cbp` ya tienen su
diseño fundamentado en fuentes oficiales (CARM/CAD, ACE/CATAIR — ver
`docs/phases/11-documents.md`); lo que falta no es revisión de este proyecto sino (a) el
registro externo ante CBSA/CBP y (b) que Carlos confirme si Sealion Cargo opera con
broker licenciado propio, partner, o planea licenciarse — sin eso no se sabe a quién le habla
realmente el conector.

**Decididas, sin checkpoint (Fases 10, 13):** Accounting y Security tienen modelo y decisiones
tomadas (ver esos documentos) — el cálculo real de impuestos en Accounting sigue sin definirse,
pero por falta de una fuente jurisdiccional confiable a investigar, no por checkpoint.

**Bloqueada por decisión legal, no técnica (Fase 14 en producción):** ADR-001 (licencia
comercial Twenty.com) sigue sin resolución humana — este sí sigue siendo un checkpoint real,
no levantado por ADR-004.

**Transversales sin bounded context propio (Fases 13, 14, 15, 16):** Security, Deployment,
Testing, Roadmap — se maduran junto con las demás, no de una sola vez.

## Orden sugerido de trabajo real (no de especificación — de implementación)

1. **Resolver infraestructura de desarrollo** (`yarn install` + Docker para
   `core/twenty-apps/`) en un entorno con acceso real — Carlos decidió no usar su máquina
   local; pendiente aprovisionar y dar acceso a un entorno alternativo (ej. VPS) a una sesión
   de agente. Sin esto, nada de Fases 2-4 ni 11 (porción genérica) puede pasar de spec a
   código.
2. **Revisión humana recomendada de Fase 5-9** (el core domain) antes de escribir migraciones
   reales — ya no es checkpoint obligatorio, pero sigue siendo la fase que más se propaga si
   hay un error de modelado.
3. **Implementar Fases 2-9 en código**, en el orden de dependencia ya establecido
   (2→3→4 en paralelo a 5→6/7/8→9).
4. **En paralelo, iniciar los trámites de las dependencias externas** (licencia Twenty.com,
   registro CBSA/CBP) y confirmar la pregunta de negocio de Fase 11 (broker propio/partner/
   licenciarse) — no bloquean 1-3, pero tienen tiempos de calendario largos.
5. **Implementar Fase 10 (Accounting) y Fase 13 (Security)** cuando convenga — ya no dependen
   de una revisión humana previa (ADR-004), pero Security sigue siendo buena idea resolverla
   antes de la primera implementación de un conector real, porque varias decisiones de
   Accounting/Integrations Hub (gestión de secretos, auditoría) ya están definidas ahí.
6. **Fase 11 (conectores de aduana):** código real solo después de (a) registro externo
   confirmado y (b) la pregunta de negocio resuelta — el diseño en sí ya no espera revisión.
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
