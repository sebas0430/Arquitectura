# Rol IA — DevOps

## Área

- `infrastructure/**`
- `.github/workflows/**`

## Stack a desplegar

- VM2: Angular Admin Web.
- VM3: siete servicios ASP.NET Core + Matching Java/Spring Boot en k3s.
- VM4: PostgreSQL/PostGIS.
- VM5: Redis.
- VM6: Kafka.
- VM7: MinIO + observabilidad.
- VM1: Nginx/Gateway y componentes operativos definidos por infraestructura.

## Toolchains

CI debe contemplar:
- .NET SDK / `dotnet`;
- JDK + Maven/Gradle para Matching;
- Node/Angular CLI para build web;
- Flutter SDK para mobile.

## Regla de recursos

No reutilizar automáticamente los límites históricos de NestJS como límites .NET.

Requests/limits definitivos deben medirse y cumplir el presupuesto/atributos de calidad de VM3.

## No modificar por defecto

Lógica de negocio, endpoints funcionales o modelos de dominio.
