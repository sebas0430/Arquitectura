# QUICKPATCH — Resumen arquitectónico para agentes

## Fuentes formales

- Requisitos: `Documentos/SRS.md`
- Arquitectura: `Documentos/SAD.md`
- Diseño: `Documentos/SDD.md`
- Datos/contratos: `Documentos/DDv2.md` (vigente temporal)
- Infraestructura: `Documentos/Documento de Infraestructura.md`

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

El nombre exacto de módulos y su implementación debe verificarse contra SAD/SDD antes de crear código.

## Integraciones

### Síncronas
El API Gateway concentra la entrada y enruta hacia servicios correspondientes.

### Asíncronas
Kafka comunica procesos de negocio donde se requiere desacoplamiento, resiliencia o procesamiento en segundo plano.

### Persistencia
PostgreSQL/PostGIS soporta datos relacionales/geoespaciales. MinIO almacena objetos/evidencias. Redis se usa según el diseño documentado.

## Restricciones que no se deben ignorar

- multi-tenancy;
- PCI-DSS y tokenización de pagos;
- infraestructura propia;
- 7 VMs fijas;
- operación académica/no 24x7;
- despliegue reproducible;
- Bogotá como alcance del MVP;
- tiempo académico limitado.

## Cómo usar este resumen

Este documento sirve para localizar la parte del SAD/SDD que debes consultar. No uses este resumen como justificación para inventar endpoints, tablas o eventos.
