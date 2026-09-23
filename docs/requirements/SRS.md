# Especificación de Requisitos de Software (SRS)
## QUICKPATCH — Ciudad de Bogotá D.C.

| | |
|---|---|
| **Curso** | Arquitectura de Software |
| **Proyecto Jira** | SCRUM — Arquitectura de Software |
| **Versión del documento** | 3.2 (MVP) |
| **Estándar de referencia** | IEEE Std 830-1998 |

---

## Índice

1. [Introducción](#1-introducción)
2. [Descripción General](#2-descripción-general)
3. [Usuarios del Sistema](#3-usuarios-del-sistema)
4. [Módulos del Producto](#4-módulos-del-producto)
5. [Requisitos Específicos](#5-requisitos-específicos)
6. [Matriz Comparativa](#6-matriz-comparativa)
7. [Apéndices](#7-apéndices)
8. [Referencias](#referencias)

---

## 1. Introducción

### 1.1 Propósito

El propósito de este documento es especificar de manera clara, completa y verificable los requisitos de software para la **Versión 3 (MVP)** de la plataforma multi-tenant de servicios técnicos para el hogar y empresas. Este documento sigue la estructura general del estándar IEEE 830 y está dirigido a:

- El equipo de desarrollo, como guía para el diseño, la implementación y las pruebas del sistema.
- El docente y evaluadores del curso de Arquitectura de Software, como soporte académico del proyecto.
- Futuros integrantes del equipo, como documentación de referencia del alcance funcional del MVP.

### 1.2 Definiciones, Acrónimos y Abreviaturas

| Término | Definición |
|---|---|
| MVP | Minimum Viable Product — Producto Mínimo Viable. |
| SRS | Software Requirements Specification — Especificación de Requisitos de Software. |
| SAD | Software Architecture Document — Documento de Arquitectura de Software. |
| DD | Detailed Design — Diseño Detallado. |
| Tenant | Entidad lógica que representa a un cliente independiente (hogar o empresa) dentro de una arquitectura multi-tenant, con sus datos aislados de otros tenants. |
| HU | Historia de Usuario. |
| RF | Requisito Funcional. |
| RNF | Requisito No Funcional. |
| Feature | Agrupación lógica de historias de usuario relacionadas dentro de una misma épica, usada en Jira como etiqueta (*label*) para organizar el backlog. |
| Módulo de Dominio | Componente del sistema que implementa directamente una capacidad de negocio central (ej. matching, pagos). |
| Módulo Transversal | Componente de soporte utilizado por varios (o todos) los módulos de dominio (ej. seguridad, multi-tenancy, logging). |
| PCI-DSS | Payment Card Industry Data Security Standard — estándar de seguridad para el manejo de datos de tarjetas de pago. |
| Matching | Proceso automático mediante el cual el sistema asigna un técnico disponible a una solicitud de servicio. |
| Admin | Usuario administrador de la plataforma, con visibilidad y control operativo sobre uno o varios tenants. |
| Proveedor | Empresa u organización que agrupa y administra un equipo de técnicos dentro de la plataforma. |

---

## 2. Descripción General

### 2.1 Perspectiva del Producto

La plataforma es un sistema nuevo, independiente, compuesto por:

- Una aplicación web/móvil para **Clientes** y **Empresas** (creación y seguimiento de solicitudes).
- Una aplicación para **Técnicos** y **Proveedores** (gestión de disponibilidad y ejecución de servicios).
- Un panel administrativo para el rol **Admin** (operación, verificación y control multi-tenant).
- Un backend con arquitectura **multi-tenant**, un motor de *matching*, y una pasarela de pagos certificada **PCI-DSS** integrada mediante un proveedor externo.

### 2.2 Funciones del Producto

A alto nivel, el MVP debe permitir:

1. Registro y autenticación segura de usuarios, con aislamiento de datos por tenant.
2. Creación de solicitudes de servicio técnico por parte de Clientes y Empresas.
3. Asignación automática (matching) de un técnico disponible a cada solicitud.
4. Ejecución y seguimiento del servicio por parte del Técnico, en tiempo real para el Cliente.
5. Procesamiento seguro de pagos y generación de comprobantes/facturas.
6. Calificación del servicio por parte del Cliente.
7. Administración operativa (verificación de técnicos, gestión de tenants) por parte del Admin.
8. Infraestructura mínima de CI/CD, pruebas automatizadas, logging/monitoreo y cifrado de datos sensibles.

### 2.3 Restricciones

- El manejo de datos de tarjetas de pago debe cumplir el estándar **PCI-DSS**; no se deben almacenar datos completos de tarjeta en los servidores propios del sistema.
- La arquitectura debe garantizar **aislamiento lógico** de datos entre tenants.
- El MVP se limita a la ciudad de **Bogotá D.C.** como zona de cobertura inicial.
- El desarrollo debe ajustarse al tiempo y alcance definidos para el curso de Arquitectura de Software.

### 2.4 Supuestos y Dependencias

- Se asume disponibilidad de un proveedor de pagos externo certificado PCI-DSS (pasarela de pagos) para el procesamiento de transacciones.
- Se asume conectividad a internet estable por parte de Clientes y Técnicos para el uso de la plataforma.
- El listado inicial de técnicos y proveedores se registra manualmente para efectos de demostración del MVP.

---

## 3. Usuarios del Sistema

La plataforma QUICKPATCH está diseñada para cinco tipos de usuario, cada uno con necesidades, nivel de acceso y objetivos distintos dentro del ciclo de negocio (*Cliente solicita → Técnico ejecuta → Cliente paga → Cliente califica*).

| Rol | Descripción | Necesidades | Nivel de acceso |
|---|---|---|---|
| Cliente | Persona natural que solicita servicios técnicos para el hogar. No requiere conocimientos técnicos previos. | Solicitar un servicio rápido y confiable, hacer seguimiento en tiempo real, pagar de forma segura y calificar la experiencia. | Autenticado — datos propios y de su tenant únicamente. |
| Empresa | Cuenta corporativa (ej. una compañía que requiere mantenimiento en sus sedes) que solicita y paga servicios técnicos a nombre de la organización. | Centralizar solicitudes y pagos de varias sedes bajo una sola cuenta; delegar la solicitud a distintos usuarios autorizados. | Autenticado — cuenta corporativa aislada como tenant propio. |
| Técnico | Persona certificada que ejecuta los servicios técnicos solicitados. Puede operar de forma independiente o vinculada a un Proveedor. | Recibir solicitudes acordes a su especialidad y zona, gestionar su disponibilidad, y recibir el pago de forma confiable. | Autenticado — solo sus propias solicitudes y pagos. |
| Proveedor | Empresa que administra un equipo de técnicos dentro de la plataforma. | Administrar su equipo de técnicos y visualizar el desempeño y los pagos generados por su equipo. | Autenticado — su equipo y su tenant asociado. |
| Admin | Usuario interno de la plataforma con permisos de verificación, supervisión y operación sobre uno o varios tenants. | Verificar técnicos/proveedores, monitorear la operación en tiempo real y administrar los tenants de la plataforma. | Autenticado con privilegios elevados — uno o varios tenants según alcance. |

---

## 4. Módulos del Producto

El sistema se organiza en dos tipos de módulos: los **módulos de dominio**, que implementan directamente una capacidad central del negocio, y los **módulos transversales**, que dan soporte de infraestructura a todos (o varios) de los módulos de dominio y no son visibles como una función de negocio independiente para el usuario final.

### 4.1 Módulos de Dominio

| Módulo | Descripción | Épica / Features |
|---|---|---|
| Gestión de Usuarios y Cuentas | Registro y autenticación de Cliente, Técnico/Proveedor y Empresa; administración de roles y permisos. | SCRUM-7 (F1.1, F1.3, F1.4) |
| Solicitudes y Matching | Creación de solicitudes de servicio y motor de asignación automática de técnico (matching). | SCRUM-8 (F2.1, F2.2) |
| Seguimiento y Calificación | Seguimiento en tiempo real del estado de la solicitud y calificación del servicio al finalizar. | SCRUM-8 (F2.3, F2.4) |
| Ejecución de Servicios (Técnico) | Gestión de disponibilidad, detalle y cierre de solicitudes asignadas, historial de servicios. | SCRUM-9 (F3.1, F3.2, F3.4) |
| Gestión de Equipos (Proveedor) | Administración del equipo de técnicos por parte de un Proveedor. | SCRUM-9 (F3.3) |
| Administración y Operaciones | Dashboard operativo, aprobación/suspensión de técnicos, gestión de tenants. | SCRUM-10 (F4.1, F4.2, F4.3) |
| Pagos y Facturación | Cobro del servicio (Cliente/Empresa), generación de comprobantes y registro de pagos recibidos. | SCRUM-11 (F5.1, F5.2, F5.3) |

### 4.2 Módulos Transversales

| Módulo | Descripción | Épica / Features |
|---|---|---|
| Multi-tenancy | Aísla lógicamente los datos de cada tenant (*tenant_id*) en todas las operaciones de lectura/escritura del sistema. Utilizado por **todos** los módulos de dominio. | SCRUM-7 (F1.2) |
| Seguridad y Control de Acceso | Autenticación, control de acceso basado en roles (RBAC) y cifrado de datos sensibles en tránsito y en reposo. | SCRUM-7 (F1.3), SCRUM-12 (F6.4) |
| Integración de Pagos (PCI-DSS) | Capa de integración con la pasarela de pagos externa certificada PCI-DSS, usada por el módulo de Pagos y por cualquier flujo que requiera cobro. | SCRUM-11 (F5.1) |
| Notificaciones | Envío de confirmaciones y alertas (correo/notificación en app) usadas por Registro, Matching, Seguimiento y Pagos. | Transversal a SCRUM-7, 8, 11 |
| Logging y Monitoreo | Registro centralizado de errores y alertas de disponibilidad, usado por todos los módulos del sistema. | SCRUM-12 (F6.3) |
| Integración y Despliegue Continuo (CI/CD) | Infraestructura de build, pruebas automáticas y despliegue que da soporte al desarrollo de todos los módulos anteriores. | SCRUM-12 (F6.1, F6.2) |
| Control de Cambios y Gestión Documental | Proceso de gestión de versiones de documentos (SRS, SAD, DD, etc.), control de cambios en el equipo y seguimiento de entregables. | SCRUM-120 (F7.1) |

**Relación entre módulos transversales y módulos de dominio**

| Módulo Transversal | Módulos de Dominio a los que da soporte |
|---|---|
| Multi-tenancy | Todos los módulos de dominio (Usuarios, Solicitudes, Ejecución, Administración, Pagos). |
| Seguridad y Control de Acceso | Todos los módulos de dominio. |
| Integración de Pagos (PCI-DSS) | Pagos y Facturación. |
| Notificaciones | Gestión de Usuarios, Solicitudes y Matching, Pagos y Facturación. |
| Logging y Monitoreo | Todos los módulos de dominio. |
| CI/CD | Todos los módulos de dominio (indirectamente, como infraestructura de entrega). |
| Control de Cambios y Gestión Documental | Todos los módulos del proyecto (documentación y proceso). |

---

## 5. Requisitos Específicos

Los requisitos funcionales se agrupan según las épicas definidas en el backlog del proyecto (Jira, proyecto SCRUM). A su vez, dentro de cada épica, las historias de usuario se agrupan en **Features** (agrupaciones lógicas registradas como etiquetas en Jira). Cada requisito conserva la trazabilidad con su historia de usuario de origen mediante su identificador de Jira.

### 5.1 Features del Sistema (Agrupaciones Funcionales)

A continuación se listan las **23 Features** identificadas en el backlog, agrupadas por épica, junto con los requisitos funcionales e historias de usuario que cada una contiene.

#### Épica 1 — Fundación del Sistema y Arquitectura Multi-tenant (SCRUM-7)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F1.1 — Registro y Autenticación | Registro de Cliente y Técnico/Proveedor, e inicio de sesión seguro para todos los roles. | RF-01, RF-02, RF-03 | SCRUM-21, 22, 23 |
| F1.2 — Arquitectura Multi-tenant | Aislamiento lógico de datos entre tenants mediante *tenant_id*. | RF-04 | SCRUM-24 |
| F1.3 — Roles y Permisos | Definición y validación de permisos diferenciados por rol (RBAC). | RF-05 | SCRUM-25 |
| F1.4 — Registro de Empresas | Registro de cuentas corporativas asociadas a un tenant propio. | RF-06 | SCRUM-26 |

#### Épica 2 — Motor de Matching y Experiencia del Cliente (SCRUM-8)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F2.1 — Solicitud de Servicio | Creación de solicitudes de servicio técnico por Cliente y Empresa, respuesta a la cotización y cancelación antes de iniciar el servicio. | RF-07, RF-08, RF-35, RF-36 | SCRUM-27, 28 (RF-35 y RF-36: historias por crear) |
| F2.2 — Motor de Matching | Asignación automática de técnico y aceptación/rechazo de la solicitud. | RF-09, RF-10 | SCRUM-29, 30 |
| F2.3 — Seguimiento del Servicio | Visualización en tiempo real del estado de la solicitud. | RF-11 | SCRUM-31 |
| F2.4 — Calificación del Servicio | Calificación del servicio por parte del Cliente al finalizar. | RF-12 | SCRUM-32 |

#### Épica 3 — Herramientas para el Técnico y Proveedores (SCRUM-9)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F3.1 — Perfil y Disponibilidad | Configuración de disponibilidad y zona de cobertura del Técnico. | RF-13 | SCRUM-33 |
| F3.2 — Gestión de Servicios Asignados | Consulta de detalle, cotización y cierre (completado, con evidencia fotográfica obligatoria) de una solicitud asignada. | RF-14, RF-15, RF-34 | SCRUM-34, 35 (RF-34: historia por crear) |
| F3.3 — Gestión de Equipo (Proveedor) | Administración del equipo de técnicos por parte de un Proveedor. | RF-16 | SCRUM-36 |
| F3.4 — Historial de Servicios | Consulta del historial de solicitudes atendidas por el Técnico. | RF-17 | SCRUM-37 |

#### Épica 4 — Panel Administrativo y Operaciones (SCRUM-10)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F4.1 — Dashboard Operativo | Panel de solicitudes activas por tenant y por estado. | RF-18 | SCRUM-38 |
| F4.2 — Gestión de Técnicos/Proveedores | Aprobación, rechazo y suspensión de técnicos y proveedores. | RF-19, RF-20 | SCRUM-39, 40 |
| F4.3 — Gestión de Tenants | Administración (activar/desactivar) de tenants. | RF-21 | SCRUM-41 |

#### Épica 5 — Procesamiento de Pagos PCI-DSS y Facturación (SCRUM-11)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F5.1 — Cobro del Servicio | Pago del servicio con tarjeta por Cliente y por Empresa (PCI-DSS). | RF-22, RF-23 | SCRUM-42, 43 |
| F5.2 — Facturación | Generación automática de comprobante/factura al confirmar el pago. | RF-24 | SCRUM-44 |
| F5.3 — Registro de Pagos Recibidos | Consulta de pagos recibidos por Técnico/Proveedor. | RF-25 | SCRUM-45 |

#### Épica 6 — Infraestructura y Preparación para Producción (SCRUM-12)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F6.1 — Integración y Despliegue Continuo | Pipeline de CI/CD con build y pruebas automáticas antes de desplegar. | RF-26 | SCRUM-46 |
| F6.2 — Pruebas del Flujo Crítico | Prueba automatizada end-to-end del flujo Cliente→Pago→Calificación. | RF-27 | SCRUM-47 |
| F6.3 — Monitoreo y Logging | Registro centralizado de errores y alertas básicas de caída de servicio. | RF-28 | SCRUM-48 |
| F6.4 — Seguridad de Datos | Cifrado de datos sensibles en tránsito y en reposo. | RF-29 | SCRUM-49 |

#### Épica 7 — Control de Cambios - Equipo (SCRUM-120)

| Feature (Jira label) | Descripción | Requisitos | HU (Jira) |
|---|---|---|---|
| F7.1 — Gestión Documental | Creación, revisión y versionado de documentos clave del proyecto (SRS, SAD, DD, Infraestructura, Herramientas) y seguimiento de entregas. | RNF-13 | SCRUM-121 a SCRUM-133 |

**Detalle de las 13 tareas de F7.1 (estado en Jira, consultado el 23 de septiembre de 2026):**

| Jira | Tarea | Responsable | Estado |
|---|---|---|---|
| SCRUM-121 | SRS Versión 2 | Angy Bautista | ✅ Finalizado |
| SCRUM-122 | Herramientas y Políticas v2 | Sebastian Sánchez Olaya | ✅ Finalizado |
| SCRUM-123 | DD | Juan Diego Rojas Vargas | ✅ Finalizado |
| SCRUM-124 | Presentación de los documentos pedidos por el profe | kathe | ✅ Finalizado |
| SCRUM-125 | SAD V1 | Sebastian Sánchez Olaya | ✅ Finalizado |
| SCRUM-126 | Mockups - Versión 1 | Angy Bautista | ✅ Finalizado |
| SCRUM-127 | Presentación de la Entrega Sprint 2 | Jorge Fortich | ✅ Finalizado |
| SCRUM-128 | Documentos de herramientas, políticas y lineamientos V3 | Sebastian Sánchez Olaya | ✅ Finalizado |
| SCRUM-129 | Documento de requerimientos SRS V3 | Angy Bautista | ✅ Finalizado |
| SCRUM-130 | Documento SAD V2 | Sebastian Sánchez Olaya | ✅ Finalizado |
| SCRUM-131 | Documento DD V2 | joseval2910 (Jose Eduardo) | ✅ Finalizado |
| SCRUM-132 | Documento de Infraestructura V1 | Sebastian Sánchez Olaya | ✅ Finalizado |
| SCRUM-133 | Documento de diseño SDD V1 | Jorge Fortich | ✅ Finalizado |

### 5.2 Requisitos Funcionales

#### Épica 1 — Fundación del Sistema y Arquitectura Multi-tenant (SCRUM-7)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-01 | El sistema debe permitir a un Cliente registrarse con correo y contraseña, validando formato y unicidad del correo. | Alta | F1.1 | SCRUM-21 |
| RF-02 | El sistema debe permitir a un Técnico/Proveedor registrarse aportando datos y documentos básicos, quedando en estado "Pendiente de verificación". | Alta | F1.1 | SCRUM-22 |
| RF-03 | El sistema debe permitir el inicio de sesión seguro para todos los roles, con bloqueo temporal tras intentos fallidos repetidos. | Alta | F1.1 | SCRUM-23 |
| RF-04 | El sistema debe aislar lógicamente los datos de cada tenant mediante un identificador de tenant (*tenant_id*) obligatorio en cada registro. | Alta | F1.2 | SCRUM-24 |
| RF-05 | El sistema debe restringir el acceso a funciones y datos según el rol del usuario autenticado (Cliente, Técnico, Proveedor, Admin, Empresa). | Alta | F1.3 | SCRUM-25 |
| RF-06 | El sistema debe permitir a una Empresa registrar una cuenta corporativa asociada a un tenant propio. | Alta | F1.4 | SCRUM-26 |

#### Épica 2 — Motor de Matching y Experiencia del Cliente (SCRUM-8)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-07 | El sistema debe permitir a un Cliente crear una solicitud de servicio indicando categoría, dirección y descripción. | Alta | F2.1 | SCRUM-27 |
| RF-08 | El sistema debe permitir a una Empresa registrar solicitudes de servicio para sus sedes corporativas. | Alta | F2.1 | SCRUM-28 |
| RF-09 | El sistema debe asignar automáticamente un técnico disponible y cercano a cada solicitud (motor de matching). | Alta | F2.2 | SCRUM-29 |
| RF-10 | El sistema debe permitir al Técnico aceptar o rechazar una solicitud asignada, reasignando automáticamente en caso de rechazo o falta de respuesta. | Alta | F2.2 | SCRUM-30 |
| RF-11 | El sistema debe mostrar al Cliente el estado actual de su solicitud en tiempo real (mínimo 4 estados). | Alta | F2.3 | SCRUM-31 |
| RF-12 | El sistema debe permitir al Cliente calificar el servicio (1 a 5) una vez la solicitud esté en estado "Completado". | Media | F2.4 | SCRUM-32 |
| RF-35 | El sistema debe permitir al Cliente aceptar o rechazar la cotización de su solicitud. Una solicitud admite como máximo 3 cotizaciones, y una cotización sin respuesta antes de su vencimiento cancela la solicitud. | Alta | F2.1 | Por crear |
| RF-36 | El sistema debe permitir al Cliente, o al administrador de su tenant, cancelar una solicitud mientras el servicio no haya iniciado, registrando el motivo de la cancelación. | Media | F2.1 | Por crear |

#### Épica 3 — Herramientas para el Técnico y Proveedores (SCRUM-9)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-13 | El sistema debe permitir al Técnico configurar su disponibilidad y zona de cobertura. | Alta | F3.1 | SCRUM-33 |
| RF-14 | El sistema debe mostrar al Técnico el detalle completo de una solicitud asignada. | Alta | F3.2 | SCRUM-34 |
| RF-15 | El sistema debe permitir al Técnico marcar una solicitud como completada únicamente después de adjuntar al menos una evidencia fotográfica del trabajo realizado, habilitando el proceso de pago. | Alta | F3.2 | SCRUM-35 |
| RF-16 | El sistema debe permitir al Proveedor registrar y administrar su equipo de técnicos. | Media | F3.3 | SCRUM-36 |
| RF-17 | El sistema debe mostrar al Técnico el historial de solicitudes atendidas. | Media | F3.4 | SCRUM-37 |
| RF-34 | El sistema debe permitir al Técnico asignado emitir una cotización de mano de obra y materiales para la solicitud; el servicio no puede iniciar sin una cotización aceptada por el Cliente. | Alta | F3.2 | Por crear |

#### Épica 4 — Panel Administrativo y Operaciones (SCRUM-10)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-18 | El sistema debe mostrar al Admin un panel con las solicitudes activas por tenant y por estado. | Alta | F4.1 | SCRUM-38 |
| RF-19 | El sistema debe permitir al Admin aprobar o rechazar el registro de nuevos técnicos y proveedores. | Alta | F4.2 | SCRUM-39 |
| RF-20 | El sistema debe permitir al Admin suspender la cuenta de un técnico. | Media | F4.2 | SCRUM-40 |
| RF-21 | El sistema debe permitir al Admin administrar (activar/desactivar) tenants. | Media | F4.3 | SCRUM-41 |

#### Épica 5 — Procesamiento de Pagos PCI-DSS y Facturación (SCRUM-11)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-22 | El sistema debe permitir al Cliente pagar el servicio con tarjeta mediante un proveedor de pagos certificado PCI-DSS. | Alta | F5.1 | SCRUM-42 |
| RF-23 | El sistema debe permitir a una Empresa pagar servicios a nombre de su cuenta corporativa. | Alta | F5.1 | SCRUM-43 |
| RF-24 | El sistema debe generar automáticamente un comprobante/factura al confirmarse un pago. | Alta | F5.2 | SCRUM-44 |
| RF-25 | El sistema debe mostrar al Técnico/Proveedor el registro de pagos recibidos por sus servicios. | Media | F5.3 | SCRUM-45 |

#### Épica 6 — Infraestructura y Preparación para Producción (SCRUM-12)

| ID | Descripción | Prior. | Feature | Jira |
|---|---|---|---|---|
| RF-26 | El sistema debe contar con un pipeline de CI/CD que ejecute build y pruebas automáticas antes de desplegar. | Alta | F6.1 | SCRUM-46 |
| RF-27 | El sistema debe contar con una prueba automatizada end-to-end del flujo crítico (solicitud → pago → calificación). | Alta | F6.2 | SCRUM-47 |
| RF-28 | El sistema debe registrar logs centralizados de errores y contar con alertas básicas de caída de servicio. | Media | F6.3 | SCRUM-48 |
| RF-29 | El sistema debe cifrar los datos sensibles en tránsito (HTTPS/TLS) y en reposo (contraseñas con hash seguro). | Alta | F6.4 | SCRUM-49 |

### 5.3 Requisitos No Funcionales

**Seguridad**
- **RNF-01** El sistema debe cumplir el estándar PCI-DSS en el manejo de información de pagos, delegando el almacenamiento de datos de tarjeta a un proveedor certificado.
- **RNF-02** Toda comunicación cliente-servidor debe realizarse mediante HTTPS/TLS.
- **RNF-03** Las contraseñas deben almacenarse utilizando un algoritmo de *hashing* seguro (nunca en texto plano).
- **RNF-04** El sistema debe registrar en log todo intento de acceso no autorizado (error 403) a recursos restringidos por rol.

**Rendimiento**
- **RNF-05** El motor de matching debe iniciar el proceso de asignación en menos de 60 segundos desde la creación de la solicitud (ambiente de prueba).
- **RNF-06** Los cambios de estado de una solicitud deben reflejarse en la interfaz del Cliente en menos de 1 minuto.

**Disponibilidad y Confiabilidad**
- **RNF-07** El sistema debe contar con un ambiente de *staging* separado de producción para las pruebas funcionales, de aceptación y de seguridad (E2E, UAT, OWASP ZAP). Las pruebas de carga y rendimiento (k6) se ejecutan sobre la infraestructura de producción, contra un tenant de prueba dedicado y en una ventana de mantenimiento programada sin usuarios activos, porque el proyecto no cuenta con una VM adicional para staging de carga (K10 del SAD, hardware fijo de 7 VMs).
- **RNF-08** Un fallo en las pruebas automatizadas funcionales, de aceptación o de seguridad del flujo crítico, ejecutadas antes del despliegue (ver RNF-07), debe bloquear el despliegue a producción. La prueba de carga (k6) se ejecuta inmediatamente después del despliegue, dentro de la misma ventana de mantenimiento: si no cumple el umbral definido, el despliegue se revierte (`kubectl rollout undo`) antes de habilitar tráfico real, porque no existe una segunda instancia de producción donde probar la carga antes de desplegar (K10).

**Escalabilidad y Arquitectura Multi-tenant**
- **RNF-09** La arquitectura debe permitir incorporar nuevos tenants (nuevas empresas o zonas geográficas) sin afectar el aislamiento de datos de los tenants existentes.
- **RNF-10** Las consultas a base de datos deben filtrar automáticamente por el *tenant_id* de la sesión activa.

**Usabilidad**
- **RNF-11** El flujo de creación de una solicitud de servicio (Cliente) debe poder completarse en un máximo de 3 pasos.
- **RNF-12** Las interfaces de Cliente, Técnico y Admin deben adaptarse a dispositivos móviles y de escritorio (diseño responsivo).

**Gestión del Proyecto y Documentación**
- **RNF-13** El equipo debe mantener un control de versiones de todos los documentos entregables (SRS, SAD, DD, etc.) y utilizar Jira para el seguimiento de tareas y el control de cambios.

### 5.4 Requisitos de Interfaces Externas

- **RIE-01 Pasarela de pagos:** el sistema debe integrarse con un proveedor de pagos externo certificado PCI-DSS mediante una API segura (tokenización de tarjeta).
- **RIE-02 Servicio de geolocalización:** el sistema debe integrarse con un servicio de mapas/geocodificación para validar direcciones y calcular cercanía entre Cliente y Técnico.
- **RIE-03 Servicio de notificaciones:** el sistema debe contar con un canal (correo electrónico y/o notificación en aplicación) para confirmar registro, asignación de técnico y cambios de estado.

---

## 6. Matriz Comparativa

Con el fin de justificar el enfoque del producto, se comparó QUICKPATCH frente a plataformas de servicios para el hogar con presencia en Colombia y/o reconocidas internacionalmente: **Timbrit** y **Chepe & Pepe** (ambas colombianas, con presencia en Bogotá), **Hogaru** (servicio recurrente de aseo, Colombia) y **TaskRabbit** (referente internacional). La información de los competidores se basa en fuentes públicas (ver Referencias) y puede no reflejar funcionalidades internas no documentadas públicamente; donde no fue posible confirmar un dato, se marca como "No especificado".

| Criterio | QUICKPATCH | Timbrit | Chepe & Pepe | Hogaru | TaskRabbit |
|---|---|---|---|---|---|
| Cobertura en Bogotá | Sí | Sí | Sí | Sí | No (sin presencia confirmada en Colombia) |
| Cuentas corporativas multi-tenant (B2B) | **Sí** | No especificado | No especificado | No (enfoque residencial) | No especificado |
| Asignación automática de técnico (matching) | **Sí** | No especificado | No especificado | No (asignación por suscripción) | No especificado |
| Pago integrado en la app (pasarela propia) | Sí (PCI-DSS) | No especificado | No especificado | Sí | Sí |
| Seguimiento en tiempo real del servicio | **Sí** | No especificado | No especificado | No aplica | No especificado |
| Calificación del servicio en la app | Sí | Sí (reseñas) | No especificado | Sí | Sí |
| Modelo de servicio | Bajo demanda puntual | Bajo demanda puntual | Bajo demanda puntual | Suscripción recurrente | Bajo demanda puntual |
| Aislamiento de datos por cliente/empresa (multi-tenant) | **Sí** | No especificado | No especificado | No especificado | No especificado |

**Conclusión de la comparación:** el diferenciador central de QUICKPATCH frente a las alternativas identificadas es la combinación de **arquitectura multi-tenant orientada a cuentas corporativas (B2B)** junto con un **motor de matching automático en tiempo real**, dos capacidades que no están confirmadas públicamente en los competidores analizados, los cuales se orientan principalmente al consumidor residencial individual.

---

## 7. Apéndices

### 7.1 Matriz de Trazabilidad Épicas — Features — Requisitos

| Épica (Jira) | Features | Requisitos | Historias de Usuario (Jira) |
|---|---|---|---|
| SCRUM-7 — Fundación y Multi-tenant | F1.1 – F1.4 | RF-01 – RF-06 | SCRUM-21 a SCRUM-26 |
| SCRUM-8 — Matching y Experiencia Cliente | F2.1 – F2.4 | RF-07 – RF-12, RF-35, RF-36 | SCRUM-27 a SCRUM-32 |
| SCRUM-9 — Herramientas Técnico/Proveedor | F3.1 – F3.4 | RF-13 – RF-17, RF-34 | SCRUM-33 a SCRUM-37 |
| SCRUM-10 — Panel Administrativo | F4.1 – F4.3 | RF-18 – RF-21 | SCRUM-38 a SCRUM-41 |
| SCRUM-11 — Pagos PCI-DSS y Facturación | F5.1 – F5.3 | RF-22 – RF-25 | SCRUM-42 a SCRUM-45 |
| SCRUM-12 — Infraestructura y QA | F6.1 – F6.4 | RF-26 – RF-29 | SCRUM-46 a SCRUM-49 |
| SCRUM-120 — Control de Cambios - Equipo | F7.1 | RNF-13 | SCRUM-121 a SCRUM-133 |

### 7.2 Resumen de Requisitos Funcionales

El presente MVP contempla un total de **32 requisitos funcionales**, agrupados en **23 Features**, organizados en **7 módulos de dominio** y soportados por **7 módulos transversales** (incluyendo la nueva épica de Control de Cambios), distribuidos en **7 épicas**. Todos los requisitos están priorizados como *Alta* o *Media* por ser esenciales para demostrar el ciclo completo del negocio: registro y autenticación multi-tenant, solicitud de servicio, asignación automática, cotización, ejecución técnica con evidencia fotográfica, pago seguro y calificación. La numeración salta de RF-29 a RF-34 porque RF-30 a RF-33 (versiones 2.x, control de cambios) se retiraron en la versión 3.0 al reemplazarse por RNF-13, y sus números no se reutilizan. Adicionalmente, se incluye un requisito no funcional (RNF-13) para la gestión documental y control de cambios del proyecto.

**Estado de avance de la Épica 7 (consultado en Jira el 23 de septiembre de 2026):** las 13 tareas (SCRUM-121 a SCRUM-133) figuran como Finalizadas, incluidas las versiones V2 y V3 de los documentos (SRS, SAD, DD, Herramientas, Infraestructura y SDD) y la presentación de la entrega del Sprint 2.

### 7.3 Control de Versiones del Documento

| Versión | Fecha | Descripción del cambio |
|---|---|---|
| 1.0 | 15 ago 2026 | Versión inicial del SRS, derivada del backlog MVP registrado en Jira (proyecto SCRUM). |
| 2.0 | 31 ago 2026 | Se incorpora el nombre del producto (QUICKPATCH); se agrega la sección de Features del sistema y su trazabilidad a requisitos e historias de usuario. |
| 2.0 | (edición posterior) | Se elimina el alcance del proyecto ya que no corresponde a este documento. Se agrega el Capítulo de Usuarios del Sistema (independiente); se agrega el Capítulo de Módulos del Producto (dominio y transversales); se agrega la Matriz Comparativa frente a plataformas similares; se reincorpora la sección de Referencias. |
| 3.0 | (edición posterior) | Se añade la Épica 7 (Control de cambios - equipo) con su Feature F7.1 y el requisito no funcional RNF-13. Se actualiza la matriz de trazabilidad y el resumen de requisitos para incluir la nueva épica y sus historias de usuario asociadas. Se renombra el producto de QUICKPATCH a **CODEBRIDGE**. |
| 3.1 | 14 sep 2026 | Conversión a Markdown. Se corrige la tabla de la Feature F7.1: se agregan SCRUM-126 y SCRUM-127, que estaban omitidos, completando el rango real de 13 tareas (SCRUM-121 a SCRUM-133) verificado directamente en Jira. Se agrega el detalle de estado y responsable de cada tarea de la Épica 7. |
| 3.2 | *(este documento)* | Se alinea con el SAD v2.11 y el DD v2.2 (dependencia DEP-10 del DD). RF-15 vuelve a exigir al menos una evidencia fotográfica antes de completar el servicio (driver D7 del SAD). Se agregan RF-34 (cotización del técnico), RF-35 (respuesta del cliente a la cotización) y RF-36 (cancelación antes de iniciar el servicio), que el SAD define en el ciclo de vida del servicio (sección 7.4); sus historias en Jira están por crear. RNF-07 y RNF-08 recuperan la redefinición del modelo de staging acordada en las versiones 2.3 y 2.4, alineada con el Documento de Infraestructura. El producto vuelve a llamarse QUICKPATCH en todo el documento: CODEBRIDGE es el nombre del curso, no del producto. Se actualiza el estado de la Épica 7 según Jira (SCRUM-127 corresponde a la presentación del Sprint 2). |

---

## Referencias

1. IEEE Std 830-1998, *IEEE Recommended Practice for Software Requirements Specifications*, Institute of Electrical and Electronics Engineers.
2. PCI Security Standards Council, *Payment Card Industry Data Security Standard (PCI-DSS)*. Disponible en: [https://www.pcisecuritystandards.org](https://www.pcisecuritystandards.org)
3. Backlog de producto QUICKPATCH — Versión 1 (MVP), proyecto **SCRUM** en Jira (Atlassian), Épicas SCRUM-7 a SCRUM-12 y SCRUM-120, Historias SCRUM-21 a SCRUM-49 y SCRUM-121 a SCRUM-133.
4. Timbrit — Plataforma colombiana de contratación de profesionales para el hogar. Disponible en: [https://www.timbrit.com.co/](https://www.timbrit.com.co/)
5. La República, *Conozca las aplicaciones disponibles que le ayudan con todas las tareas del hogar*, 2019. Disponible en: [https://www.larepublica.co/internet-economy/conozca-las-aplicaciones-disponibles-que-le-ayudan-con-todas-las-tareas-del-hogar-2936445](https://www.larepublica.co/internet-economy/conozca-las-aplicaciones-disponibles-que-le-ayudan-con-todas-las-tareas-del-hogar-2936445)
6. La República, *Conozca cinco aplicaciones que le ayudan a reparar y asear su hogar*. Disponible en: [https://www.larepublica.co/infraestructura/conozca-cinco-aplicaciones-que-le-ayudan-a-reparar-y-asear-su-hogar-2897117](https://www.larepublica.co/infraestructura/conozca-cinco-aplicaciones-que-le-ayudan-a-reparar-y-asear-su-hogar-2897117)
7. TaskRabbit — Aplicación de reserva de servicios para el hogar (Google Play). Disponible en: [https://play.google.com/store/apps/details?id=com.taskrabbit.droid.consumer](https://play.google.com/store/apps/details?id=com.taskrabbit.droid.consumer)
8. Google Labs, *Stitch — AI-powered UI design tool*. Disponible en: [https://stitchwith.google.com](https://stitchwith.google.com)
