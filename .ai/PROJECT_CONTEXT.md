# QUICKPATCH — Contexto global para IA

## Propósito

QUICKPATCH es una plataforma multi-tenant de servicios técnicos para hogares y empresas. Conecta clientes con técnicos, gestiona solicitudes, matching, seguimiento, pagos, reputación, comunicaciones y evidencia del servicio.

Este archivo es un resumen operativo. No reemplaza SRS, SAD, SDD ni DD.

## Stack vigente de referencia

La documentación actual define, entre otros componentes:

- Aplicación móvil: Flutter.
- Frontend web administrativo: Next.js.
- Backend distribuido por dominios/microservicios.
- PostgreSQL + PostGIS.
- Apache Kafka para integración asíncrona.
- Redis para cache y soporte operativo.
- MinIO para evidencias/objetos.
- Infraestructura propia sobre 7 VMs.
- Docker/Compose y k3s según el componente.
- Ansible para aprovisionamiento.
- Observabilidad con herramientas definidas en el documento de infraestructura.

> Antes de introducir o cambiar tecnología, valida SAD, SDD e Infraestructura.

## Reglas arquitectónicas críticas

1. El sistema es multi-tenant.
2. El aislamiento entre tenants no puede debilitarse.
3. El backend no debe almacenar PAN/CVV de tarjetas.
4. Kafka se usa donde la arquitectura define integración asíncrona.
5. Los cambios de contratos deben ser explícitos.
6. La infraestructura productiva está limitada por las restricciones del SAD.
7. Las evidencias de servicio forman parte del diseño vigente.
8. Las decisiones arquitectónicas deben mantener trazabilidad con requisitos y atributos de calidad.

## Flujo Git

Modelo adoptado: GitFlow.

- `main`: versión estable.
- `develop`: integración.
- `feature/*`: nuevas funcionalidades desde `develop`.
- `bugfix/*`: correcciones de desarrollo/release.
- `release/*`: preparación de entrega.
- `hotfix/*`: corrección urgente desde `main`.

Convención recomendada para trabajo por rol:

- `feature/backend/<descripcion>`
- `feature/frontend/<descripcion>`
- `feature/qa/<descripcion>`
- `feature/devops/<descripcion>`

## Estrategia de contexto

Carga contexto progresivamente:

1. Este archivo.
2. Resumen arquitectónico.
3. Rol.
4. Contratos/requisitos relacionados.
5. Código de la tarea.

No leas todo el repositorio sin necesidad.

## Contratos compartidos

Los siguientes elementos son fronteras entre equipos:

- API REST/OpenAPI.
- Esquemas de eventos Kafka.
- Modelos compartidos explícitos.
- Variables/configuración pública entre aplicación e infraestructura.

Un cambio en una frontera debe señalar qué roles consumidores se ven afectados.

## Definition of Done mínima

Una tarea no está terminada hasta que:

- cumple el requisito asociado;
- respeta arquitectura y multi-tenancy;
- tiene pruebas apropiadas;
- no rompe contratos conocidos;
- documenta cambios relevantes;
- pasa lint/format/checks aplicables.
