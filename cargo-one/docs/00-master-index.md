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
| 5 | Shipment | 🔲 No iniciada | Fase 1 | Sí (con revisión de modelo de datos) | Núcleo del dominio logístico |
| 6 | Ocean | 🔲 No iniciada | Fase 5 | Sí | |
| 7 | Air | 🔲 No iniciada | Fase 5 | Sí | |
| 8 | Ground | 🔲 No iniciada | Fase 5 | Sí | |
| 9 | Warehouse | 🔲 No iniciada | Fase 5 | Sí | |
| 10 | Accounting | 🔲 No iniciada | Fase 1, 5 | **No — checkpoint humano obligatorio** | Dinero real |
| 11 | Documents (incluye Integrations Hub) | 🔲 No iniciada | Fase 1 | Parcial — checkpoint en conectores de aduana | CARM/ACE requieren registro externo primero |
| 12 | AI Platform | 🔲 No iniciada | Fase 1, 11 | Sí | |
| 13 | Security | 🔲 No iniciada | Fase 1 | **No — checkpoint humano obligatorio** | |
| 14 | Deployment | 🔲 No iniciada | Fase 1 | Parcial | |
| 15 | Testing | 🔲 No iniciada | Todas las anteriores relevantes | Sí | |
| 16 | Roadmap | 🔲 No iniciada | Todas | Sí (documentación) | |

Leyenda: 🔲 No iniciada · 🟡 En curso · 🟢 Cerrada · ⛔ Bloqueada (esperando dependencia externa)

## Dependencias externas al proyecto (no resolubles por un agente)

Estas dependen de terceros/trámites, no de código. Vale la pena arrancarlas en paralelo
cuanto antes porque tienen tiempos de calendario largos:

- **Licencia comercial con Twenty.com** — necesaria antes de cualquier despliegue a clientes
  reales (bloquea Fase 14 en producción, no bloquea desarrollo/documentación).
- **Registro como Trade Chain Partner + proveedor EDI ante CBSA** (para CARM) — proceso de
  aprobación gubernamental, semanas/meses. Bloquea el conector real de `integrations/carm-cbsa`
  en Fase 11, no bloquea el resto del proyecto.
- **Registro equivalente ante CBP (ACE)** para el lado USA, si aplica al alcance de Sealion Cargo.

## Próximo paso

Fase 1 cerrada. `core/twenty-apps/` resuelto estructuralmente (ver
`docs/decisions/003-twenty-apps-scaffolding.md`): las tres apps (`sales-extensions-app`,
`quotation-app`, `integrations-hub-app`) están escafoldadas con `create-twenty-app`, sin
`core/twenty/` clonado (no hace falta — ver ADR-003).

Fases 2 (CRM) y 3 (Customers) quedan en 🟡 En curso: falta `yarn install` (bloqueado en este
sandbox por red — corepack no puede descargar `yarn@4.13.0`) y una instancia de Twenty
corriendo vía Docker (el daemon no está disponible en este sandbox) para poder crear los
campos/objetos custom reales y cerrar los criterios de aceptación de cada fase. Ver
`cargo-one/core/twenty-apps/README.md` para los comandos exactos que debe correr Carlos en
una máquina con Docker y red completa.

Fase 4 (Sales) también tiene su work order completo (`docs/phases/04-sales.md`): define
`Quotation`/`QuotationLine` sobre `quotation-app`, enlazado a `Opportunity` nativo de Twenty y
reutilizando `Trade Lane`/`incoterm` de Fases 2-3, con el contrato de evento
`quotation.accepted` documentado (no implementado — no hay consumidor hasta Fase 5). Con esto,
las tres porciones del bounded context CRM & Sales (Fases 2, 3, 4) quedan completamente
especificadas y consistentes entre sí, todas en 🟡 por el mismo bloqueante de infraestructura.

Fase 5 (Shipment) puede especificarse en paralelo — solo depende de Fase 1, cuya spec ya
existe — y es la siguiente candidata natural: es el núcleo del dominio logístico y el
consumidor real del evento `quotation.accepted` que Fase 4 dejó documentado.

Pendiente en paralelo (no bloquea desarrollo): ADR-001 (licencia comercial con Twenty.com)
sigue sin resolución humana — solo bloquea Fase 14 en producción con clientes reales.
