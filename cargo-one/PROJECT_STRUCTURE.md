# Estructura del repositorio — Cargo One

Monorepo con separación clara entre el núcleo (Twenty, no se modifica) y los satélites
(dominio logístico, IA, integraciones — construidos por nosotros).

```
cargo-one/
├── CLAUDE.md                      # Leer primero, siempre
├── PROJECT_STRUCTURE.md           # Este archivo
├── README.md                      # Onboarding humano
│
├── docs/
│   ├── 00-master-index.md         # Estado de las 16 fases
│   ├── decisions/                 # ADRs — decisiones de arquitectura numeradas
│   │   ├── 001-twenty-license.md
│   │   ├── 002-multi-tenant-strategy.md
│   │   └── ...
│   └── phases/                    # Un archivo por fase (work orders para agentes)
│       ├── 01-architecture.md
│       ├── 02-crm.md
│       ├── 03-customers.md
│       └── ... (hasta 16-roadmap.md)
│
├── core/
│   └── twenty-apps/                # Nuestras extensiones vía Apps framework. NO hay un
│                                    # core/twenty/ con el monorepo clonado — ver ADR-003:
│                                    # cada app es un proyecto npm independiente generado
│                                    # con `npx create-twenty-app`, que habla con una
│                                    # instancia de Twenty (local o self-hosted) por API.
│       ├── README.md               # Cómo se generaron, estado, próximos pasos
│       ├── quotation-app/          # Fase 4 (Sales)
│       ├── sales-extensions-app/   # Fases 2-3 (CRM, Customers)
│       └── integrations-hub-app/   # El "admin de tokens" — ver Fase 11
│
├── services/                      # Dominio logístico — NestJS, fuera del core de Twenty
│   ├── shipment-service/
│   ├── customs-service/           # Incluye lógica CBSA/CARM y ACE/CBP (checkpoint humano)
│   ├── warehouse-service/
│   ├── accounting-service/        # Checkpoint humano obligatorio
│   ├── document-pipeline-service/ # Upload → OCR → clasificación → extracción → aprobación
│   └── ai-gateway-service/        # Prompt management, LLM router, embeddings, RAG
│
├── integrations/                  # Conectores del Integrations Hub (patrón "thin connector")
│   ├── shared/
│   │   └── connector-interface.ts # Contrato común: provider, capabilities, mapEvent()
│   ├── amazon-sp-api/
│   ├── wayfair-api/
│   ├── carm-cbsa/                 # Requiere registro previo como Trade Chain Partner — ver ADR
│   └── ace-cbp/
│
├── infra/
│   ├── docker/
│   ├── k8s/                       # Si aplica más adelante (Fase 14)
│   └── ci/
│
└── .claude/
    └── settings.json              # Permisos y configuración de Claude Code para este repo
```

## Reglas de dependencia entre carpetas

- `services/*` y `integrations/*` **nunca** importan código directamente de Twenty ni de sus
  apps. Toda comunicación es vía la API pública de Twenty (REST/GraphQL) o eventos.
- `core/twenty-apps/*` sí puede usar los SDKs oficiales de Twenty (`twenty-sdk`,
  `twenty-client-sdk`) porque corren dentro del framework de extensión soportado
  (`create-twenty-app`). Ver `docs/decisions/003-twenty-apps-scaffolding.md`.
- Cada carpeta bajo `services/` corresponde a uno o más bounded contexts definidos en
  `docs/phases/01-architecture.md` (sección DDD). Si una carpeta empieza a mezclar
  responsabilidades de dos bounded contexts distintos, es señal de que hay que dividirla.
