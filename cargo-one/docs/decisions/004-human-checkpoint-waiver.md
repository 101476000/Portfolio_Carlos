# ADR 004 — Alcance revisado de los checkpoints humanos de `CLAUDE.md`

**Estado:** 🟢 Resuelto. Decisión explícita de Carlos Figuera (operador humano del proyecto),
2026-07-14, en sesión de agente.

## Contexto

`CLAUDE.md` (regla 3) definía como "no negociable" que Accounting/Billing/Payments,
Security, y cualquier integración con CBSA/CARM/ACE/CBP requerían revisión humana explícita
antes de que un agente pudiera avanzar más allá de una propuesta. Después de que las Fases 10
(Accounting) y 13 (Security) se entregaran como propuestas sin decisión (ver esos documentos
en su estado previo a esta ADR), Carlos pidió explícitamente:

1. Eliminar el checkpoint humano obligatorio para Accounting y Security — el agente puede
   decidir y avanzar sin pausar a pedir aprobación.
2. Para los conectores de aduana (CBSA/CARM, ACE/CBP): en vez de un checkpoint que bloquea
   todo, el agente debe **investigar las fuentes oficiales** (cbsa-asfc.gc.ca, cbp.gov) y
   fundamentar cualquier decisión en ellas, en vez de inventar reglas o simplemente detenerse.

## Decisión

- **Accounting (Fase 10) y Security (Fase 13):** el checkpoint humano obligatorio de
  `CLAUDE.md` regla 3 queda **levantado** para estas dos fases. El agente puede tomar
  decisiones de diseño razonables (chart of accounts propuesto, gestión de secretos,
  aislamiento multi-tenant, etc.) y avanzarlas sin esperar revisión de Carlos. Esto no elimina
  el buen juicio: sigue sin inventarse una fórmula de impuestos o una regla contable sin
  fundamento, pero la falta de esa fórmula ya no bloquea el resto del modelo — se documenta
  como pendiente de definición de negocio, no como bloqueante de checkpoint.
- **Customs (conectores `carm-cbsa`/`ace-cbp`, dentro de Fase 11):** el checkpoint de
  "detenerse y preguntar a Carlos" se reemplaza por **"investigar las fuentes oficiales y
  fundamentar con cita"** (`CLAUDE.md` regla 4 se mantiene — sigue prohibido inventar reglas
  de compliance — pero la vía para resolverlo ya no es solo "preguntar al humano", es
  "investigar y citar la fuente oficial; si la investigación no resuelve la duda, ahí sí
  preguntar"). La investigación real hecha en esta sesión (ver `docs/phases/11-documents.md`,
  sección de conectores de aduana) es la aplicación concreta de esta regla — no reemplaza una
  revisión legal/de un customs broker licenciado antes de una transmisión real a producción,
  porque **eso no es un checkpoint de este proyecto, es un requisito legal externo** (solo un
  broker licenciado con delegación de autoridad puede presentar un CAD ante CBSA — ver fuente
  citada en Fase 11). Ningún ADR de este repositorio puede levantar ese requisito porque no lo
  impuso este proyecto, lo impone CBSA/CBP.

## Lo que NO cambia

- Regla 1 de `CLAUDE.md` (nunca tocar código fuente de Twenty) — intacta.
- Regla 2 (licencia AGPL/comercial, ADR-001) — intacta, sigue pendiente de resolución humana.
- Regla 4 (no inventar reglas de compliance aduanero) — intacta en espíritu: se reemplaza
  "pregúntale al humano" por "investígalo en la fuente oficial primero", no por "asúmelo".
- El requisito legal externo de que solo un broker licenciado (o un importador autorizado
  auto-declarando) puede presentar ante CBSA/CBP — no es negociable por este proyecto, ver
  Fase 11.

## Regla para agentes

- En Accounting y Security, proponer y decidir sin pausar a pedir revisión — sí dejar
  explícito en cada documento qué se decidió y por qué, para que Carlos pueda objetar después
  si algo no le sirve (revisión posterior, no previa).
- En Customs, antes de fundamentar cualquier regla de negocio, buscar la fuente oficial
  (cbsa-asfc.gc.ca, cbp.gov, o documentación técnica citada por ellos) y citarla en el
  documento. Si la búsqueda no da una respuesta clara o hay señales contradictorias entre
  fuentes, ahí sí se detiene y se pregunta — no se asume la interpretación más conveniente.
- Esta ADR no reabre discusión sobre ADR-001 (licencia Twenty) ni sobre el requisito legal de
  broker licenciado en Customs — esos siguen bloqueados por fuera de lo que este proyecto
  puede decidir.
