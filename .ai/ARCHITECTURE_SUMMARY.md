# QUICKPATCH — Resumen arquitectónico para agentes

## Fuentes

- SRS: `docs/requirements/SRS.md`
- SAD: `docs/architecture/SAD.md`
- SDD: `docs/architecture/SDD.md`
- DD: `docs/design/DD.md`
- Infraestructura: `docs/infrastructure/INFRASTRUCTURE.md`
- ADR tecnológica: `docs/architecture/adr/ADR-012-stack-tecnologico-polyglot.md`

## Canales

- Admin: Angular Web.
- Cliente/técnico: Flutter Mobile.

## Servicios

| Servicio | Stack |
|---|---|
| Identity | ASP.NET Core |
| Actors | ASP.NET Core |
| Catalog | ASP.NET Core |
| ServiceRequest | ASP.NET Core |
| Matching | Java + Spring Boot |
| Ranking | ASP.NET Core |
| Payments | ASP.NET Core |
| Communication | ASP.NET Core |

## Integración

### REST
Interacción síncrona cuando el consumidor requiere respuesta inmediata.

### Kafka
Hechos de dominio y coordinación asíncrona. La EDA no obliga a convertir toda comunicación a eventos.

### Datos
PostgreSQL/PostGIS. Cada servicio mantiene ownership lógico de sus datos.

### Infraestructura
8 microservicios en VM3/k3s; resto de roles distribuidos entre las 7 VMs según `INFRASTRUCTURE.md`.

## Restricciones

- multi-tenancy;
- tokenización y PCI-DSS;
- 7 VMs;
- infraestructura propia;
- Bogotá como alcance MVP;
- contratos explícitos;
- tiempo académico limitado.
