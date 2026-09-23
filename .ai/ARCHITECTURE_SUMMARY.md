# QUICKPATCH — Resumen arquitectónico para agentes

## Fuentes formales

- Requisitos: `docs/requirements/SRS.md`
- Arquitectura: `docs/architecture/SAD.md`
- Diseño: `docs/architecture/SDD.md`
- Datos/contratos: `docs/design/DD.md`
- Infraestructura: `docs/infrastructure/INFRASTRUCTURE.md`
- Políticas de trabajo: `docs/governance/WORKING_AGREEMENTS.md`
- Índice documental: `docs/README.md`

## Componentes de dominio

La documentación vigente referencia ocho capacidades/microservicios principales:

- Identity
- Actors
- Catalog
- ServiceRequest
- Matching
- Ranking
- Payments
- Communication

El nombre exacto de módulos y su implementación debe verificarse contra `docs/architecture/SAD.md` y `docs/architecture/SDD.md` antes de crear código.

## Integraciones

### Síncronas

El API Gateway concentra la entrada y enruta hacia los servicios correspondientes.

### Asíncronas

Kafka comunica procesos de negocio donde se requiere desacoplamiento, resiliencia o procesamiento en segundo plano.

### Persistencia

PostgreSQL/PostGIS soporta datos relacionales y geoespaciales.

MinIO almacena objetos y evidencias.

Redis se usa según el diseño arquitectónico y operativo documentado.

## Restricciones que no se deben ignorar

- multi-tenancy;
- PCI-DSS y tokenización de pagos;
- infraestructura propia;
- 7 VMs fijas;
- operación académica/no 24x7;
- despliegue reproducible;
- Bogotá D.C. como alcance geográfico del MVP;
- tiempo académico limitado.

## Precedencia documental

1. `docs/requirements/SRS.md`
2. `docs/architecture/SAD.md`
3. `docs/architecture/SDD.md`
4. `docs/design/DD.md`
5. `docs/infrastructure/INFRASTRUCTURE.md`
6. `docs/governance/WORKING_AGREEMENTS.md`

Este resumen sirve para localizar información.

No debe utilizarse como justificación para inventar endpoints, tablas, eventos o decisiones arquitectónicas.
