# ADR-012 — Stack tecnológico políglota de QUICKPATCH

- **Estado:** Aceptado
- **Fecha:** 2026-09-23
- **Decisión:** Stack tecnológico por canal y microservicio

## Contexto

QUICKPATCH utiliza una arquitectura de microservicios orientada a eventos (EDA) y debe cumplir dos restricciones académicas obligatorias:

1. incluir al menos un componente implementado en .NET;
2. incluir al menos un módulo implementado en Java.

Adicionalmente, el producto diferencia claramente dos canales de usuario:

- administración mediante aplicación web;
- clientes y técnicos mediante aplicación móvil.

La documentación previa contenía referencias simultáneas a NestJS, .NET, Spring Boot y Next.js, generando ambigüedad para implementación, infraestructura, pruebas y agentes de IA.

## Decisión

Se adopta el siguiente stack vigente:

| Área | Tecnología |
|---|---|
| Panel administrativo web | Angular + TypeScript |
| Aplicación móvil cliente/técnico | Flutter + Dart |
| Backend principal | ASP.NET Core / .NET |
| Matching Service | Java + Spring Boot |
| Integración síncrona | REST/HTTPS |
| Integración asíncrona | Apache Kafka |
| Persistencia | PostgreSQL + PostGIS |
| Acceso a datos .NET | EF Core + Npgsql |
| Acceso a datos Matching | Spring Data/JPA + PostgreSQL/PostGIS |
| Cache | Redis |
| Evidencias/objetos | MinIO |
| Gateway | Nginx |
| Orquestación de microservicios | k3s |
| Aprovisionamiento | Ansible |
| Observabilidad | Prometheus + Loki + Grafana |

## Asignación por microservicio

| Microservicio | Tecnología |
|---|---|
| Identity | ASP.NET Core |
| Actors | ASP.NET Core |
| Catalog | ASP.NET Core |
| ServiceRequest | ASP.NET Core |
| Matching | Java + Spring Boot |
| Ranking | ASP.NET Core |
| Payments | ASP.NET Core |
| Communication | ASP.NET Core |

## Reglas de integración

REST se utiliza cuando el consumidor necesita una respuesta síncrona inmediata.

Kafka se utiliza para hechos de dominio y procesos desacoplados/asíncronos. La adopción de EDA no implica reemplazar toda interacción síncrona por eventos.

Los contratos REST se versionan en `docs/contracts/openapi/`.

Los contratos de eventos se versionan en `docs/contracts/events/`.

La interoperabilidad entre .NET y Java depende de contratos, no de compartir código de dominio.

## Consecuencias

### Positivas

- Se cumplen las restricciones académicas de .NET y Java.
- Se elimina la coexistencia innecesaria de NestJS como tercer stack backend.
- Matching conserva Java/Spring Boot, coherente con el diseño especializado ya documentado.
- Los servicios transaccionales comparten un stack principal .NET.
- Kafka permite integrar servicios políglotas mediante contratos explícitos.
- Angular queda enfocado en el panel administrativo.
- Flutter queda enfocado en clientes y técnicos móviles.

### Costos

- El equipo debe mantener dos toolchains backend: .NET y Java.
- CI/CD debe detectar y ejecutar pipelines distintos para servicios .NET y Matching.
- Observabilidad, serialización de eventos y convenciones deben ser equivalentes en ambos stacks.
- Los contratos deben ser especialmente estrictos para evitar divergencias entre lenguajes.

## Decisiones descartadas

### Todo Spring Boot

Se descartó porque no cumpliría por sí solo la restricción obligatoria de incluir .NET.

### NestJS + Spring Boot + .NET

Se descartó por introducir tres stacks backend sin una necesidad de dominio que justifique la complejidad adicional.

### Flutter Web para administración

Se descartó como stack principal del administrador. El panel administrativo se implementará como aplicación web Angular, mientras Flutter se reserva para cliente/técnico móvil.

## Trazabilidad

Esta decisión debe mantenerse sincronizada con:

- `docs/design/DD.md`
- `docs/architecture/SDD.md`
- `docs/infrastructure/INFRASTRUCTURE.md`
- `docs/governance/WORKING_AGREEMENTS.md`
- `.ai/PROJECT_CONTEXT.md`
- `.ai/ARCHITECTURE_SUMMARY.md`
- `.ai/roles/`
