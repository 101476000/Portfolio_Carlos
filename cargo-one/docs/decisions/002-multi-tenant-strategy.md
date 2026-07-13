# ADR 002 — Estrategia Multi-Tenant

**Estado:** 🔲 Pendiente — se resuelve como parte del contenido de Fase 1.

## Contexto

Ver `docs/phases/01-architecture.md` (a generar en Fase 1) para el análisis completo de
las tres opciones evaluadas:

- **Opción A** — PostgreSQL Row Level Security
- **Opción B** — Schema per Tenant (nota: Twenty ya usa este patrón nativamente para sus
  propios workspaces — `workspace_{uuid}` — lo cual es un dato relevante para decidir si el
  dominio logístico sigue el mismo patrón o uno distinto)
- **Opción C** — Database per Tenant

## Decisión

A completar en Fase 1. Este archivo es el placeholder donde esa decisión debe quedar
registrada formalmente una vez tomada, junto con el razonamiento (ventajas, desventajas,
costo operativo, estrategia de backup y migración).

## Regla para agentes

Ninguna fase de dominio logístico (5 en adelante) debe empezar a definir modelos de datos
sin que este ADR esté 🟢 Resuelto — el modelo de tenancy afecta el diseño de cada tabla.
