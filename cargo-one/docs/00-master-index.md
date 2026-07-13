# Master Index — Cargo One

Estado vivo del proyecto. Cualquier agente que empiece una sesión nueva debe revisar esta
tabla primero para saber qué está cerrado, qué está en curso, y qué depende de qué.

**Regla de secuenciación:** no empieces una fase marcada "Bloqueada" sin resolver primero
su dependencia. Ver columna "Depende de".

| # | Fase | Estado | Depende de | Agente puede trabajar solo | Notas |
|---|------|--------|------------|------------------------------|-------|
| 1 | Arquitectura, Twenty Analysis, DDD, Multi-tenant, Legal | 🔲 No iniciada | — | Sí (documentación) | Bloquea TODO lo demás |
| 2 | CRM | 🔲 No iniciada | Fase 1 | Sí | Extiende objetos nativos de Twenty |
| 3 | Customers | 🔲 No iniciada | Fase 1, 2 | Sí | |
| 4 | Sales | 🔲 No iniciada | Fase 1, 2, 3 | Sí | |
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

Fase 1 no iniciada. Es la única fase sin bloqueos — puede arrancar de inmediato.
