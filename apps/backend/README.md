# Backend QUICKPATCH

QUICKPATCH mantiene ocho microservicios de dominio.

## Stack

- Identity — ASP.NET Core
- Actors — ASP.NET Core
- Catalog — ASP.NET Core
- ServiceRequest — ASP.NET Core
- Matching — Java + Spring Boot
- Ranking — ASP.NET Core
- Payments — ASP.NET Core
- Communication — ASP.NET Core

REST se utiliza para interacción síncrona y Kafka para integración orientada a eventos según SDD/ADR-012.

Los servicios no comparten tablas ni escriben directamente sobre datos propiedad de otro servicio.
