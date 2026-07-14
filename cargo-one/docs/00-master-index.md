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
| 10 | Accounting | 🟡 En curso | Fase 1, 5 | **No — checkpoint humano obligatorio** | Propuesta de modelo en `docs/phases/10-accounting.md`, REQUIERE REVISIÓN HUMANA total, nada decidido |
| 11 | Documents (incluye Integrations Hub) | 🟡 En curso | Fase 1 | Parcial — checkpoint en conectores de aduana | Modelo listo en `docs/phases/11-documents.md`; Documents + conectores genéricos sin checkpoint, `carm-cbsa`/`ace-cbp` sí (y requieren registro externo) |
| 12 | AI Platform | 🟡 En curso | Fase 1, 11 | Sí | Modelo listo en `docs/phases/12-ai-platform.md`; fija la regla de que checkpoints de `CLAUDE.md` aplican también a agentes de IA |
| 13 | Security | 🟡 En curso | Fase 1 | **No — checkpoint humano obligatorio** | Checklist de áreas en `docs/phases/13-security.md`, sin decisiones tomadas — REQUIERE REVISIÓN HUMANA total |
| 14 | Deployment | 🟡 En curso | Fase 1 | Parcial | Topología de desarrollo/self-host propuesta en `docs/phases/14-deployment.md`; producción bloqueada por ADR-001 |
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
- **Registro equivalente ante CBP (ACE)** para el lado USA, si aplica al alcance de Sealion Cargo.

## Próximo paso

**Las 16 fases tienen su work order/modelo especificado.** Ninguna tiene código de servicio
implementado todavía. Para la narrativa completa de qué falta y en qué orden conviene
implementarlo, ver `docs/phases/16-roadmap.md` — este documento es la fuente de verdad del
*estado* por fase (tabla de arriba), Fase 16 es la narrativa de *qué sigue*; se actualizan
juntos, no por separado.

Resumen de los tres bloqueos más importantes ahora mismo:

1. **Infraestructura de desarrollo** (`core/twenty-apps/`): falta `yarn install` + Docker en
   una máquina con red completa — ver `cargo-one/core/twenty-apps/README.md`. Bloquea pasar
   de spec a código en Fases 2-4 y la porción genérica de Fase 11.
2. **Revisión humana del core domain** (Fases 5-9, `Shipment` + Ocean/Air/Ground/Warehouse):
   ya tiene una autorevisión de agente aplicada (ver `docs/phases/05-shipment.md`), pero
   sigue pendiente la revisión de Carlos antes de escribir migraciones reales.
3. **Revisión humana total, sin excepción** (Fases 10 y 13 — Accounting y Security): estos
   dos documentos son preguntas y propuestas de forma, no decisiones. No avanzar a código sin
   que Carlos las resuelva explícitamente (documentadas como ADRs nuevas en
   `docs/decisions/`).

Pendiente en paralelo (no bloquea desarrollo/documentación): ADR-001 (licencia comercial con
Twenty.com) y el registro externo ante CBSA/CBP (Fase 11, conectores de aduana) — ambos son
trámites de terceros con tiempos de calendario largos, vale la pena arrancarlos ya.
