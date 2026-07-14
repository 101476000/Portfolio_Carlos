# ADR 001 — Estrategia de licenciamiento sobre Twenty CRM

**Estado:** 🔲 Pendiente de resolución humana — NO es una tarea de agente.

## Contexto

Twenty CRM es AGPL-3.0. Cargo One es un producto comercial **multi-tenant** que se ofrecerá
como servicio a terceros — Sealion Cargo es el primer tenant/cliente ancla, no el único
(aclaración explícita de Carlos, ver `docs/decisions/005-multi-tenant-product-clarification.md`).
Bajo AGPL-3.0, esto obliga a publicar como código abierto cualquier modificación usada para
dar servicio por red, salvo que se adquiera una licencia comercial del titular de derechos —
el hecho de que sea multi-tenant (varios freight forwarders, no solo uso interno de Sealion
Cargo) hace este punto más urgente, no menos: es exactamente el escenario de "dar servicio
por red a terceros" que dispara la obligación de AGPL-3.0.

Twenty.com ofrece una excepción de licencia comercial ("twenty-ee") para casos como este —
ver el archivo LICENSE del repo oficial (github.com/twentyhq/twenty/blob/main/LICENSE).

## Decisión pendiente

- [ ] Contactar a Twenty.com (o partner certificado) para negociar licencia comercial/dual.
- [ ] Confirmar por escrito los términos: alcance, costo, renovación, qué partes del código
      quedan cubiertas.
- [ ] Actualizar este ADR con la decisión final y fecha de firma.

## Regla para agentes mientras este ADR esté "Pendiente"

- Se puede desarrollar y documentar con normalidad (Fases 1-13, 15, 16).
- **No se debe considerar el proyecto listo para desplegar a un cliente real** (Fase 14 en
  producción) hasta que este ADR se marque como 🟢 Resuelto con evidencia de la licencia firmada.
- Si una tarea de agente requiere asumir que el despliegue público ya es legal, DETENTE y
  pregunta al humano.
