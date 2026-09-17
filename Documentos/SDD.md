# SDD - Software Design Description - QUICKPATCH

**Proyecto:** QUICKPATCH - Plataforma multi-tenant de servicios técnicos  
**Documento:** Descripción de Diseño de Software (SDD)  
**Estado:** Borrador integrable - Vista Lógica desarrollada  
**Fecha:** Septiembre de 2026

---

# 1. Introducción

Este documento consolida el diseño de software de QUICKPATCH a partir de la arquitectura, requisitos, modelo de datos y lineamientos de desarrollo vigentes. La organización del SDD se mantiene por vistas complementarias del sistema, integrando una **Vista Lógica** ya desarrollada y dejando preparadas las secciones que deberán ser completadas por los responsables de las demás vistas.

La estructura toma como referencia el enfoque 4+1 para separar la descripción lógica, los procesos en ejecución, la organización del desarrollo, el despliegue físico y los escenarios que conectan las demás vistas.

---

# 2. Fuentes de diseño y actualización de diagramas

Los diagramas y descripciones del SDD deben mantenerse alineados con las fuentes vigentes del proyecto. Ningún diagrama debe actualizarse de forma aislada si el cambio contradice requisitos, decisiones arquitectónicas, contratos o la implementación real.

| Artefacto / vista | Fuentes primarias de diseño | Elementos que deben mantenerse sincronizados |
|---|---|---|
| **Vista Lógica** | `SAD.md`, `SRS.md`, `DD.md` | Microservicios, módulos, dominios, entidades, relaciones lógicas, responsabilidades, contratos y eventos. |
| **Vista de Procesos** | `SRS.md`, `DD.md`, `SAD.md`, escenarios de calidad | Flujos síncronos/asíncronos, secuencias, productores/consumidores, concurrencia, reintentos, fallos. |
| **Vista de Desarrollo** | Repositorio de código, `Documento Politicas y Herramientas.md`, `SAD.md` | Carpetas, paquetes, proyectos, dependencias, convenciones, librerías, pruebas y CI/CD. |
| **Vista Física** | `SAD.md`, manifiestos de despliegue, archivos de infraestructura y políticas | VMs, contenedores, redes, puertos, servicios, ambientes, observabilidad y seguridad de infraestructura. |
| **Vista de Escenarios (+1)** | `SRS.md`, flujos de negocio, historias de usuario, escenarios de calidad | Casos representativos que atraviesan y validan las otras cuatro vistas. |
| **Diagramas de datos** | `DD.md`, contratos vigentes, migraciones/esquemas implementados | Entidades, campos, relaciones físicas, referencias lógicas y propiedad de datos por servicio. |
| **Diagramas de componentes** | `SAD.md`, `SDD.md`, código implementado | Límites de servicio, interfaces, dependencias permitidas y canales de comunicación. |

## 2.1 Regla de actualización

Para cualquier diagrama nuevo o modificado se debe verificar, según corresponda:

- requisito funcional o no funcional relacionado;
- microservicio o módulo propietario;
- contrato REST o evento involucrado;
- datos que consume o produce;
- relación con ADRs vigentes;
- correspondencia con el código, repositorio y despliegue real.

---

# 3. Arquitectura

## 3.1 Vista Lógica del Sistema

La arquitectura lógica del backend se organiza por dominios y servicios. Los clientes acceden a través del API Gateway; los microservicios contienen capacidades de negocio separadas; Kafka gestiona la integración asíncrona; y PostgreSQL/PostGIS, Redis y MinIO soportan persistencia, cache y evidencias.

### 3.1.1 Vista lógica general por capas

![Vista lógica general por capas](sdd_v2_assets/01_vista_logica_capas.png)

**Figura 1. Vista lógica general por capas de QUICKPATCH.**

La representación organiza el sistema en capa de cliente, entrada/presentación, servicios de negocio, integración y datos. Los ocho microservicios definidos en la arquitectura de alto nivel aparecen dentro de la capa de negocio y se conectan con los recursos de integración y persistencia correspondientes.

### 3.1.2 Componentes lógicos principales del backend

![Diagrama de componentes del backend](sdd_v2_assets/02_componentes_backend.png)

**Figura 2. Componentes lógicos principales del backend.**

El API Gateway concentra la entrada de solicitudes síncronas. ServiceRequest, Matching, Payments, Ranking y Communication participan en los flujos de eventos por Kafka. Cada servicio conserva su límite funcional y su propiedad de datos.

### 3.1.3 Principios de organización lógica

#### Separación por dominio

Cada microservicio representa una capacidad de negocio diferenciada y mantiene sus reglas dentro de su propio límite lógico.

#### Propiedad de datos

Cada servicio es propietario de los datos asociados a su dominio. Las relaciones internas al mismo servicio pueden implementarse con claves foráneas físicas; las relaciones entre dominios se representan mediante identificadores lógicos, contratos o eventos.

Ejemplo:

```text
ServiceRequest.client_id -> UUID del usuario en Identity
```

#### Comunicación por contratos

- REST se utiliza para operaciones en las que el cliente necesita una respuesta inmediata.
- Kafka se utiliza para coordinación asíncrona entre dominios.

#### Consistencia entre servicios

La comunicación asíncrona implica consistencia eventual entre dominios. Los eventos se gestionan con mecanismos de idempotencia, reintentos y Transactional Outbox.

#### Multi-tenancy

Los datos de negocio asociados a empresas conservan `tenant_id`. El contexto de tenant se deriva de la identidad autenticada y se aplica en las operaciones de lectura y escritura correspondientes.

---

## 3.2 Descomposición lógica por microservicio

### 3.2.1 API Gateway

**Responsabilidades:**

- enrutamiento hacia el servicio correspondiente;
- validación inicial del contexto de autenticación;
- propagación de identidad, rol y tenant;
- aplicación de políticas transversales de API;
- rechazo de tokens inválidos o expirados.

El Gateway no contiene reglas propias de matching, solicitudes, pagos, ranking ni facturación.

### 3.2.2 Identity Service

**Dominio:** identidad, autenticación, usuarios y tenancy.

**Módulos lógicos:**

- Auth
- Tenants
- Users
- Authorization / Roles

| Concepto | Tipo | Responsabilidad |
|---|---|---|
| `Tenant` | Entidad persistente | Empresa o espacio lógico independiente dentro de QUICKPATCH. |
| `User` | Entidad persistente | Identidad autenticable de una persona. |
| `TechnicianProfile` | Entidad persistente | Información específica del perfil técnico vinculada a un usuario. |
| `Role` | Concepto de autorización | Conjunto de permisos para cada tipo de usuario. |
| `AuthenticatedContext` | Objeto lógico | Propaga `userId`, `tenantId` y rol. |

**Datos documentados:** `tenants`, `users`, `technician_profiles`.

### 3.2.3 Actors Service

**Dominio:** actores de negocio distintos de la identidad técnica.

**Módulos lógicos:**

- Suppliers
- Allies
- Clients

| Concepto | Estado | Responsabilidad |
|---|---|---|
| `Supplier` | Conceptual / evolutivo | Proveedor de materiales o repuestos. |
| `Ally` | Conceptual / evolutivo | Técnico, profesional o empresa aliada que presta servicios. |
| `Client` | Conceptual / evolutivo | Cliente hogar o empresarial que contrata servicios. |

La persistencia detallada de estos conceptos se incorpora al DD cuando el Sprint correspondiente defina sus atributos y reglas.

### 3.2.4 Catalog Service

**Dominio:** catálogo de servicios y especialidades.

**Responsabilidades:**

- mantener categorías y especialidades;
- exponer opciones válidas para la creación de solicitudes;
- suministrar información de catálogo a clientes y procesos internos.

| Concepto | Persistencia | Descripción |
|---|---|---|
| `ServiceCategory` | `service_categories` | Categoría o tipo de servicio solicitado. |

### 3.2.5 ServiceRequest Service

**Dominio:** ciclo de vida de la solicitud de servicio.

**Módulos lógicos:**

- Request Management
- Service Lifecycle
- Ratings
- State Validation

| Concepto | Tipo | Responsabilidad |
|---|---|---|
| `ServiceRequest` | Entidad / agregado central | Representa la solicitud y controla sus transiciones válidas. |
| `ServiceCategory` | Entidad relacionada | Clasifica el servicio requerido. |
| `Rating` | Entidad | Registra la evaluación posterior al servicio. |
| `ServiceRequestStatus` | Value Object / Enum conceptual | Estado válido de la solicitud. |

**Estados actuales:**

```text
buscando_tecnico
    -> en_espera
    -> asignado
    -> en_progreso
    -> completado
    -> pagado
```

**Datos documentados:** `service_categories`, `service_requests`, `ratings`.

**Referencias lógicas externas:** `client_id`, `technician_id`, `tenant_id`.

**Eventos:**

- produce `service-request.created`;
- produce `service-request.completed`;
- consume `matching.technician-assigned`;
- consume `matching.no-technician-available`;
- consume `payment.approved`;
- consume `payment.rejected` cuando corresponda al flujo.

### 3.2.6 Matching Service

**Tecnología asignada:** Java + Spring Boot.

**Dominio:** selección y asignación de técnicos según disponibilidad, cobertura y criterios de proximidad.

**Módulos lógicos:**

- Availability
- Coverage
- Candidate Search
- Matching Attempts
- Assignment

| Concepto | Persistencia | Responsabilidad |
|---|---|---|
| `TechnicianAvailability` | `technician_availability` | Disponibilidad temporal de un técnico. |
| `CoverageZone` | `coverage_zones` | Área geográfica cubierta por un técnico. |
| `MatchingAttempt` | `matching_attempts` | Intento de asignación y resultado. |
| `Candidate` | Objeto lógico | Técnico evaluado durante el matching. |
| `MatchResult` | Objeto lógico | Resultado del proceso de matching. |

**Flujo lógico principal:**

El siguiente diagrama representa el flujo lógico principal del proceso de matching.

![Diagrama de flujo del Matching Service](sdd_v3_assets/05_matching_flow.png)

*Figura 3. Flujo lógico principal del Matching Service.*

### 3.2.7 Ranking Service

**Dominio:** reputación y evaluación agregada.

**Responsabilidades:**

- recalcular reputación de técnicos/aliados;
- mantener ranking de servicio y, cuando aplique, ranking de materiales/proveedores;
- consumir resultados y evaluaciones del ciclo de servicio.

Los modelos persistentes definitivos de ranking permanecen sujetos a evolución del DD.

### 3.2.8 Payments Service

**Dominio:** pagos y facturación.

**Módulos lógicos:**

- Payment Processing
- PSP Adapter
- Billing
- Invoice Management

| Concepto | Persistencia | Responsabilidad |
|---|---|---|
| `Payment` | `payments` | Estado de una operación de pago sin almacenar PAN/CVV. |
| `Invoice` | `invoices` | Factura asociada al pago o servicio. |
| `PaymentTokenReference` | Value Object | Referencia al token suministrado por el PSP. |
| `PaymentStatus` | Value Object / Enum conceptual | Estado controlado de la transacción. |

**Eventos:** `payment.approved`, `payment.rejected`.

### 3.2.9 Communication Service

**Dominio:** notificaciones y comunicación desacoplada.

**Responsabilidades:**

- reaccionar a eventos del ciclo de servicio;
- generar notificaciones para clientes y técnicos;
- ejecutar comunicaciones que no deben bloquear operaciones síncronas.

Chat y reclamaciones permanecen como funcionalidades futuras hasta que sean incorporadas por un Sprint y requisito explícito.

---

## 3.3 Modelo lógico de dominio y relaciones

![Modelo lógico de dominio](sdd_v2_assets/03_modelo_logico_dominio.png)

**Figura 3. Modelo lógico de dominio y referencias entre servicios.**

Las relaciones continuas representan relaciones internas al mismo dominio que pueden implementarse como claves foráneas. Las relaciones punteadas representan referencias lógicas entre servicios independientes.

### 3.3.1 Relaciones internas

```text
service_requests.category_id -> service_categories.id
ratings.service_request_id   -> service_requests.id
invoices.payment_id          -> payments.id
```

### 3.3.2 Referencias lógicas entre servicios

```text
service_requests.client_id      -> Identity / User
service_requests.technician_id  -> Identity / Technician
matching_attempts.request_id    -> ServiceRequest / ServiceRequest
payments.service_request_id     -> ServiceRequest / ServiceRequest
```

Un servicio no ejecuta escrituras directas sobre tablas propietarias de otro dominio ni define Foreign Keys físicas entre límites de servicio diferentes.

---

## 3.4 Contratos lógicos principales

### 3.4.1 REST

```text
POST /v1/auth/register/client
POST /v1/auth/register/technician
POST /v1/auth/register/company
POST /v1/auth/login
GET  /v1/users/me

POST /v1/service-requests
GET  /v1/service-requests/{id}
POST /v1/service-requests/{id}/start
POST /v1/service-requests/{id}/complete
POST /v1/service-requests/{id}/rating
POST /v1/service-requests/{id}/payment
GET  /v1/payments/{id}/invoice
```

### 3.4.2 Eventos de dominio

```text
service-request.created
matching.technician-assigned
matching.no-technician-available
service-request.completed
payment.approved
payment.rejected
```

**Sobre común del evento:**

```json
{
  "eventId": "uuid",
  "eventVersion": 1,
  "correlationId": "uuid",
  "tenantId": "uuid",
  "producer": "service-name",
  "occurredAt": "timestamp",
  "data": {}
}
```

---

# 4. Diseño Detallado

## 4.1 Estructura interna de los microservicios backend

![Capas internas de un microservicio](sdd_v2_assets/04_capas_microservicio.png)

**Figura 4. Estructura lógica interna de un microservicio.**

La estructura interna se divide en cuatro capas lógicas:

| Capa | Contenido |
|---|---|
| **API / Interface** | Controllers, endpoints, validación de entrada y contratos expuestos. |
| **Application** | Casos de uso, comandos, queries y coordinación del flujo de aplicación. |
| **Domain** | Entidades, value objects y reglas de negocio. |
| **Infrastructure** | Repositorios, Kafka, PostgreSQL, Redis, PSP y adaptadores externos. |

**Dirección conceptual de dependencias:**

```text
API -> Application -> Domain
Infrastructure -> Application / Domain mediante interfaces
```

El detalle definitivo de proyectos, carpetas y paquetes debe quedar sincronizado con la Vista de Desarrollo.

## 4.2 Reglas transversales del backend

### Multi-tenancy

- los datos de negocio incluyen contexto de tenant cuando corresponde;
- las consultas aplican aislamiento de tenant;
- el tenant se deriva del contexto autenticado;
- PostgreSQL RLS puede actuar como defensa adicional.

### Seguridad de pagos

- no se persiste PAN/CVV;
- se almacena únicamente información permitida y referencias tokenizadas del PSP.

### Idempotencia

Los consumidores de eventos toleran mensajes repetidos y utilizan `eventId` para identificar efectos ya procesados.

### Transactional Outbox

Las operaciones que modifican estado de negocio y publican eventos registran localmente el cambio y el evento pendiente; el publicador reintenta ante indisponibilidad temporal de Kafka.

### Trazabilidad

Los cambios relevantes del ciclo de servicio y pago conservan información suficiente para reconstrucción y auditoría.

## 4.3 Trazabilidad con atributos de calidad

| Decisión de diseño | Atributos relacionados | Reflejo en el diseño |
|---|---|---|
| Microservicios por dominio | Escalabilidad, disponibilidad, mantenibilidad | Servicios separados y despliegue independiente. |
| Kafka como bus de eventos | Disponibilidad, escalabilidad, trazabilidad | Integración asíncrona y contratos de eventos. |
| PostgreSQL + PostGIS | Rendimiento | Consultas geoespaciales del Matching Service. |
| `tenant_id` + controles de acceso | Seguridad, escalabilidad | Aislamiento lógico de empresas. |
| Outbox + idempotencia | Disponibilidad, trazabilidad | Publicación confiable y tolerancia a duplicados. |
| Tokenización de pagos | Seguridad | Backend sin almacenamiento de PAN/CVV. |

## 4.4 Elementos de diseño todavía evolutivos

Permanecen sujetos a definición de Sprints posteriores:

1. persistencia definitiva de Actors (`Supplier`, `Ally`, `Client`);
2. persistencia detallada de Ranking;
3. modelos persistentes de Notification, Chat y Complaints;
4. modelos relacionados con payroll-lite, si entran en alcance implementable;
5. nuevos eventos o endpoints introducidos por historias de usuario futuras;
6. value objects y clases auxiliares surgidos de la implementación.

---

# 5. Vista de Procesos - pendiente de integración

**Corresponde a:** Backend Developers + QA.

## Recomendación de contenido

- procesos y servicios en ejecución;
- flujos síncronos y asíncronos;
- secuencia de creación de solicitud y matching;
- secuencia de pago y facturación;
- productores y consumidores Kafka;
- consumer groups;
- concurrencia del matching;
- reintentos, backoff y DLQ;
- idempotencia y Transactional Outbox;
- manejo de fallos de consumidores y Kafka;
- escenarios de carga y puntos de sincronización.

## Diagramas recomendados

- diagrama de secuencia: creación de solicitud -> matching -> asignación;
- diagrama de secuencia: completar servicio -> pago -> factura;
- diagrama de secuencia: indisponibilidad de Kafka -> Outbox -> reintento;
- diagrama de actividad para transiciones principales del ciclo de servicio.

---

# 6. Vista de Desarrollo - pendiente de integración

**Corresponde a:** Backend + Frontend (Miguel).

## Recomendación de contenido

- estructura real del repositorio;
- organización por aplicación y microservicio;
- proyectos/capas del backend .NET;
- paquetes del Matching Service en Java/Spring Boot;
- organización Flutter;
- organización Next.js;
- librerías compartidas permitidas;
- contratos y DTOs;
- ubicación de pruebas;
- dependencias entre proyectos;
- convenciones de nombres;
- reglas de acoplamiento entre servicios;
- integración con CI/CD por servicio.

## Diagramas recomendados

- diagrama de paquetes del repositorio;
- diagrama de componentes por aplicación;
- diagrama de dependencias entre proyectos/módulos;
- mapa de carpetas y artefactos de build.

---

# 7. Vista Física - pendiente de integración

**Corresponde a:** DevOps (Sebastian).

## Recomendación de contenido

- topología de las 7 VMs;
- API Gateway / Nginx;
- Next.js;
- k3s en VM de backend;
- PostgreSQL + PostGIS;
- Redis;
- Kafka + Kafka UI;
- MinIO y observabilidad;
- redes, puertos y reglas de seguridad;
- ambientes Dev / QA / Prod;
- contenedores y manifiestos;
- healthchecks, readiness y liveness;
- orden de arranque;
- Ansible;
- CI/CD y estrategia de despliegue;
- limitación del clúster k3s de un solo nodo.

## Diagramas recomendados

- diagrama UML de despliegue;
- topología de red y VMs;
- distribución de contenedores por nodo;
- flujo de entrada desde cliente hasta Gateway y servicios;
- diagrama de ambientes Dev / QA / Prod.

---

# 8. Vista de Escenarios (+1) - pendiente de integración

**Corresponde a:** Product Owner (Angy) + todo el equipo.

## Recomendación de escenarios

1. cliente registra una cuenta y crea una solicitud;
2. sistema encuentra y asigna un técnico;
3. no existe técnico disponible y la solicitud queda en espera;
4. técnico inicia y completa el servicio;
5. cliente realiza el pago;
6. PSP aprueba o rechaza el pago;
7. cliente califica el servicio;
8. empresa/tenant consulta información sin acceder a datos de otro tenant;
9. Kafka presenta una falla temporal y el evento se recupera mediante Outbox.

## Para cada escenario se recomienda documentar

- actor;
- precondiciones;
- flujo principal;
- flujos alternos;
- servicios participantes;
- datos involucrados;
- eventos y endpoints;
- relación con Vista Lógica;
- relación con Vista de Procesos;
- relación con Vista de Desarrollo;
- relación con Vista Física;
- requisitos funcionales y no funcionales asociados.

## Diagramas recomendados

- casos de uso de escenarios principales;
- secuencia de extremo a extremo para los escenarios críticos;
- matriz escenario -> componentes -> vistas.

---

# 9. Integración entre vistas

Antes de consolidar una versión final del SDD se debe verificar:

- [ ] todo microservicio de la Vista Lógica existe en la Vista de Desarrollo;
- [ ] todo componente desplegable de la Vista de Desarrollo aparece en la Vista Física;
- [ ] los flujos de la Vista de Procesos utilizan componentes definidos en la Vista Lógica;
- [ ] los escenarios (+1) pueden seguirse a través de las cuatro vistas;
- [ ] los contratos REST coinciden con DD/OpenAPI;
- [ ] los eventos coinciden con DD/AsyncAPI;
- [ ] los estados de `ServiceRequest` son consistentes con SRS, DD y código;
- [ ] los nombres de servicios son uniformes en SAD, SDD, repositorio y despliegue;
- [ ] ningún servicio modifica directamente datos propiedad de otro dominio;
- [ ] las decisiones de seguridad multi-tenant aparecen de forma coherente en todas las vistas.

---

# 10. Estado actual del documento

| Sección | Corresponde a | Estado |
|---|---|---|
| Vista Lógica | Backend | **Desarrollada - lista para revisión** |
| Vista de Procesos | Backend + QA | Pendiente |
| Vista de Desarrollo | Backend + Frontend / Miguel | Pendiente |
| Vista Física | DevOps / Sebastian | Pendiente |
| Vista de Escenarios (+1) | Angy + equipo | Pendiente |
| Revisión cruzada | Todo el equipo | Pendiente al completar las vistas |

---

## Nota para asistentes de IA

Al utilizar Claude Code, Codex u otro asistente sobre este SDD:

1. tratar `SRS.md` como fuente de requisitos;
2. tratar `SAD.md` como fuente de decisiones arquitectónicas y atributos de calidad;
3. tratar `DD.md` como fuente del modelo de datos y contratos vigentes;
4. utilizar este SDD para el diseño interno y las responsabilidades lógicas;
5. no inventar módulos, entidades o eventos para completar diagramas;
6. si existe contradicción entre documentos, señalarla antes de modificar el diseño;
7. mantener separadas las responsabilidades de las vistas;
8. actualizar los diagramas junto con las fuentes de diseño correspondientes.
