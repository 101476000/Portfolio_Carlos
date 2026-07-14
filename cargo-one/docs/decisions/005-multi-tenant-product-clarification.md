# ADR 005 — Aclaración de producto: multi-tenant desde el día uno, no solo para Sealion Cargo

**Estado:** 🟢 Resuelto. Aclaración explícita de Carlos Figuera, 2026-07-14.

## Contexto

Fase 14 (Deployment) había dejado como pregunta abierta, sin asumir una respuesta: "¿el
self-host es de un solo tenant (Sealion Cargo) o multi-cliente desde el día uno?". Varias
otras fases usaban lenguaje que, sin ser técnicamente incorrecto (la arquitectura de
ADR-002 ya es schema-per-tenant, genuinamente multi-tenant), sonaba como si Sealion Cargo
fuera el único cliente posible — ej. `CLAUDE.md` decía "para Sealion Cargo", y Fase 11
(Customs) trataba "¿Sealion Cargo opera con broker propio o partner?" como si fuera una
decisión de negocio única para todo el sistema, en vez de una configuración que cada tenant
resuelve para sí mismo.

Carlos confirmó explícitamente: Cargo One **no es software a medida para Sealion Cargo** —
es un producto SaaS que se vende a cualquier freight forwarder. Sealion Cargo es la empresa
de Carlos y el primer tenant/cliente ancla, pero cualquiera que "contrate" el servicio debe
poder crear su propio workspace y configurar los datos de su propia empresa ahí.

## Decisión

1. **La arquitectura no cambia** — ADR-002 (schema-per-tenant) ya estaba diseñada
   correctamente para esto. Lo que cambia es que quedaba sin modelar un pedazo real:
   **cómo un tenant nuevo configura los datos de su propia empresa** (no los datos de sus
   clientes — eso ya lo cubre Fase 3 — sino los datos de la empresa que se registra a usar
   Cargo One). Se agrega `OrganizationProfile` en Fase 2 (CRM) — ver ese documento.
2. **Decisiones que antes se trataban como "una sola respuesta para todo el sistema" pasan a
   ser configuración por tenant:** en particular, `CustomsDeclaration.broker_of_record_company_id`
   (Fase 11) — no es "¿Sealion Cargo tiene broker?", es "cada tenant configura su propio
   arreglo de customs filing en su `OrganizationProfile`, y Sealion Cargo configura el suyo
   como cualquier otro tenant".
3. **Fase 14** resuelve su pregunta abierta: self-host multi-cliente desde el día uno, no
   mono-tenant. La topología de desarrollo/self-host ya propuesta no necesita cambiar (Docker
   Compose con un Twenty compartido y bases de datos schema-per-tenant ya asumía esto), pero
   se documenta como decisión explícita en vez de pregunta pendiente.
4. **Lenguaje corregido** en `CLAUDE.md`, `README.md`, y donde "Sealion Cargo" se usaba de
   forma que sonaba a cliente único — se deja claro que es el primer tenant, no el producto
   completo.

## Lo que NO cambia

- ADR-002 (multi-tenancy schema-per-tenant) — ya estaba bien diseñada para esto, se reafirma.
- El resto del modelo de datos de Fases 2-13 — la mayoría ya era correctamente por-tenant
  (todo lleva `workspace_id`); el gap era específicamente la falta de un objeto de
  configuración de la propia empresa del tenant, no un error de aislamiento.

## Regla para agentes

- Nunca volver a escribir una fase asumiendo que "la empresa" del sistema es Sealion Cargo —
  Sealion Cargo es un tenant como cualquier otro, aunque sea el primero y el que opera Carlos.
- Cualquier dato de configuración específico de la empresa que usa la plataforma (no de sus
  clientes) va en `OrganizationProfile` (Fase 2) o una extensión de ese mismo objeto, nunca
  hardcodeado ni tratado como una decisión de negocio única para todo el sistema.
- Si una fase futura necesita distinguir comportamiento por tenant (ej. feature flags,
  planes de suscripción), ese es un tema nuevo que merece su propia ADR — esta no lo resuelve,
  solo establece que "una empresa por tenant, configurable por su propio usuario" es la base.
