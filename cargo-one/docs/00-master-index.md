# Master Index — Cargo One

Estado vivo del proyecto. Cualquier agente que empiece una sesión nueva debe revisar esta
tabla primero para saber qué está cerrado, qué está en curso, y qué depende de qué.

**Regla de secuenciación:** no empieces una fase marcada "Bloqueada" sin resolver primero
su dependencia. Ver columna "Depende de".

| # | Fase | Estado | Depende de | Agente puede trabajar solo | Notas |
|---|------|--------|------------|------------------------------|-------|
| 1 | Arquitectura, Twenty Analysis, DDD, Multi-tenant, Legal | 🟢 Cerrada | — | Sí (documentación) | Ver `docs/phases/01-architecture.md`. ADR-002 resuelto. Desbloquea Fases 2-5 |
| 2 | CRM | 🟡 En curso | Fase 1 | Sí | Spec lista, app `sales-extensions-app` escafoldada. Falta `yarn install` + Docker — ver ADR-003 |
| 3 | Customers | 🟡 En curso | Fase 1, 2 | Sí | Spec lista en `docs/phases/03-customers.md`. Mismo bloqueante que Fase 2 |
| 4 | Sales | 🟡 En curso | Fase 1, 2, 3 | Sí | Spec lista, app `quotation-app` escafoldada. Mismo bloqueante que Fase 2/3 |
| 5 | Shipment | 🟡 En curso | Fase 1 | Sí (con revisión de modelo de datos) | Modelo de datos listo en `docs/phases/05-shipment.md`; `services/shipment-service` sin código todavía |
| 6 | Ocean | 🟡 En curso | Fase 5 | Sí | Modelo listo en `docs/phases/06-ocean.md`, extiende Fase 5 sin tocarla |
| 7 | Air | 🟡 En curso | Fase 5 | Sí | Modelo listo en `docs/phases/07-air.md`, mismo patrón que Fase 6 |
| 8 | Ground | 🟡 En curso | Fase 5 | Sí | Modelo listo en `docs/phases/08-ground.md` (sin tabla de `CargoUnit` — no hace falta) |
| 9 | Warehouse | 🟡 En curso | Fase 5 | Sí | Modelo listo en `docs/phases/09-warehouse.md`; servicio propio (`warehouse-service`), no extiende `shipment-service` |
| 10 | Accounting | 🟡 En curso | Fase 1, 5 | Sí (checkpoint levantado, ADR-004) | Modelo y reglas decididos en `docs/phases/10-accounting.md`; cálculo de impuestos real sigue sin definir (no es checkpoint, es dato jurisdiccional) |
| 11 | Documents, Customs e Integrations Hub | 🟡 En curso | Fase 1, 5 | Sí, salvo transmisión real a aduana | Modelo de los tres contextos listo (Customs se agregó en esta ronda — ver `docs/phases/11-documents.md`), conectores de aduana fundamentados en fuentes oficiales — transmisión real sigue gateada por ley, no por este proyecto |
| 12 | AI Platform | 🟡 En curso | Fase 1, 11 | Sí | Modelo listo, incluye `AIProviderConfig` (configuración de credenciales de IA por tenant, reutiliza `Connector` de Fase 11) y casos de uso concretos — ver `docs/phases/12-ai-platform.md` |
| 13 | Security | 🟡 En curso | Fase 1 | Sí (checkpoint levantado, ADR-004) | Decisiones tomadas en `docs/phases/13-security.md`; `.claude/settings.json` corregido en la misma sesión |
| 14 | Deployment | 🟡 En curso | Fase 1 | Parcial | VPS de desarrollo real levantado (Hostinger, Twenty+Postgres+Redis+Traefik verificados sanos) — ver `docs/phases/14-deployment.md`; producción sigue bloqueada por ADR-001 |
| 15 | Testing | 🟡 En curso | Todas las anteriores relevantes | Sí | Estrategia por capa en `docs/phases/15-testing.md`; se aplica junto a cada fase de dominio, no al final |
| 16 | Roadmap | 🟡 En curso | Todas | Sí (documentación) | Snapshot y orden de implementación en `docs/phases/16-roadmap.md` |

Leyenda: 🔲 No iniciada · 🟡 En curso · 🟢 Cerrada · ⛔ Bloqueada (esperando dependencia externa)

## Dependencias externas al proyecto (no resolubles por un agente)

Estas dependen de terceros/trámites, no de código. Vale la pena arrancarlas en paralelo
cuanto antes porque tienen tiempos de calendario largos:

- **Licencia comercial con Twenty.com** — necesaria antes de cualquier despliegue a clientes
  reales (bloquea Fase 14 en producción, no bloquea desarrollo/documentación).
- **Registro como Trade Chain Partner + proveedor EDI ante CBSA** (para CARM) — proceso de
  aprobación gubernamental, semanas/meses. Bloquea el conector real de `integrations/carm-cbsa`
  en Fase 11, no bloquea el resto del proyecto.
- **Registro equivalente ante CBP (ACE)** para el lado USA, por cada tenant que opere del lado
  estadounidense (empezando por Sealion Cargo, no exclusivo de ella — ver ADR-005).

## Próximo paso

**Las 16 fases tienen su work order/modelo especificado y decidido** (checkpoint humano de
Accounting y Security levantado explícitamente por Carlos vía
`docs/decisions/004-human-checkpoint-waiver.md`; conectores de aduana investigados y
fundamentados en fuentes oficiales en vez de bloqueados — ver `docs/phases/11-documents.md`).
Ninguna fase tiene código de servicio implementado todavía. Para la narrativa completa, ver
`docs/phases/16-roadmap.md`.

Bloqueos reales que quedan (ya no son de "falta revisión humana", son de infraestructura o de
requisitos legales externos que ningún ADR de este repo puede levantar):

1. **Infraestructura de desarrollo** (`core/twenty-apps/`): falta `yarn install` + Docker en
   un entorno con red completa. Carlos indicó que no lo hará en su máquina local — pendiente
   definir dónde (ej. un VPS de Hostinger u otro proveedor); requiere que una sesión de agente
   tenga acceso (SSH u otro medio) a ese entorno para completarlo. Bloquea pasar de spec a
   código en Fases 2-4 y la porción genérica de Fase 11.
2. **Revisión humana del core domain** (Fases 5-9, `Shipment` + Ocean/Air/Ground/Warehouse):
   sigue teniendo su propia recomendación de revisión (no un checkpoint de `CLAUDE.md`, sino
   buena práctica dado que seis fases dependen de este modelo) — ya tiene una autorevisión de
   agente aplicada (`docs/phases/05-shipment.md`).
3. **Requisito legal externo, no checkpoint de proyecto** (Fase 11, conectores de aduana):
   transmitir datos reales a CBSA/CARM o ACE/CBP requiere un customs broker licenciado con
   delegación de autoridad — cada tenant configura el suyo en su propio `OrganizationProfile`
   (Fase 2, ADR-005); Sealion Cargo, como primer tenant, también necesita completar el suyo
   antes de poder transmitir.

Pendiente en paralelo (no bloquea desarrollo/documentación): ADR-001 (licencia comercial con
Twenty.com) y el registro externo ante CBSA/CBP — ambos son trámites de terceros con tiempos
de calendario largos, vale la pena arrancarlos ya.

**Gap resuelto:** Customs (`CustomsDeclaration`, `CustomsDeclarationLine`, `DutyAssessment`,
`ComplianceDocument`) ya tiene modelo de datos completo, agregado a la Fase 11 (renombrada
"Documents, Customs e Integrations Hub") en vez de crear una fase numerada nueva — evita
renumerar Fases 12-16 y sus referencias cruzadas. Detalle y razonamiento en
`docs/phases/11-documents.md`.

**Aclaración de producto (ADR-005):** Cargo One es multi-tenant desde el día uno — Sealion
Cargo es el primer tenant, no el único cliente posible. Se agregó `OrganizationProfile`
(Fase 2) para que cualquier freight forwarder que contrate el servicio configure los datos de
su propia empresa (moneda/INCOTERMS por defecto, broker de aduana propio, branding) sin tocar
código. Ver `docs/decisions/005-multi-tenant-product-clarification.md`.

**Producto: UX intuitivo + IA configurable por tenant.** Carlos pidió que el producto sea
user-friendly y que la IA sea funcionalidad central, no accesorio, con un lugar donde cada
tenant configure sus propias credenciales de proveedor de IA (Claude, ChatGPT, u otro). Se
agregó `AIProviderConfig` en Fase 12, reutilizando `Connector`/`ConnectorCredential` de
Fase 11 en vez de un mecanismo de credenciales nuevo — la pantalla de configuración vive
dentro de `integrations-hub-app`. El principio de UX queda documentado en `CLAUDE.md` y en
Fase 12 para aplicarse a cualquier pantalla futura del proyecto, no solo esa.
