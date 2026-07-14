# Cargo One — Sealion Cargo

Plataforma SaaS **multi-tenant** de freight forwarding construida sobre Twenty CRM,
desarrollada por un developer único orquestando agentes de Claude Code. Sealion Cargo es el
primer tenant/cliente ancla, no el único — cualquier freight forwarder que contrate el
servicio configura los datos de su propia empresa en su propio workspace (ver ADR-005 en
`docs/decisions/`).

## Antes de escribir código

Lee `CLAUDE.md`. En serio, primero eso — es el archivo que cualquier agente lee
automáticamente al abrir este repo, y describe restricciones que no son opcionales
(licencia, checkpoints humanos, qué no tocar de Twenty).

## Cómo usar este repo con Claude Code

1. **Instala Claude Code** si aún no lo tienes: https://docs.claude.com (busca "Claude Code
   installation" — no confíes en instrucciones viejas, la documentación cambia).
2. Clona/abre este repo en Claude Code. `CLAUDE.md` se carga automáticamente como contexto
   de proyecto en cada sesión nueva.
3. Antes de pedirle a un agente que trabaje en una fase, dile explícitamente cuál —
   ejemplo: *"Trabaja en la Fase 5 (Shipment) siguiendo docs/phases/05-shipment.md"*.
   No asumas que el agente "ya sabe" en qué fase estás — siempre referencia el archivo.
4. Revisa `docs/00-master-index.md` regularmente. Es tu tablero de control del proyecto
   completo — actualízalo tú mismo si notas que un agente no lo hizo.
5. Presta atención a las tareas marcadas como "REQUIERE REVISIÓN HUMANA" en el output de
   los agentes — especialmente en customs y accounting. No las mergees sin leerlas de verdad.

## Orden recomendado de trabajo (primeras semanas)

1. Fase 1 completa (arquitectura + Twenty analysis + DDD + multi-tenant + legal) — esto no
   se puede paralelizar, todo lo demás depende de aquí.
2. En paralelo a Fase 1 (son trámites, no código): iniciar conversación de licencia comercial
   con Twenty.com, e iniciar registro como Trade Chain Partner ante CBSA si el timeline lo
   permite — ambos tardan semanas/meses de calendario.
3. Una vez cerrada Fase 1: Fases 2-4 (CRM/Customers/Sales) pueden avanzar mientras en paralelo
   arrancas Fase 5 (Shipment), ya que ambas dependen solo de Fase 1.

## Estructura del proyecto

Ver `PROJECT_STRUCTURE.md`.

## Decisiones de arquitectura

Ver `docs/decisions/`. Cualquier decisión importante nueva debe registrarse ahí como ADR
numerada, no quedar solo en una conversación de chat.
