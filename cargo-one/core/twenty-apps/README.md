# twenty-apps — Apps de Cargo One sobre Twenty

Estas tres carpetas son las extensiones de Twenty que corresponden a las Fases 2-4 (CRM,
Customers, Sales) y Fase 11 (Documents/Integrations Hub) del proyecto — ver
`docs/00-master-index.md` y `docs/phases/`.

## Cómo se generaron

Cada carpeta es el resultado de `npx create-twenty-app@latest <nombre>` (el scaffolder
oficial de Twenty — ver https://docs.twenty.com/developers/extend/apps/getting-started).
**No** son un fork ni una copia del monorepo de Twenty: son proyectos npm independientes que
usan `twenty-sdk`/`twenty-client-sdk` para hablar con una instancia de Twenty ya corriendo
(local vía Docker, o remota). Ver `docs/decisions/003-twenty-apps-scaffolding.md` para el
razonamiento completo de por qué se descartó clonar el monorepo como submódulo.

| Carpeta | Fase que cubre | Propósito |
|---|---|---|
| `sales-extensions-app/` | Fase 2 (CRM), Fase 3 (Customers) | Campos/objetos custom: `company_role`, `contact_role`, `Trade Lane`, `CustomerLogisticsProfile` — ver `docs/phases/02-crm.md` y `docs/phases/03-customers.md` |
| `quotation-app/` | Fase 4 (Sales) | `Quotation` y lógica de pipeline de ventas — fase aún no especificada |
| `integrations-hub-app/` | Fase 11 (Documents/Integrations Hub) | Objetos `Connector`/`ConnectorCredential` — admin de tokens de conectores externos |

## Estado actual (dejado así intencionalmente)

- Scaffolding completo: estructura de proyecto, `package.json`, `src/`, config de lint/test.
- `yarn install` **no se completó**: `corepack` necesita descargar el binario de
  `yarn@4.13.0` desde un host que el proxy saliente de este entorno de agente no permite
  (`403` al intentar el túnel HTTPS). No es un bug del scaffold — es una restricción de red
  de esta sesión sandbox.
- El servidor local de Twenty **no se inició**: requiere Docker corriendo (`docker info`
  falla en este entorno — el daemon no está disponible aquí), y `create-twenty-app` lo
  detectó y saltó ese paso automáticamente sin fallar el resto del scaffold.
- Ningún objeto/campo custom fue creado todavía en un workspace real de Twenty — eso requiere
  los dos pasos anteriores resueltos primero.

## Próximos pasos (en una máquina con Docker y red completa — no en este sandbox)

Para cada app:

```bash
cd cargo-one/core/twenty-apps/<nombre-de-app>
yarn install                              # requiere red completa (corepack/yarn 4)
yarn twenty dev                            # levanta Twenty local vía Docker y autentica
# o, contra una instancia ya self-hosteada:
yarn twenty remote:add --url <url-del-workspace>
```

A partir de ahí, seguir el work order de la fase correspondiente (`docs/phases/02-crm.md`,
`docs/phases/03-customers.md`, etc.) para definir los objetos/campos custom con
`twenty-sdk`, y marcar los criterios de aceptación de esa fase a medida que se verifiquen
contra la instancia real.
