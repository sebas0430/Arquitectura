# QUICKPATCH

Plataforma multi-tenant de servicios técnicos para hogares y empresas.

## Stack vigente

| Área | Tecnología |
|---|---|
| Admin web | Angular + TypeScript |
| Mobile cliente/técnico | Flutter + Dart |
| Backend principal | ASP.NET Core / .NET |
| Matching | Java + Spring Boot |
| Eventos | Apache Kafka |
| Persistencia | PostgreSQL + PostGIS |
| Cache | Redis |
| Evidencias | MinIO |
| Gateway | Nginx |
| Orquestación backend | k3s |
| Aprovisionamiento | Ansible |

La decisión está formalizada en `docs/architecture/adr/ADR-012-stack-tecnologico-polyglot.md`.

## Monorepo

```text
apps/
├── web/
├── mobile/
└── backend/
    └── services/
        ├── identity/
        ├── actors/
        ├── catalog/
        ├── service-request/
        ├── matching/
        ├── ranking/
        ├── payments/
        └── communication/

tests/
├── integration/
├── contract/
├── e2e/
├── performance/
└── security/

infrastructure/
├── ansible/
├── vm1-gateway/
├── vm2-web/
├── vm3-k3s/
├── vm4-database/
├── vm5-redis/
├── vm6-kafka/
└── vm7-storage-observability/
```

## Documentación

- `docs/requirements/SRS.md`
- `docs/architecture/SAD.md`
- `docs/architecture/SDD.md`
- `docs/design/DD.md`
- `docs/infrastructure/INFRASTRUCTURE.md`
- `docs/governance/WORKING_AGREEMENTS.md`

## IA

- Codex/agentes: `AGENTS.md`
- Claude Code: `CLAUDE.md`
- Contexto: `.ai/`

## Contratos

- REST: `docs/contracts/openapi/`
- Kafka: `docs/contracts/events/`

## GitFlow

Trabajo normal:

`feature/* -> develop -> release/* -> main`

Los PR de feature deben apuntar a `develop`.
