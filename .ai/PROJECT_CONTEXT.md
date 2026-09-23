# QUICKPATCH — Contexto global para IA

## Propósito

QUICKPATCH es una plataforma multi-tenant de servicios técnicos para hogares y empresas. Conecta clientes con técnicos y gestiona solicitudes, matching, seguimiento, evidencias, pagos, reputación y comunicaciones.

Este archivo resume decisiones vigentes; no reemplaza SRS, SAD, SDD ni DD.

## Stack vigente

- Admin Web: Angular + TypeScript.
- Mobile cliente/técnico: Flutter + Dart.
- Backend principal: ASP.NET Core / .NET.
- Matching Service: Java + Spring Boot.
- Arquitectura backend: 8 microservicios + EDA.
- Integración síncrona: REST/HTTPS.
- Integración asíncrona: Apache Kafka.
- Persistencia: PostgreSQL + PostGIS.
- Acceso a datos .NET: EF Core + Npgsql.
- Matching: Spring Data/JPA.
- Cache: Redis.
- Evidencias/objetos: MinIO.
- Gateway: Nginx.
- Backend en VM3: k3s.
- Aprovisionamiento: Ansible.
- Observabilidad: Prometheus + Loki + Grafana.

Decisión formal: `docs/architecture/adr/ADR-012-stack-tecnologico-polyglot.md`.

## Microservicios

- Identity — ASP.NET Core
- Actors — ASP.NET Core
- Catalog — ASP.NET Core
- ServiceRequest — ASP.NET Core
- Matching — Java + Spring Boot
- Ranking — ASP.NET Core
- Payments — ASP.NET Core
- Communication — ASP.NET Core

## Reglas críticas

1. Multi-tenancy no puede debilitarse.
2. Ningún servicio escribe directamente en tablas propiedad de otro servicio.
3. Backend no almacena PAN/CVV.
4. REST se usa cuando se necesita respuesta síncrona.
5. Kafka se usa para hechos de dominio/procesos desacoplados definidos por arquitectura.
6. Contratos REST y eventos son fronteras versionadas.
7. Evidencias forman parte del diseño vigente.
8. Cambios arquitectónicos requieren trazabilidad/ADR.

## GitFlow

- `main`: estable.
- `develop`: integración.
- `feature/*`: trabajo desde develop.
- `release/*`: preparación de entrega.
- `hotfix/*`: correcciones urgentes desde main.

## Contexto progresivo

1. este archivo;
2. `.ai/ARCHITECTURE_SUMMARY.md`;
3. rol;
4. requisito/ADR/contrato;
5. código de la tarea.

No cargar todo el repositorio por defecto.
