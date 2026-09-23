# Documento de Arquitectura de Software (SAD) V2.11 — QUICKPATCH

---

## 1. Drivers y Killers

Esta sección identifica las fuerzas que determinan las decisiones de arquitectura del sistema. Los _drivers_ son las necesidades de negocio y técnicas que impulsan la arquitectura hacia cierta forma. Los _killers_ son las restricciones que limitan o invalidan alternativas de diseño.

### 1.1 Drivers

Cada driver es una necesidad que da forma a la arquitectura. La columna "Origen" indica de dónde sale, con referencia a los requisitos del SRS o a una decisión del equipo; la sección 1.4 traza cada driver hasta los escenarios de calidad y las decisiones de arquitectura que motiva.

|ID|Driver|Origen|Implicación arquitectónica|
|---|---|---|---|
|D1|Asignar automáticamente al técnico más cercano y disponible|RF-09, RF-10, RNF-05; SRS, sección 2.2 (función 3)|Consultas geoespaciales (PostGIS), estado de disponibilidad por técnico y reasignación automática ante rechazo o falta de respuesta. La asignación es inmediata: no hay agendamiento por horario|
|D2|Seguimiento del servicio en tiempo real|RF-11, RNF-06; diferenciador frente a la competencia (SRS, sección 6)|Cada cambio de estado se publica como evento y se entrega al cliente a través del API Gateway, sin que el cliente consulte por su cuenta|
|D3|Reputación del técnico basada en calificaciones|RF-12; SRS, sección 2.2 (función 6)|Ranking Service separado que consume los eventos de evaluación. El ranking de proveedores de materiales queda fuera del MVP|
|D4|Comprobante automático a nombre de la plataforma (merchant of record)|RF-22, RF-23, RF-24; SRS, sección 2.2 (función 5)|Payments emite la factura al consumir `payment.approved` y la plataforma es la única emisora frente al cliente. No incluye liquidación a técnicos ni contabilidad (K1)|
|D5|Multi-tenancy y soporte a múltiples empresas oferentes|RF-04, RF-06, RNF-09, RNF-10; modelo de negocio B2B2E|Aislamiento de datos por tenant desde el diseño inicial|
|D6|Trazabilidad ante reclamaciones|RF-28; necesidad de resolver disputas entre cliente y técnico|Registro histórico de eventos del ciclo de vida del servicio|
|D7|Evidencia fotográfica obligatoria para completar un servicio|Decisión confirmada con el equipo (12 sep 2026)|Almacenamiento de objetos en infraestructura propia (MinIO en VM7), tabla `service_evidence` y bloqueo de la transición a `completado` sin al menos una foto|

> Los requisitos de pago con PCI-DSS, que antes eran el driver D4, pasan al killer K2: son una restricción, no una fuerza que impulse el diseño. La numeración anterior saltaba del D5 al D8 sin que existieran D6 ni D7; en esta versión los drivers se numeran de forma consecutiva (equivalencias en la sección 9, versión 2.10).

### 1.2 Killers

Un killer es una restricción dura: descarta alternativas de diseño que de otro modo serían válidas. Los recortes de alcance que no descartan alternativas de arquitectura se listan aparte, en la sección 1.3.

|ID|Killer|Origen|Qué prohíbe|
|---|---|---|---|
|K1|No es un ERP|Alcance del MVP definido por el equipo con el Product Owner (SRS, sección 2.2: las funciones del producto no incluyen contabilidad ni nómina)|Contabilidad general, nómina legal, liquidación y pagos a técnicos, inventarios. La facturación del modelo merchant of record (D4) se limita a emitir el comprobante de cada pago confirmado|
|K2|Cumplimiento PCI-DSS|La norma PCI-DSS prohíbe almacenar el código de seguridad (CVV) tras la autorización; el equipo decide además no almacenar el número de tarjeta (PAN) para reducir su alcance de cumplimiento|Almacenar o registrar en logs el CVV o el número de tarjeta, y que los datos de tarjeta pasen por el backend propio: la captura va directo a la pasarela certificada, que devuelve solo un token|
|K3|Tiempo académico de aproximadamente 3 meses|Calendario del curso|Módulos y funcionalidades que no quepan en los sprints del curso; lo que no entra se declara fuera de alcance (sección 1.3)|
|K4|*Retirado en la versión 2.9*|—|Era un recorte de alcance, no una restricción de arquitectura: pasa a la sección 1.3 como FA1|
|K5|Sin presupuesto: producción corre en infraestructura propia|El proyecto no tiene presupuesto; las VMs las aprovisiona la universidad|Que el cómputo o los datos que atienden usuarios en producción corran fuera de las 7 VMs del laboratorio (bases de datos, colas o almacenamiento administrados en la nube, aunque tengan capa gratuita) y cualquier servicio con costo. Se permiten herramientas gratuitas de desarrollo y CI que no guardan datos de usuarios (GitHub Actions, GitHub Container Registry). La única dependencia externa de producción es la pasarela de pagos, exigida por K2|
|K6|Operación limitada a Bogotá D.C.|Alcance geográfico del MVP|Otras ciudades o países: multi-moneda, varios idiomas, despliegue multi-región y normativa extranjera. No exime de la normativa colombiana que aplica al producto (protección de datos personales, Ley 1581 de 2012, y facturación)|
|K7|Sin operación 24/7|Equipo de estudiantes, sin turnos ni guardias|Failover automático entre máquinas, despliegue multi-zona y cualquier diseño que dependa de que alguien atienda incidentes fuera del horario de trabajo. El reinicio de pods y el rollback de k3s dentro de una misma VM sí se permiten|
|K8|*Retirado en la versión 2.9*|—|Era un recorte de alcance, no una restricción de arquitectura: pasa a la sección 1.3 como FA2|
|K9|Red privada del laboratorio, sin dominio público|Las 7 VMs viven en la red privada de la universidad (`10.43.x.x`) y no son alcanzables desde Internet (Documento de Infraestructura, sección 11)|Exponer servicios directamente a Internet, usar dominio y DNS públicos, certificados de una autoridad pública (Let's Encrypt) y depender de llamadas entrantes de terceros, como los webhooks de la pasarela de pagos, sin un mecanismo de acceso aprobado por el equipo|
|K10|Hardware fijo: 7 VMs idénticas|Aprovisionadas por el laboratorio: 4 vCPU, 11 GiB de RAM y 68 GB de disco cada una (Documento de Infraestructura, sección 3)|Diseños que requieran más máquinas o capacidad distinta por rol, como un clúster de Kubernetes de varios nodos o una VM por microservicio; los recursos de cada servicio se ajustan a esta capacidad (Documento de Infraestructura, sección 5.6)|
|K11|Una sola persona administra la infraestructura|Estructura del equipo: un único rol de DevOps|Configuración manual o no reproducible de las VMs: toda configuración se hace con Ansible desde un solo inventario (sección 5.4)|

> **Consecuencia de K9 sin resolver:** los técnicos en campo con datos móviles y los webhooks de la pasarela de pagos necesitan alcanzar el sistema desde fuera de la red del laboratorio, lo que K9 hoy impide. El equipo debe decidir el mecanismo: (a) limitar la demostración a la red del campus y reemplazar los webhooks por consulta periódica del estado del pago; (b) exponer solo el API Gateway mediante un túnel gratuito, evaluado contra K5; o (c) solicitar a la universidad una IP pública o una regla de NAT hacia VM1.

> **Sobre K7 y la recuperación en 12–24 h:** la ventana de recuperación manual de 12 a 24 horas (AC5-E1) no la impone K7: es el objetivo que el equipo fijó para operar dentro de K7, sin guardias.

### 1.3 Fuera de alcance

Recortes del MVP derivados de K3. No son restricciones de arquitectura: el diseño no los impide, simplemente no se construyen en esta versión.

|ID|Fuera de alcance|Consecuencia|
|---|---|---|
|FA1|Modo offline o manejo de conectividad intermitente (antes K4)|La aplicación móvil asume conexión con el sistema; no hay sincronización diferida|
|FA2|Sitio público con SEO (antes K8)|El frontend web se limita al panel administrativo: sin páginas públicas indexables ni integración con redes sociales|
|FA3|Chat y módulo de reclamos dentro de la aplicación (roadmap 1.x)|Las disputas se resuelven fuera de la aplicación, con la trazabilidad de D6 como evidencia|

### 1.4 Trazabilidad de los drivers

Cada driver se sigue desde los requisitos que lo originan hasta los escenarios de calidad que lo miden y las decisiones de arquitectura que motiva.

|Driver|Requisitos (SRS)|Escenarios de calidad|Decisiones de arquitectura|
|---|---|---|---|
|D1 Asignación automática|RF-09, RF-10, RF-13, RNF-05, RIE-02|AC1-E1, AC2-E1, AC2-E3, AC2-E4, AC9-E4|ADR-004 (PostgreSQL + PostGIS), ADR-006 (Kafka)|
|D2 Seguimiento en tiempo real|RF-11, RNF-06, RIE-03|AC2-E3, AC4-E9|ADR-006 (Kafka), ADR-007 (Outbox)|
|D3 Reputación del técnico|RF-12, RF-19, RF-20|AC4-E6, AC9-E2|ADR-003 (Ranking Service independiente)|
|D4 Comprobante a nombre de la plataforma|RF-22, RF-23, RF-24, RIE-01|AC3-E1, AC9-E5|ADR-009 (tokenización de pagos)|
|D5 Multi-tenancy|RF-04, RF-06, RF-21, RNF-09, RNF-10|AC6-E2, AC8-E1|ADR-005 (shared-schema con RLS)|
|D6 Trazabilidad ante reclamaciones|RF-28, RNF-04|AC6-E5, AC6-E6, AC6-E7, AC7-E4|ADR-006 (Kafka), ADR-007 (Outbox)|
|D7 Evidencia fotográfica|RF-15 (ver nota)|AC6-E5|Sin ADR propio: la decisión de almacenamiento (MinIO en VM7) está en la sección 5.1|

> **Nota sobre D7:** la exigencia de evidencia fotográfica para completar un servicio fue confirmada por el equipo y está en el DD (`service_evidence`), pero la versión actual del SRS no la incluye en RF-15. Hasta que el SRS se alinee, su origen es la decisión del equipo.

## 2. Atributos de Calidad

Los atributos de calidad expresan, en términos medibles, las propiedades que el sistema debe cumplir. Cada atributo seleccionado está sustentado por uno o más de los drivers o killers definidos en la sección anterior.

Los 9 ACs mapean 1 a 1 con las **9 características de ISO/IEC 25010:2023**, en su orden canónico. Donde QUICKPATCH tenía dos ACs distintos sustentando en el fondo la misma característica ISO, se fusionaron (ver nota de fusión debajo de la tabla); donde una característica no tenía ningún AC, se agregó.

|ID|Atributo (característica ISO)|Sustentado por|
|---|---|---|
|AC1|Functional Suitability|SRS — requisitos funcionales, cumplimiento funcional del sistema|
|AC2|Performance Efficiency|D1, D2|
|AC3|Compatibility|D4, ISO/IEC 25010:2023 — RIE-01|
|AC4|Interaction Capability|ISO/IEC 25010:2023 — RNF-11, RNF-12|
|AC5|Reliability|K5, K7|
|AC6|Security|K2, D5, D6, D7|
|AC7|Maintainability|K3|
|AC8|Flexibility|D5, ISO/IEC 25010:2023 — ADR-002, RNF-09, RIE-02|
|AC9|Safety|D1, D3; decisión del equipo con el Product Owner: riesgo físico y patrimonial por fallos de la plataforma, ver sección 3.9|

**Fusiones aplicadas:**
- El AC de Seguridad (antes AC2) absorbe los escenarios de *Accountability* del antiguo AC de Trazabilidad (antes AC6) — ambos son, a nivel de característica ISO, **Security**. El escenario de idempotencia (antes AC6-E3) no es Accountability, así que se reclasificó dentro de **Reliability** (subcaracterística *Fault Tolerance*), no dentro de Security.
- El AC de Flexibilidad (antes AC9) absorbe los escenarios de *Adaptability*/*Scalability* del antiguo AC de Escalabilidad (antes AC4) — ambos son, a nivel de característica ISO, **Flexibility**. Un escenario de cada AC resultó redundante (medía literalmente lo mismo que otro ya existente) y se retiró en vez de mantenerse duplicado — ver sección 3.
- Un escenario del antiguo AC de Compatibilidad (antes AC8) en realidad es *Replaceability*, y se movió al AC de Flexibilidad — su sustento (RIE-02) se movió con él, hacia AC8.

> **Pendiente, y más relevante de lo que parece:** RIE-03 (servicio de notificaciones) sustentaba al antiguo AC de Compatibilidad junto con RIE-01 y RIE-02, y no tiene un AC propio que lo sustente formalmente. Esto no significa que el sistema de notificaciones esté fuera de todo escenario — AC2-E3 ya mide "menos de 7 segundos desde la creación de la solicitud hasta la notificación" y AC5-E5 usa "doble notificación" como ejemplo de efecto duplicado a evitar — sino que **una dependencia externa (RIE-03) queda dentro de un SLO comprometido (los 7 segundos de AC2-E3) sin que ningún escenario de interoperabilidad la cubra explícitamente**. Communication Service (que incluye Notifications) es uno de los 8 microservicios de dominio con VM propia (sección 5.1, 4.2.2) — no es un módulo transversal. Un escenario nuevo bajo *Co-existence* o *Functional Completeness* cerraría este hueco; *Interoperability* no serviría porque AC3-E1 ya la cubre y no sumaría al conteo de subcaracterísticas.

> AC1 y AC4 no derivan directamente de un driver o killer (sección 1): se sustentan en los requisitos del SRS y en los atributos genéricos de la norma **ISO/IEC 25010:2023**. AC3 y AC8 combinan un driver (D4 y D5) con la norma. AC9 (Safety) se sustenta en D1 y D3 y en la definición que el equipo acordó con el Product Owner de qué significa Safety para QUICKPATCH — ver sección 3.9.

---

## 3. Escenarios de Calidad

Cada escenario sigue la estructura de seis partes: fuente del estímulo, estímulo, ambiente, artefacto, respuesta y medida de la respuesta.

Cada escenario se clasifica en uno de tres tipos:

- **Uso**: operación normal del sistema, lo que ocurre en el día a día.
- **Cambio**: crecimiento o modificación esperada del sistema con el tiempo (nuevos tenants, nuevo código, nuevos servicios).
- **Fallo**: algo se rompe o deja de responder — cómo reacciona el sistema ante condiciones adversas o inesperadas.

### 3.1 AC1 — Functional Suitability

Característica agregada en esta versión (ver sección 2). Cubre si el sistema hace lo que promete (*Functional Correctness*), si ofrece las funciones necesarias (*Functional Completeness*) y si esas funciones son las apropiadas para las tareas del usuario (*Functional Appropriateness*). Cada una de las tres subcaracterísticas tiene un escenario.

**Escenario 1 — Filtro de especialidad en el matching** · _Tipo: Uso_ · _(Functional Correctness)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema (tras crear una solicitud)|
|Estímulo|El motor de matching busca técnicos candidatos para una categoría de servicio específica|
|Ambiente|Operación normal|
|Artefacto|Motor de matching — filtro de especialidad|
|Respuesta|Solo se consideran candidatos cuya especialidad registrada coincide con la categoría solicitada|
|Medida|0% de asignaciones a un técnico de especialidad distinta a la solicitada, verificable en `matching_attempts`|

**Escenario 2 — Cobertura de requisitos del MVP** · _Tipo: Uso_ · _(Functional Completeness)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de QA o evaluador|
|Estímulo|Verifica que el sistema implementa los requisitos funcionales del MVP antes de la entrega|
|Ambiente|Release candidate desplegado|
|Artefacto|Requisitos funcionales de prioridad Alta del SRS y sus pruebas de aceptación|
|Respuesta|Cada requisito tiene al menos una prueba de aceptación ejecutable, y el flujo crítico solicitud → pago → calificación (RF-27) se ejecuta completo|
|Medida|100% de los requisitos funcionales de prioridad Alta con una prueba de aceptación aprobada en el checklist de aceptación del release|

**Escenario 3 — Información suficiente para ejecutar el servicio** · _Tipo: Uso_ · _(Functional Appropriateness)_

|Parte|Contenido|
|---|---|
|Fuente|Técnico|
|Estímulo|Consulta el detalle de una solicitud que se le asignó (RF-14)|
|Ambiente|Operación normal|
|Artefacto|Pantalla de detalle de la solicitud|
|Respuesta|Muestra lo necesario para ejecutar el servicio sin contactar al cliente por otro canal|
|Medida|100% de las solicitudes asignadas muestran categoría, dirección y descripción del problema|

> **Pendiente:** la votación ATAM de los escenarios de este AC (requiere al cliente/stakeholders de negocio, no solo al equipo).

### 3.2 AC2 — Performance Efficiency

**Escenario 1 — Búsqueda de técnicos cercanos** · _Tipo: Uso_ · _(Time Behaviour)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente|
|Estímulo|Solicita técnicos cercanos disponibles|
|Ambiente|Horario de alta demanda|
|Artefacto|Mecanismo de búsqueda por ubicación y disponibilidad|
|Respuesta|Devuelve lista ordenada por distancia y ranking|
|Medida|Menos de 3 segundos para el 95% de las solicitudes|

**Escenario 2 — Consulta de historial** · _Tipo: Uso_ · _(Time Behaviour)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente o técnico|
|Estímulo|Consulta su historial de servicios|
|Ambiente|Operación normal, historial con múltiples solicitudes acumuladas|
|Artefacto|Módulo de gestión de solicitudes|
|Respuesta|Devuelve el historial paginado|
|Medida|Menos de 1.5 segundos para cargar una página de resultados|

**Escenario 3 — Procesamiento en segundo plano** · _Tipo: Uso_ · _(Time Behaviour)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema (tras la creación de una solicitud)|
|Estímulo|Se dispara la búsqueda de técnicos candidatos|
|Ambiente|Operación normal|
|Artefacto|Mecanismo de procesamiento asíncrono|
|Respuesta|Identifica y notifica a los técnicos candidatos|
|Medida|Menos de 7 segundos desde la creación de la solicitud hasta la notificación|

**Escenario 4 — Pico de carga inesperado** · _Tipo: Fallo_ · _(Capacity)_

|Parte|Contenido|
|---|---|
|Fuente|Carga de usuarios|
|Estímulo|Un pico de solicitudes supera la capacidad prevista (ej. campaña o promoción viral)|
|Ambiente|Evento de demanda no anticipado|
|Artefacto|Matching Service|
|Respuesta|El sistema degrada de forma controlada (cola de espera) en vez de caerse por completo|
|Medida|El sistema soporta hasta **150 solicitudes de matching concurrentes** (estimado como 3 veces una carga esperada de ~50 solicitudes concurrentes en hora pico) sin devolver errores ni caerse por completo; el tiempo de respuesta puede degradarse más allá de los 3 segundos definidos en el Escenario 1, pero el servicio sigue respondiendo|

**Escenario 5 — Uso de recursos bajo carga normal** · _Tipo: Uso_ · _(Resource Utilization)_

|Parte|Contenido|
|---|---|
|Fuente|Carga de usuarios|
|Estímulo|Operación sostenida en hora pico, con ~50 solicitudes de matching concurrentes (la carga esperada del Escenario 4)|
|Ambiente|Operación normal, 8 microservicios en VM3 (4 vCPU, 11 GiB de RAM)|
|Artefacto|Pods en k3s con los `requests` y `limits` del Documento de Infraestructura, sección 5.6|
|Respuesta|Cada servicio opera dentro de sus límites declarados y el consumo queda registrado|
|Medida|0 reinicios por OOMKilled y RAM total de VM3 menor o igual a 6.5 GiB (máximo teórico del presupuesto de recursos), medidos con `kubectl` y Grafana durante la prueba de carga con k6|

> **Nota:** el antiguo escenario "crecimiento del catálogo de técnicos" (Escalabilidad) medía literalmente el mismo umbral que el Escenario 1 (<3s) bajo crecimiento de datos en vez de carga concurrente — se retiró por redundante en vez de mantenerse como escenario aparte.

### 3.3 AC3 — Compatibility

**Escenario 1 — Integración con la pasarela de pagos externa** · _Tipo: Uso_ · _(Interoperability)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente o Empresa|
|Estímulo|Confirma el pago de un servicio|
|Ambiente|Operación normal|
|Artefacto|Integración con la pasarela de pagos certificada PCI-DSS (RIE-01)|
|Respuesta|El sistema envía la solicitud de cobro en el formato esperado por la pasarela y procesa la respuesta sin errores de interoperabilidad|
|Medida|100% de las transacciones se procesan sin errores de formato o protocolo con la pasarela|

**Escenario 2 — Convivencia de los 8 microservicios en VM3** · _Tipo: Fallo_ · _(Co-existence)_

|Parte|Contenido|
|---|---|
|Fuente|Un microservicio de baja prioridad (ej. Ranking Service)|
|Estímulo|Alcanza su límite de CPU o RAM|
|Ambiente|Operación normal, los 8 servicios comparten VM3|
|Artefacto|Matching Service (QoS Guaranteed) y los demás pods|
|Respuesta|Kubernetes limita únicamente al pod que excede su límite, sin desalojar ni degradar a los demás|
|Medida|El Matching mantiene menos de 3 segundos para el 95% de las solicitudes (AC2-E1) y 0 pods ajenos desalojados|

### 3.4 AC4 — Interaction Capability

**Escenario 1 — Creación de solicitud en pocos pasos** · _Tipo: Uso_ · _(Operability)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente|
|Estímulo|Inicia el flujo de creación de una solicitud de servicio|
|Ambiente|Operación normal, desde la app móvil|
|Artefacto|Interfaz de creación de solicitud|
|Respuesta|Completa el flujo indicando categoría, dirección y descripción|
|Medida|100% de las solicitudes se crean en un máximo de 3 pasos (RNF-11)|

**Escenario 2 — Uso en distintos dispositivos** · _Tipo: Uso_ · _(Operability)_

|Parte|Contenido|
|---|---|
|Fuente|Cualquier usuario autenticado (Cliente, Técnico, Admin)|
|Estímulo|Accede a la plataforma desde un dispositivo móvil o de escritorio|
|Ambiente|Operación normal|
|Artefacto|Interfaces Angular (panel admin) y Flutter (app móvil)|
|Respuesta|La interfaz se adapta al tamaño de pantalla sin pérdida de funcionalidad|
|Medida|100% de las pantallas son utilizables sin scroll horizontal ni elementos cortados en los tamaños soportados (RNF-12)|

**Escenario 3 — Reconocer si la aplicación sirve para lo que necesito** · _Tipo: Uso_ · _(Appropriateness Recognizability)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente nuevo (hogar)|
|Estímulo|Abre la aplicación por primera vez con un problema del hogar por resolver (ej. una fuga)|
|Ambiente|Primer uso, sin instrucciones previas|
|Artefacto|Pantalla inicial de la aplicación móvil|
|Respuesta|Muestra las categorías de servicio disponibles y la acción para solicitar un servicio, de modo que el usuario reconoce que la aplicación resuelve su necesidad|
|Medida|Al menos 4 de 5 usuarios de prueba identifican en menos de 30 segundos cómo solicitar un servicio, sin ayuda|

**Escenario 4 — Primer uso del técnico sin capacitación** · _Tipo: Uso_ · _(Learnability)_

|Parte|Contenido|
|---|---|
|Fuente|Técnico nuevo, ya aprobado|
|Estímulo|Recibe su primera solicitud asignada y debe aceptarla, ejecutarla y marcarla como completada (RF-10, RF-14, RF-15)|
|Ambiente|Primer uso, sin capacitación previa ni manual|
|Artefacto|Aplicación móvil del técnico|
|Respuesta|El técnico completa el ciclo sin ayuda externa|
|Medida|Al menos 4 de 5 técnicos de prueba completan el ciclo sin asistencia y en menos de 10 minutos, sin contar el tiempo del servicio en sí|

**Escenario 5 — Protección ante errores del usuario** · _Tipo: Fallo_ · _(User Error Protection)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente o Empresa|
|Estímulo|Envía una solicitud con datos incompletos o con una dirección que no se puede validar, o está por confirmar una acción irreversible como el pago|
|Ambiente|Operación normal|
|Artefacto|Formulario de solicitud (RF-07, RF-08) y flujo de pago (RF-22)|
|Respuesta|Valida antes de enviar, indica el campo y cómo corregirlo, y pide confirmación explícita antes de una acción irreversible|
|Medida|0 solicitudes creadas con campos obligatorios vacíos o con dirección no validada por el servicio de geolocalización (RIE-02); 100% de los errores de validación indican el campo y la corrección esperada|

**Escenario 6 — Motivación para calificar el servicio** · _Tipo: Uso_ · _(User Engagement)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente|
|Estímulo|Su servicio pasa a `completado`|
|Ambiente|Operación normal|
|Artefacto|Pantalla de calificación (RF-12) y seguimiento del estado (RF-11)|
|Respuesta|Presenta la calificación de 1 a 5 de forma inmediata y sencilla, sin pasos adicionales|
|Medida|Al menos el 60% de los servicios completados reciben calificación del cliente, medido con `ratings` frente a `service_requests` en estado `completado`|

**Escenario 7 — Uso con distintas capacidades** · _Tipo: Uso_ · _(User Assistance)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente con baja visión o dificultad motriz|
|Estímulo|Crea una solicitud de servicio con el texto del sistema ampliado al 200% o con lector de pantalla|
|Ambiente|Aplicación móvil (Flutter)|
|Artefacto|Pantallas del flujo de solicitud|
|Respuesta|El flujo se completa sin que el contenido se corte o se superponga, y los controles tienen etiquetas accesibles para el lector de pantalla|
|Medida|100% de las pantallas del flujo de solicitud pasan las pruebas de accesibilidad de Flutter (contraste de texto, etiquetas y tamaño mínimo de las áreas táctiles) y el flujo se completa con el texto al 200%|

**Escenario 8 — Uso en dispositivos de gama baja** · _Tipo: Uso_ · _(Inclusivity)_

|Parte|Contenido|
|---|---|
|Fuente|Técnico o cliente con un teléfono Android de gama baja|
|Estímulo|Usa el flujo principal: solicitar un servicio o aceptar una solicitud|
|Ambiente|Conexión móvil estable (FA1: sin modo offline)|
|Artefacto|Aplicación móvil (Flutter)|
|Respuesta|El flujo funciona con fluidez sin exigir un dispositivo reciente|
|Medida|El flujo completo de solicitud y de aceptación se completa sin cierres inesperados y cada pantalla carga en menos de 3 segundos en el dispositivo de referencia de gama baja que defina el equipo|

**Escenario 9 — Seguimiento del servicio que se explica solo** · _Tipo: Uso_ · _(Self-descriptiveness)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente|
|Estímulo|Consulta el estado de su solicitud en curso (RF-11)|
|Ambiente|Operación normal|
|Artefacto|Pantalla de seguimiento del servicio|
|Respuesta|Muestra por sí sola el estado actual, qué sigue y quién es el técnico asignado, sin abrir otras pantallas ni consultar ayuda externa|
|Medida|100% de los estados de la solicitud (mínimo 4, RF-11) muestran un texto que explica qué ocurre y cuál es el siguiente paso; al menos 4 de 5 usuarios de prueba describen correctamente el estado sin ayuda|

> **Nota:** los escenarios 3, 4 y 9 se miden con pruebas de usabilidad con 5 usuarios, y el 5 y el 7 con pruebas automáticas y manuales sobre la aplicación; hay que definir quién las ejecuta (QA o Product Owner) y cuándo. El escenario 8 requiere que Frontend defina el dispositivo de referencia de gama baja.

### 3.5 AC5 — Reliability

**Escenario 1 — Caída de un componente no crítico** · _Tipo: Fallo_ · _(Availability / Recoverability)_

|Parte|Contenido|
|---|---|
|Fuente|Falla de infraestructura|
|Estímulo|Un componente no crítico deja de responder|
|Ambiente|Operación normal|
|Artefacto|Sistema en general|
|Respuesta|Las funciones core (autenticación, solicitud de servicio) siguen operando desde los componentes restantes|
|Medida|Recuperación manual en un plazo de **12 a 24 horas**, dado que no hay operación 24/7|

**Escenario 2 — Mantenimiento planeado** · _Tipo: Uso_ · _(Availability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de DevOps|
|Estímulo|Se requiere aplicar una actualización o mantenimiento programado|
|Ambiente|Ventana anunciada al equipo|
|Artefacto|Sistema en general|
|Respuesta|El mantenimiento se realiza sin pérdida de datos, con downtime acotado y comunicado|
|Medida|Downtime planeado menor a **2 horas**|

**Escenario 3 — Caída de un microservicio individual** · _Tipo: Fallo_ · _(Fault Tolerance)_

|Parte|Contenido|
|---|---|
|Fuente|Falla de infraestructura o bug de despliegue|
|Estímulo|Un microservicio (ej. Ranking Service) deja de responder|
|Ambiente|Operación normal|
|Artefacto|Sistema de microservicios (VM3)|
|Respuesta|Los demás 7 servicios siguen operando; los eventos dirigidos al servicio caído se acumulan en Kafka sin perderse|
|Medida|0% de eventos perdidos durante la caída; el servicio recupera y procesa el backlog pendiente al reiniciar|

**Escenario 4 — Kafka no disponible temporalmente** · _Tipo: Fallo_ · _(Fault Tolerance)_

|Parte|Contenido|
|---|---|
|Fuente|Falla de infraestructura|
|Estímulo|El bus de eventos deja de responder|
|Ambiente|Operación normal|
|Artefacto|Todos los microservicios|
|Respuesta|Las operaciones síncronas críticas (vía API Gateway) siguen funcionando; los eventos se reintentan vía Transactional Outbox (ADR-007) hasta que Kafka se recupere|
|Medida|0 eventos perdidos; reintento automático sin intervención manual dentro de la ventana de recuperación del Escenario 1|

**Escenario 5 — Evento duplicado por reintento** · _Tipo: Fallo_ · _(Fault Tolerance — reubicado desde el antiguo AC de Trazabilidad, no es Accountability)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema (reintento tras falla temporal de red o de Kafka)|
|Estímulo|Un mismo evento se publica o se consume más de una vez|
|Ambiente|Tras recuperación de una falla temporal (ver Escenario 4)|
|Artefacto|Consumidores de eventos (idempotencia, ADR-007)|
|Respuesta|El efecto del evento se aplica una sola vez, pese a llegar duplicado|
|Medida|0 efectos duplicados (ej. doble notificación, doble cálculo de ranking), verificable por `eventId`|

**Escenario 6 — Operación sin fallos del flujo crítico** · _Tipo: Uso_ · _(Faultlessness)_

|Parte|Contenido|
|---|---|
|Fuente|Clientes y técnicos, y la prueba E2E automatizada (RF-27)|
|Estímulo|Uso normal del sistema durante la semana de evaluación|
|Ambiente|Producción, operación normal|
|Artefacto|Flujo crítico solicitud → pago → calificación|
|Respuesta|Las peticiones se completan sin errores del servidor|
|Medida|Menos del 1% de respuestas 5xx (medido con los logs del gateway en Loki) y 0 fallos en las ejecuciones programadas de la prueba E2E|

### 3.6 AC6 — Security

**Escenario 1 — Protección de datos de pago** · _Tipo: Uso_ · _(Confidentiality)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente|
|Estímulo|Ingresa datos de tarjeta en el checkout|
|Ambiente|Cualquier transacción de pago|
|Artefacto|Mecanismo de procesamiento de pagos|
|Respuesta|Tokeniza la información sin que el backend la almacene ni la registre en logs|
|Medida|0 ocurrencias de número de tarjeta o CVV en base de datos o logs|

**Escenario 2 — Aislamiento multi-tenant** · _Tipo: Uso_ · _(Confidentiality)_

|Parte|Contenido|
|---|---|
|Fuente|Usuario autenticado de un tenant|
|Estímulo|Realiza una consulta a la API|
|Ambiente|Operación normal|
|Artefacto|Backend — control de acceso a datos|
|Respuesta|Solo devuelve datos correspondientes a su propio tenant|
|Medida|0 casos de fuga de datos entre tenants en pruebas de aislamiento|

**Escenario 3 — Control de acceso por rol** · _Tipo: Uso_ · _(Confidentiality / Integrity — Confidentiality si la acción reservada es de lectura, Integrity si es de escritura como aprobar un pago o cambiar un estado)_

|Parte|Contenido|
|---|---|
|Fuente|Usuario autenticado con un rol específico|
|Estímulo|Intenta acceder a una acción reservada a otro rol|
|Ambiente|Operación normal|
|Artefacto|Backend — control de acceso basado en roles|
|Respuesta|Rechaza la acción con un error de autorización|
|Medida|100% de los endpoints sensibles validan el rol en el backend, no solo en el cliente|

**Escenario 4 — Token comprometido o expirado** · _Tipo: Fallo_ · _(Authenticity)_

|Parte|Contenido|
|---|---|
|Fuente|Atacante externo o cliente con sesión vencida|
|Estímulo|Intenta usar un token JWT robado, manipulado o expirado|
|Ambiente|Cualquier momento|
|Artefacto|Identity Service|
|Respuesta|Rechaza la petición antes de que llegue a cualquier microservicio downstream y fuerza reautenticación|
|Medida|100% de los tokens inválidos o expirados son rechazados en el Identity Service / API Gateway|

**Escenario 5 — Reconstrucción de una disputa** · _Tipo: Uso_ · _(Accountability — antiguo AC de Trazabilidad)_

|Parte|Contenido|
|---|---|
|Fuente|Administrador del tenant|
|Estímulo|Recibe un reclamo de un cliente sobre un servicio específico|
|Ambiente|Operación normal|
|Artefacto|Registro de eventos del ciclo de vida del servicio|
|Respuesta|Puede reconstruir la línea de tiempo completa del servicio (cotización, aceptación, entrega, evaluación), incluyendo la evidencia fotográfica adjuntada por el técnico al completar el servicio|
|Medida|100% de las transiciones de estado del servicio quedan registradas, sin pasos faltantes en la secuencia|

**Escenario 6 — Registro de cambios en pagos** · _Tipo: Uso_ · _(Accountability)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema|
|Estímulo|Se realiza un cambio de estado en un pago (creado, aprobado, rechazado, reembolsado)|
|Ambiente|Operación normal|
|Artefacto|Módulo de auditoría|
|Respuesta|Registra el origen del cambio, el momento, y el estado anterior y nuevo|
|Medida|100% de los cambios de estado en pagos quedan registrados con marca de tiempo|

**Escenario 7 — Acción crítica no repudiable** · _Tipo: Uso_ · _(Non-repudiation)_

|Parte|Contenido|
|---|---|
|Fuente|Técnico o cliente que niega haber realizado una acción (ej. "yo no completé ese servicio")|
|Estímulo|Disputa sobre una acción crítica: aceptar, completar o pagar un servicio|
|Ambiente|Resolución de una disputa|
|Artefacto|`audit_logs` y eventos del ciclo de vida del servicio|
|Respuesta|El sistema demuestra quién ejecutó la acción con identidad autenticada, momento y `correlationId`, en un registro que ni los servicios ni los usuarios pueden alterar|
|Medida|(1) 100% de las acciones críticas quedan asociadas a un usuario autenticado y a una marca de tiempo generada por el servidor; (2) en una prueba en staging, 100% de los intentos de modificar o borrar registros de `audit_logs` con la credencial de la aplicación son rechazados|

**Escenario 8 — Fuerza bruta contra el login** · _Tipo: Fallo_ · _(Resistance)_

|Parte|Contenido|
|---|---|
|Fuente|Atacante externo|
|Estímulo|Intentos automáticos de adivinar credenciales contra `POST auth/login`|
|Ambiente|Operación normal|
|Artefacto|Identity Service y API Gateway (bloqueo temporal, RF-03)|
|Respuesta|Bloquea temporalmente la cuenta atacada, limita la tasa de peticiones del origen y registra cada bloqueo; los usuarios legítimos siguen autenticándose|
|Medida|En un ataque simulado desde un origen (script o k6), 0 accesos exitosos con credenciales inválidas; 100% de los intentos posteriores al umbral configurado son rechazados y quedan en log (verificable en Loki)|

### 3.7 AC7 — Maintainability

**Escenario 1 — Pipeline de integración continua** · _Tipo: Cambio_ · _(Testability)_

|Parte|Contenido|
|---|---|
|Fuente|Desarrollador del equipo|
|Estímulo|Hace push de un cambio de código a un módulo|
|Ambiente|Desarrollo activo|
|Artefacto|Pipeline de integración continua (build y pruebas automáticas)|
|Respuesta|El pipeline corre y reporta si el cambio rompió algo|
|Medida|Completa build y pruebas en menos de **10 minutos**|

**Escenario 2 — Acoplamiento entre módulos** · _Tipo: Cambio_ · _(Modularity)_

|Parte|Contenido|
|---|---|
|Fuente|Desarrollador del equipo|
|Estímulo|Implementa una funcionalidad nueva o corrige un error|
|Ambiente|Desarrollo activo|
|Artefacto|Estructura de microservicios|
|Respuesta|El cambio requiere modificaciones solo en el o los servicios directamente relacionados|
|Medida|La mayoría de los cambios (80% o más) afectan un solo servicio; cambios que afectan tres o más son la excepción|

**Escenario 3 — Incorporación de un nuevo microservicio** · _Tipo: Cambio_ · _(Modifiability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de desarrollo|
|Estímulo|Se requiere agregar una nueva capacidad de negocio no cubierta por los 8 servicios actuales|
|Ambiente|Evolución del sistema más allá del alcance académico actual|
|Artefacto|Arquitectura de microservicios|
|Respuesta|El nuevo servicio se agrega y se conecta a Kafka con sus propios eventos, sin modificar el código de los servicios existentes|
|Medida|0 cambios de código requeridos en los microservicios existentes para incorporar uno nuevo|

**Escenario 4 — Diagnóstico de un fallo en producción** · _Tipo: Fallo_ · _(Analysability)_

|Parte|Contenido|
|---|---|
|Fuente|Ingeniero de DevOps o Backend|
|Estímulo|Recibe el reporte de una solicitud en estado incorrecto (ej. pago aprobado sin cambio de estado)|
|Ambiente|Producción|
|Artefacto|Logs centralizados (Loki/Grafana) y eventos con `correlationId`|
|Respuesta|Rastrea el flujo entre servicios filtrando por `correlationId` y localiza el servicio y el evento donde falló|
|Medida|Causa identificada en 1 hora o menos desde que el ingeniero inicia la revisión, usando solo Grafana/Loki y sin acceso por SSH a los contenedores|

**Escenario 5 — Reutilización de la lógica transversal** · _Tipo: Cambio_ · _(Reusability)_

|Parte|Contenido|
|---|---|
|Fuente|Desarrollador Backend|
|Estímulo|Crea un microservicio nuevo que necesita contexto de tenant, idempotencia, Outbox y logging|
|Ambiente|Desarrollo|
|Artefacto|Módulos transversales compartidos|
|Respuesta|Los reutiliza en vez de reimplementarlos|
|Medida|0 copias de esa lógica entre servicios: una sola implementación por stack (ASP.NET Core y Spring Boot), verificable en revisión de PR|

### 3.8 AC8 — Flexibility

**Escenario 1 — Alta de nuevo tenant** · _Tipo: Cambio_ · _(Adaptability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo administrador de la plataforma|
|Estímulo|Se registra una nueva empresa (tenant) en el sistema|
|Ambiente|Operación normal, sin interrumpir a los tenants existentes|
|Artefacto|Módulo de gestión de tenants y esquema de base de datos|
|Respuesta|El nuevo tenant queda operativo con sus datos aislados, sin requerir cambios estructurales en el esquema|
|Medida|Tenant funcional en menos de **1 hora**, sin downtime para tenants existentes|

**Escenario 2 — Escalado independiente de un microservicio** · _Tipo: Cambio_ · _(Scalability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de DevOps|
|Estímulo|El Matching Service requiere más capacidad por aumento sostenido de demanda|
|Ambiente|Crecimiento sostenido de solicitudes|
|Artefacto|Matching Service (contenedor en VM3)|
|Respuesta|Se despliegan instancias adicionales del contenedor de ese servicio, sin tocar los demás 7|
|Medida|El escalado no requiere cambios de código ni downtime en los demás microservicios|

**Escenario 3 — Incorporación de un nuevo canal o tipo de cliente** · _Tipo: Cambio_ · _(Adaptability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de desarrollo|
|Estímulo|Se evalúa agregar un nuevo cliente (ej. una versión web para Clientes, no solo para Admin)|
|Ambiente|Evolución del sistema más allá del alcance académico actual|
|Artefacto|API Gateway y contratos REST/eventos existentes|
|Respuesta|El nuevo cliente consume los mismos contratos ya definidos, sin requerir cambios en los microservicios de dominio|
|Medida|0 cambios de código en los microservicios de dominio para incorporar un nuevo tipo de cliente|

**Escenario 4 — Cambio de proveedor de geolocalización** · _Tipo: Cambio_ · _(Replaceability — reubicado desde el antiguo AC de Compatibilidad)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de desarrollo|
|Estímulo|Se requiere reemplazar el servicio de mapas/geocodificación actual (RIE-02) por otro proveedor|
|Ambiente|Evolución del sistema|
|Artefacto|Módulo de integración de geolocalización (Matching Service)|
|Respuesta|El cambio se limita a la capa de integración externa, sin afectar la lógica de negocio del matching|
|Medida|El reemplazo del proveedor no requiere cambios en los demás microservicios|

**Escenario 5 — Reconstrucción del entorno desde cero** · _Tipo: Cambio_ · _(Installability)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de DevOps|
|Estímulo|Debe reinstalar una VM desde cero tras una falla|
|Ambiente|VM recién provisionada (Ubuntu 22.04)|
|Artefacto|Playbooks de Ansible y manifiestos de k3s|
|Respuesta|Deja la VM operativa ejecutando solo los playbooks|
|Medida|VM operativa en 4 horas o menos, dentro de la ventana de recuperación de 12–24 h (sección 1.2, nota sobre K7), sin pasos manuales fuera de los documentados; re-ejecutar el playbook da `changed=0`|

> **Nota:** dos escenarios se retiraron por redundantes con los de arriba, no por pérdida de cobertura: "nuevo tenant con reglas propias" (medía lo mismo que el Escenario 1 — alta de tenant, *Adaptability*) y ya se cuenta el crecimiento del catálogo bajo AC2/Performance Efficiency.

### 3.9 AC9 — Safety

Safety, en ISO/IEC 25010:2023, es el grado en que el producto evita causar daño a personas, al negocio, al software, a la propiedad o al entorno en su contexto de uso. A diferencia de *Security*, que protege al sistema de un atacante, Safety protege a las personas y sus bienes de los propios fallos del sistema.

Para QUICKPATCH, el equipo definió el alcance así: la plataforma no debe exponer a clientes ni técnicos a riesgo físico o patrimonial por sus propios fallos — no enviar a un domicilio a un técnico que no esté habilitado, restringir lo que puede hacer cada estado del servicio y, ante la duda, fallar hacia el lado seguro. La protección de datos y el triángulo confidencialidad-integridad-disponibilidad no se cubren aquí: son *Security* (AC6) y *Reliability* (AC5).

**Escenario 1 — Transiciones seguras del ciclo de servicio** · _Tipo: Uso_ · _(Operational Constraint)_

|Parte|Contenido|
|---|---|
|Fuente|Técnico o usuario que intenta una operación fuera de secuencia (ej. completar un servicio que no inició)|
|Estímulo|Solicita cambiar una solicitud a un estado no permitido (ej. `completado` sin pasar por `en_progreso`, o `pagado` sin pago aprobado)|
|Ambiente|Operación normal|
|Artefacto|Service Request Service (máquina de estados)|
|Respuesta|Rechaza la transición, conserva el estado anterior y deja el intento registrado en el log|
|Medida|0 transiciones inválidas en `service_requests`: `completado` solo desde `en_progreso` y `pagado` solo tras `payment.approved`, verificable en la tabla y en el log|

**Escenario 2 — Técnico con calificaciones bajas** · _Tipo: Uso_ · _(Risk Identification)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema (cálculo periódico sobre las calificaciones)|
|Estímulo|Un técnico acumula un promedio de calificaciones menor a 3.0 tras al menos 5 servicios calificados|
|Ambiente|Operación normal|
|Artefacto|Panel administrativo (RF-18) y `technician_profiles.average_rating`|
|Respuesta|El técnico queda señalado para que el Admin evalúe suspenderlo (RF-20)|
|Medida|100% de los técnicos que cumplen la condición aparecen señalados en el panel del Admin, sin suspensión automática|

**Escenario 3 — Servicio sin cierre** · _Tipo: Uso_ · _(Risk Identification)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema|
|Estímulo|Una solicitud permanece en `asignado` o `en_progreso` más de 4 horas sin cambiar de estado|
|Ambiente|Operación normal|
|Artefacto|Service Request Service y panel del Admin (RF-18)|
|Respuesta|La solicitud se marca para revisión del Admin (el técnico no llegó o no cerró el servicio)|
|Medida|100% de las solicitudes que superan ese tiempo quedan marcadas en el panel del Admin|

**Escenario 4 — Sin candidatos elegibles** · _Tipo: Fallo_ · _(Fail Safe)_

|Parte|Contenido|
|---|---|
|Fuente|Cliente o Empresa|
|Estímulo|Crea una solicitud y no hay ningún técnico elegible disponible (aprobado, con la especialidad, en la zona y disponible)|
|Ambiente|Operación normal|
|Artefacto|Matching Service|
|Respuesta|No relaja el criterio de elegibilidad para conseguir un técnico: la solicitud queda `en_espera` y el cliente recibe la notificación de que no hay técnicos disponibles (`matching.no-technician-available`)|
|Medida|0 asignaciones a técnicos que no cumplen el filtro, aunque no haya candidatos, verificable en `matching_attempts`|

**Escenario 5 — Pasarela de pagos sin respuesta** · _Tipo: Fallo_ · _(Fail Safe)_

|Parte|Contenido|
|---|---|
|Fuente|Falla del proveedor de pagos|
|Estímulo|La pasarela no responde o devuelve un error al procesar un pago|
|Ambiente|Operación normal|
|Artefacto|Payments Service|
|Respuesta|No confirma el pago ni cambia el estado del servicio; el cliente puede reintentar sin riesgo de cobro doble|
|Medida|0 servicios en `pagado` sin un evento `payment.approved` y 0 cobros duplicados por reintento|

**Escenario 6 — Alerta de operación en riesgo** · _Tipo: Fallo_ · _(Hazard Warning)_

|Parte|Contenido|
|---|---|
|Fuente|Sistema de monitoreo (Prometheus y Grafana)|
|Estímulo|El uso de RAM de VM3, o el de disco de VM4 o VM7, supera el 85%|
|Ambiente|Operación normal|
|Artefacto|Alertas de Grafana sobre las métricas de `node_exporter`|
|Respuesta|Emite una alerta al equipo antes de que el riesgo afecte la operación (desalojo de pods, caída de la base de datos o del almacenamiento)|
|Medida|Alerta emitida en menos de 15 minutos desde que se cumple la condición|

**Escenario 7 — Integración de un componente nuevo verificada de punta a punta** · _Tipo: Cambio_ · _(Safe Integration)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de desarrollo|
|Estímulo|Incorpora un servicio nuevo o cambia un proveedor externo que participa en el flujo de matching|
|Ambiente|Pipeline de CI, antes del despliegue|
|Artefacto|Prueba E2E del flujo crítico (RF-27)|
|Respuesta|Ejecuta el flujo completo solicitud → asignación → pago → calificación y verifica que un técnico no aprobado no recibe ofertas; si falla, bloquea el despliegue (RNF-08)|
|Medida|100% de las ejecuciones de la prueba E2E pasan antes del despliegue y 0 despliegues con la prueba fallida|

**Escenario 8 — Cambio de proveedor de geolocalización sin alterar la cobertura** · _Tipo: Cambio_ · _(Safe Integration)_

|Parte|Contenido|
|---|---|
|Fuente|Equipo de desarrollo|
|Estímulo|Reemplaza el proveedor de geolocalización (RIE-02)|
|Ambiente|Pipeline de CI|
|Artefacto|Filtro de cobertura del Matching Service|
|Respuesta|El filtro de cobertura produce los mismos resultados que con el proveedor anterior sobre un conjunto fijo de casos de regresión|
|Medida|100% de los casos de regresión de cobertura dan el mismo resultado antes y después del cambio|

> **Pendiente:** hay escenarios de Safety que dependen de dos decisiones de diseño que el equipo debe cerrar con Backend y el Product Owner: (1) el mecanismo por el que el Matching conoce el estado de verificación del técnico (`technician_profiles.verification_status`, propiedad de Identity Service; ningún evento del DD lo comunica hoy) y (2) un atributo de riesgo físico en `service_categories` (el Product Owner confirmó que algunas categorías se tratan como de riesgo físico; la lista está por definir). Con ellas se redactarían escenarios de *Operational Constraint* y *Fail Safe* sobre la habilitación del técnico, de *Risk Identification* sobre categorías de riesgo, de *Hazard Warning* sobre asignaciones a técnicos no habilitados y de *Safe Integration* con pruebas de contrato de elegibilidad.


### 3.10 Priorización de escenarios

Siguiendo el método ATAM, cada escenario se prioriza en dos ejes votados por separado, cada uno desde una perspectiva distinta:

- **Importancia de negocio** (vota el cliente / stakeholders de negocio): qué tanto le duele al éxito del proyecto si este escenario no se cumple.
- **Dificultad arquitectónica** (vota el equipo de arquitectura): qué tanto esfuerzo, riesgo técnico o incertidumbre implica lograrlo.

Un escenario con importancia de negocio **Alta** se considera **prioritario** — es candidato a sustentar una decisión de arquitectura (ADR) con su trade-off explícito; el resto queda documentado pero no mueve decisiones de fondo. Cuando la dificultad arquitectónica también es Alta (Alta/Alta), el escenario es el más crítico de los prioritarios, porque combina alto impacto de negocio con alto riesgo técnico. Esto no garantiza que todo escenario prioritario termine anclado a un ADR explícito, ni que todo ADR se apoye solo en escenarios Alta/Alta — ver la nota al final de esta sección y la matriz de la sección 6, que documentan el sustento real en vez de forzar una correspondencia perfecta que no existe.

> Los escenarios reubicados conservan la votación que ya tenían (el escenario en sí no cambió, solo su clasificación de característica ISO). Los escenarios nuevos o retirados están marcados abajo; los nuevos quedan pendientes de votación con cliente/stakeholders — no se inventa esa votación aquí.

> Los 10 escenarios agregados en la versión 2.6 (AC1-E2, AC1-E3, AC2-E5, AC3-E2, AC5-E6, AC6-E7, AC6-E8, AC7-E4, AC7-E5, AC8-E5) son una primera propuesta: sus medidas deben validarse con el equipo y votarse en la sesión ATAM. AC6-E7 y AC7-E5 dependen además de confirmación de Backend.

> Los 8 escenarios de Safety agregados en la versión 2.7 (AC9-E1 a AC9-E8) también son una primera propuesta: sus medidas (calificación de 3.0, 5 servicios, 4 horas, 85%, 15 minutos) deben validarse con el equipo y votarse en la sesión ATAM. AC9-E6 es una alerta operativa, más cercana a *Reliability* que a *Safety*, y puede reemplazarse cuando se cierre el escenario de alerta sobre la habilitación de técnicos.

> Los 7 escenarios de Interaction Capability agregados en la versión 2.8 (AC4-E3 a AC4-E9) también son una primera propuesta: sus umbrales (4 de 5 usuarios, 30 segundos, 10 minutos, 60% de calificaciones, 3 segundos) deben validarse con el equipo y votarse en la sesión ATAM.

|Escenario|Atributo|Importancia de negocio|Dificultad arquitectónica|Prioridad|
|---|---|---|---|---|
|AC1-E1 Filtro de especialidad en el matching|Functional Suitability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC1-E2 Cobertura de requisitos del MVP|Functional Suitability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC1-E3 Información suficiente para ejecutar el servicio|Functional Suitability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC2-E1 Búsqueda de técnicos cercanos|Performance Efficiency|Alta|Alta|**Alta**|
|AC2-E2 Consulta de historial|Performance Efficiency|Media|Baja|Baja|
|AC2-E3 Procesamiento en segundo plano|Performance Efficiency|Alta|Media|**Alta**|
|AC2-E4 Pico de carga inesperado|Performance Efficiency|Media|Alta|Media|
|AC2-E5 Uso de recursos bajo carga normal|Performance Efficiency|*Pendiente de votación*|*Pendiente de votación*|—|
|AC3-E1 Integración con pasarela de pagos|Compatibility|Media|Media|Media|
|AC3-E2 Convivencia de los 8 microservicios en VM3|Compatibility|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E1 Creación de solicitud en pocos pasos|Interaction Capability|Media|Baja|Baja|
|AC4-E2 Uso en distintos dispositivos|Interaction Capability|Media|Baja|Baja|
|AC4-E3 Reconocer si la aplicación sirve para lo que necesito|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E4 Primer uso del técnico sin capacitación|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E5 Protección ante errores del usuario|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E6 Motivación para calificar el servicio|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E7 Uso con distintas capacidades|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E8 Uso en dispositivos de gama baja|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC4-E9 Seguimiento del servicio que se explica solo|Interaction Capability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC5-E1 Caída de componente no crítico|Reliability|Alta|Media|**Alta**|
|AC5-E2 Mantenimiento planeado|Reliability|Media|Baja|Baja|
|AC5-E3 Caída de un microservicio|Reliability|Alta|Alta|**Alta**|
|AC5-E4 Kafka no disponible|Reliability|Alta|Alta|**Alta**|
|AC5-E5 Evento duplicado por reintento|Reliability|Alta|Alta|**Alta**|
|AC5-E6 Operación sin fallos del flujo crítico|Reliability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC6-E1 Protección de datos de pago|Security|Alta|Media|**Alta**|
|AC6-E2 Aislamiento multi-tenant|Security|Alta|Alta|**Alta**|
|AC6-E3 Control de acceso por rol|Security|Media|Baja|Baja|
|AC6-E4 Token comprometido o expirado|Security|Alta|Media|**Alta**|
|AC6-E5 Reconstrucción de una disputa|Security|Alta|Media|**Alta**|
|AC6-E6 Registro de cambios en pagos|Security|Alta|Baja|Media|
|AC6-E7 Acción crítica no repudiable|Security|*Pendiente de votación*|*Pendiente de votación*|—|
|AC6-E8 Fuerza bruta contra el login|Security|*Pendiente de votación*|*Pendiente de votación*|—|
|AC7-E1 Pipeline de integración continua|Maintainability|Media|Baja|Baja|
|AC7-E2 Acoplamiento entre módulos|Maintainability|Media|Media|Media|
|AC7-E3 Incorporación de nuevo microservicio|Maintainability|Baja|Media|Baja|
|AC7-E4 Diagnóstico de un fallo en producción|Maintainability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC7-E5 Reutilización de la lógica transversal|Maintainability|*Pendiente de votación*|*Pendiente de votación*|—|
|AC8-E1 Alta de nuevo tenant|Flexibility|Media|Media|Media|
|AC8-E2 Escalado independiente de servicio|Flexibility|Media|Media|Media|
|AC8-E3 Nuevo canal o tipo de cliente|Flexibility|Alta|Media|**Alta**|
|AC8-E4 Cambio de proveedor de geolocalización|Flexibility|Baja|Baja|Baja|
|AC8-E5 Reconstrucción del entorno desde cero|Flexibility|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E1 Transiciones seguras del ciclo de servicio|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E2 Técnico con calificaciones bajas|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E3 Servicio sin cierre|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E4 Sin candidatos elegibles|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E5 Pasarela de pagos sin respuesta|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E6 Alerta de operación en riesgo|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E7 Integración de un componente nuevo verificada de punta a punta|Safety|*Pendiente de votación*|*Pendiente de votación*|—|
|AC9-E8 Cambio de proveedor de geolocalización sin alterar la cobertura|Safety|*Pendiente de votación*|*Pendiente de votación*|—|

**Escenarios "oro" (prioridad Alta) tras la reorganización:** AC2-E1, AC2-E3, AC5-E1, AC5-E3, AC5-E4, AC5-E5, AC6-E1, AC6-E2, AC6-E4, AC6-E5, AC8-E3 — 11 escenarios, el mismo número que antes de la reestructuración (la reorganización reclasifica, no cambia cuántos escenarios son prioritarios). De estos, 5 son Alta/Alta (AC2-E1, AC5-E3, AC5-E4, AC5-E5, AC6-E2) y 6 son Alta importancia/Media dificultad (AC2-E3, AC5-E1, AC6-E1, AC6-E4, AC6-E5, AC8-E3) — ambos grupos califican como prioritarios bajo el criterio de esta sección (importancia de negocio Alta).

De estos 11, **4 no sustentan ningún ADR todavía** (AC5-E1, AC6-E4, AC6-E5, AC8-E3, ver matriz de la sección 6) — no implica un error, solo que ninguna decisión de arquitectura se ha tomado en torno a ellos. A la inversa, **ADR-003 se apoya también en AC8-E2**, un escenario Media/Media, además de en AC5-E3 y AC5-E4 (Alta/Alta): el trade-off de escalado independiente por servicio que documenta ADR-003 es real aunque ese escenario en particular todavía no haya sido votado como prioritario.

**Total de escenarios en esta versión: 51**, cubriendo las 40 subcaracterísticas de ISO/IEC 25010:2023. En Safety hay más de un escenario para *Risk Identification*, *Fail Safe* y *Safe Integration*, y quedan pendientes los escenarios de Safety que dependen de decisiones de diseño (sección 3.9). La cobertura es completa en cantidad; la validación de las medidas y la votación ATAM siguen pendientes.

---

## 4. Arquitectura de Alto Nivel (HLD)

Esta sección presenta la vista general de componentes del sistema y cómo se conectan entre sí, sin entrar aún al detalle interno de cada uno. La arquitectura se plantea como un conjunto de **microservicios independientes**, organizados según los dominios de negocio, que se comunican principalmente mediante **eventos publicados en Apache Kafka** (Event-Driven Architecture) en lugar de llamadas síncronas directas entre servicios.

### 4.1 Diagrama general

```mermaid
flowchart TB
    subgraph Usuarios["Usuarios"]
        direction LR
        U1[Cliente]
        U2[Aliado/Tecnico]
        U3[Empleado]
        U4[Proveedor]
        U5[Empresa]
        U6[Admin]
    end

    subgraph Frontend["Clientes"]
        direction LR
        WEB["Angular (Web)<br/>Panel Admin"]
        MOBILE["Flutter<br/>(iOS/Android)"]
    end

    Usuarios --> Frontend

    WEB -->|"HTTPS - REST - WebSocket"| GW
    MOBILE -->|"HTTPS - REST - WebSocket"| GW

    GW["API Gateway"]

    subgraph Services["Microservicios"]
        direction LR
        S1["Identity Service<br/>Auth . Tenants . Users"]
        S2["Actors Service<br/>Suppliers . Allies . Clients"]
        S3["Catalog Service"]
        S4["Matching Service<br/>+ CoverageZone"]
        S5["ServiceRequest Service<br/>orquesta ciclo de vida"]
        S6["Ranking Service<br/>servicio + materiales"]
        S7["Payments Service<br/>Payments . Billing . Payroll-lite"]
        S8["Communication Service<br/>Notifications<br/>(Chat . Complaints: futuro)"]
    end

    GW --> Services

    KAFKA{{"Apache Kafka<br/>Bus de eventos"}}

    Services <--> KAFKA

    Services --> DB[("PostgreSQL + PostGIS<br/>por servicio o esquema")]
    Services --> CACHE[("Redis<br/>cache / colas cortas")]
    Services --> STORAGE[("MinIO<br/>archivos y evidencias")]
```

### 4.2 Componentes

#### 4.2.1 Clientes (frontend)

- **Angular (Web)**: panel administrativo del tenant — gestión de empleados, aliados, proveedores y reportes. Exclusivamente backoffice, sin sitio público indexable (ver FA2).
- **Flutter (iOS/Android)**: aplicación para clientes y para técnicos/aliados en campo.

Ambos clientes se comunican con el sistema a través de un **API Gateway** único, que enruta cada solicitud al microservicio correspondiente y resuelve autenticación y rate limiting de forma centralizada.

#### 4.2.2 Microservicios

Cada servicio es responsable de un dominio de negocio, con su propio ciclo de despliegue:

|Servicio|Módulos que agrupa|
|---|---|
|Identity Service|Auth, Tenants, Users|
|Actors Service|Suppliers, Allies, Clients|
|Catalog Service|Catalog (especialidades, servicios publicados)|
|Matching Service|Matching, CoverageZone|
|ServiceRequest Service|ServiceRequest (orquesta cotización → prestación → evaluación)|
|Ranking Service|Ranking (servicio prestado; el de materiales queda fuera del MVP, ver D3)|
|Payments Service|Payments, Billing, Payroll-lite|
|Communication Service|Notifications (Chat y Complaints planificados para versiones futuras — ver Roadmap 1.x, fuera del alcance del MVP)|

#### 4.2.3 Persistencia

- **PostgreSQL + PostGIS**: cada servicio gestiona su propia base o esquema, evitando acoplamiento a nivel de datos entre servicios. PostGIS resuelve las consultas geoespaciales del Matching Service.
- **Redis**: cache y colas de corta duración, compartido como infraestructura de soporte.

#### 4.2.4 Comunicación entre servicios (Event-Driven)

**Apache Kafka** deja de ser solo un mecanismo de procesamiento en segundo plano y pasa a ser el **canal principal de comunicación entre microservicios**: cada servicio publica eventos de su dominio (por ejemplo, `SERVICE_REQUESTED`, `QUOTE_ACCEPTED`, `PAYMENT_APPROVED`) y los demás servicios interesados los consumen de forma asíncrona, sin acoplarse directamente entre sí vía llamadas REST síncronas.

Ejemplos de flujo de eventos:

- `ServiceRequest Service` publica `SERVICE_REQUESTED` → lo consume `Matching Service` (busca candidatos) y `Communication Service` (notifica)
- `ServiceRequest Service` publica `SERVICE_EVALUATED` → lo consume `Ranking Service` (recalcula reputación)
- `Payments Service` publica `PAYMENT_APPROVED` → lo consume `ServiceRequest Service` (libera el servicio) y `Communication Service` (notifica al cliente)

### 4.3 Principio rector: comunicación por eventos, no por llamadas directas

Los servicios no se llaman entre sí de forma síncrona salvo cuando el usuario necesita una respuesta inmediata (a través del API Gateway). Toda coordinación entre dominios de negocio ocurre mediante eventos publicados y consumidos vía Kafka. Esto reduce el acoplamiento entre servicios, pero introduce complejidad adicional: consistencia eventual (los datos entre servicios no se actualizan de forma instantánea) y necesidad de manejar fallos de entrega de eventos (ver ADR-007, Transactional Outbox + idempotencia, ahora crítico para _todo_ el sistema y no solo para tareas en segundo plano).

---

## 5. Arquitectura de Infraestructura

Esta sección describe cómo se distribuye el sistema sobre las 7 VMs propias (K5, K10), y cómo se automatiza su configuración y despliegue dado que una sola persona (DevOps) administra las 7 máquinas (K11).

### 5.1 Distribución de VMs

|VM|IP|Rol|Qué corre|
|---|---|---|---|
|VM1|10.43.100.168|Gateway / Entry point|Nginx + API Gateway — enruta tráfico a Angular y a los 8 microservicios en VM3|
|VM2|10.43.98.15|Frontend Web|Angular (panel administrativo del tenant — sin sitio público, ver FA2)|
|VM3|10.43.98.205|Backend — microservicios|8 microservicios (Identity, Actors, Catalog, Matching, ServiceRequest, Ranking, Payments, Communication) como Deployments de Kubernetes (k3s)|
|VM4|10.43.98.209|Base de datos|PostgreSQL + PostGIS (fuente de verdad, incluye datos geoespaciales)|
|VM5|10.43.98.29|Cache / colas cortas|Redis (cache y coordinación temporal para procesos programados)|
|VM6|10.43.99.12|Mensajería asíncrona|Apache Kafka + Kafka UI (matching, ranking, notificaciones, pagos)|
|VM7|10.43.99.8|Storage + Observabilidad|MinIO (evidencias fotográficas obligatorias al completar un servicio, ver RF-15) + Prometheus + Loki + Grafana (métricas y logs, ver Documento de Infraestructura, sección 7)|

> **Nota sobre storage de evidencias (MinIO vs. Cloudflare R2):** confirmado con el equipo que la evidencia fotográfica sí está en el alcance del MVP. El storage se aprovisiona como **MinIO self-hosted en VM7**, consistente con K5. El diagrama de la presentación de Sprint 1 mostraba "MinIO / Cloudflare R2" como si fueran intercambiables; se descarta Cloudflare R2 para esta versión por K5: guardaría datos de usuarios en producción (las evidencias) fuera de las 7 VMs, aunque su capa gratuita no tenga costo. GitHub Container Registry sí se usa porque solo guarda imágenes de contenedor, no datos de usuarios. Si en el futuro se reconsidera, debe evaluarse explícitamente contra K5 antes de adoptarlo.

Dado K10 (hardware fijo de 7 VMs), los 8 microservicios no reciben una VM cada uno. Los 8 corren dentro de VM3, orquestados con Kubernetes (k3s, clúster de un solo nodo), lo que permite escalado independiente por servicio, auto-healing y rolling updates sin downtime — ver ADR-011 (sección 5.5) y ADR-003 (sección 6).

### 5.2 Principio de distribución

- **Separación por criticidad**: lo síncrono (VM1-VM4) queda aislado de lo asíncrono (VM6), así si Kafka se satura no tumba la API.
- **Base de datos sola en su VM** (VM4): es el recurso más sensible — nunca comparte máquina con procesos que puedan consumir su CPU/RAM.
- **Redis separado de Kafka** (VM5 vs. VM6): aunque ambos son infraestructura de soporte, tienen patrones de carga distintos (Redis = baja latencia constante, Kafka = throughput por ráfagas).

> **Limitación reconocida — punto único de falla en VM3:** k3s aísla los 8 microservicios entre sí a nivel de pod, con auto-healing (si un pod falla, Kubernetes lo reinicia automáticamente sin afectar a los demás, ver escenario AC5-E3), pero **siguen compartiendo la misma máquina física** al ser un clúster de un solo nodo. Si VM3 completa falla (hardware, memoria agotada, etc.), los 8 servicios caen simultáneamente porque no hay un segundo nodo al cual Kubernetes pueda reprogramar los pods. Esto limita el beneficio de "resiliencia ante fallos aislados" atribuido a los microservicios en el ADR-003 al nivel de proceso/pod, no al nivel de máquina — un clúster multi-nodo eliminaría esta limitación, pero requeriría VMs adicionales que violan K10 (hardware fijo de 7 VMs). El presupuesto de CPU/RAM por microservicio que mitiga el riesgo de que la memoria agotada dispare esta falla se define en el Documento de Infraestructura, sección 5.6.

### 5.3 Orden de arranque

Existen dependencias de arranque entre componentes: PostgreSQL y Kafka deben estar disponibles antes que los microservicios. Este orden se garantiza con healthchecks en Docker Compose, probes de Kubernetes (`readinessProbe`/`livenessProbe`) en k3s, y/o con un script de orquestación:

1. VM4 (PostgreSQL/PostGIS) y VM5 (Redis)
2. VM6 (Kafka) — los microservicios dependen del bus de eventos para operar correctamente
3. VM3 (los 8 microservicios, vía Kubernetes/k3s)
4. VM2 (Angular) y VM1 (Nginx Gateway / API Gateway)
5. VM7 (MinIO + Observabilidad) — independiente, puede iniciar en paralelo

### 5.4 Automatización, despliegue y CI/CD

El detalle operativo de esta sección (playbooks de Ansible, manifiestos de Kubernetes, estructura del pipeline de CI/CD, runner self-hosted, gates por rama) se documenta y se mantiene actualizado en el **Documento de Infraestructura V1**, no aquí — evita que el mismo contenido opere desde dos lugares y se desincronice. Los principios que sí son decisiones de arquitectura, y por tanto pertenecen a este documento, son:

- Una sola persona (DevOps) administra las 7 VMs → la automatización con Ansible no es opcional (Documento de Infraestructura, sección 5.1).
- Solo VM3 usa Kubernetes (k3s); el resto de VMs usa Docker Compose, porque son las únicas que alojan múltiples servicios independientes que se benefician de esa orquestación (ver ADR-011, sección 5.5).
- El pipeline de CI/CD está separado por microservicio para que un cambio en uno no dispare el de los 8 (Documento de Infraestructura, sección 6).

### 5.5 ADR de infraestructura

|ID|Decisión|Atributo priorizado|Atributo sacrificado|Justificación|
|---|---|---|---|---|
|ADR-011|Kubernetes (k3s, clúster de un solo nodo en VM3) para orquestar los 8 microservicios; Docker Compose para el resto de VMs|AC5 Reliability (rolling updates sin downtime, auto-healing de contenedores)|Costo/simplicidad operativa (curva de aprendizaje y administración de un clúster, aunque sea de un solo nodo)|Kubernetes real (vía k3s) sin salirse del presupuesto de 7 VMs (K5); el equipo asume conscientemente la mayor complejidad operativa pese a K7 (sin operación 24/7), confiando en la capacidad propia para administrarlo|

> **Nota:** se descartó un clúster de Kubernetes multi-nodo (vía `kubeadm` completo) por requerir VMs adicionales dedicadas al control plane, lo cual viola K10. k3s resuelve esto al ser una distribución de Kubernetes completa pero liviana, capaz de correr en un solo nodo (VM3) sin sacrificar la API estándar de Kubernetes ni los manifiestos de Deployment/Service. El resto de las VMs (base de datos, cache, mensajería, storage) se mantiene en Docker Compose simple, ya que no alojan múltiples servicios independientes que se beneficien de orquestación.

---

## 6. Trade-offs y ADRs

Cada ADR (Architecture Decision Record) documenta una decisión de arquitectura ya tomada. Un trade-off arquitectónico siempre ocurre **entre atributos de calidad**: se prioriza uno a costa de otro. La columna "Justificación" conecta cada decisión con el driver/killer y el escenario prioritario (sección 3.10) que la motivaron.

|ID|Decisión|Atributo priorizado|Atributo sacrificado|Justificación|
|---|---|---|---|---|
|ADR-002|Flutter como cliente único móvil|AC7 Maintainability|AC2 Performance Efficiency (nativo por plataforma)|K3: una sola base de código a cambio de perder rendimiento/APIs nativas óptimas por plataforma|
|ADR-003|Microservicios + Event-Driven Architecture|AC8 Flexibility, AC5 Reliability|AC7 Maintainability|Fallos aislados y escalado independiente por servicio (AC5-E3, AC5-E4, AC8-E2), a costa de mayor complejidad de desarrollo y riesgo frente a K3, K7|
|ADR-004|PostgreSQL + PostGIS|AC2 Performance Efficiency|AC7 Maintainability (flexibilidad de esquema de un motor NoSQL)|D1: consultas geoespaciales nativas para el matching (AC2-E1)|
|ADR-005|Shared-schema con `tenant_id` + RLS|AC7 Maintainability (costo/velocidad de implementación)|AC6 Security (aislamiento físico total)|D5, K3: aislamiento lógico vía RLS en vez de un esquema o base separada por tenant, por menor costo de implementación dentro del tiempo del curso (AC6-E2)|
|ADR-006|Kafka como bus de eventos central|AC5 Reliability, AC8 Flexibility|AC7 Maintainability, AC2 Performance Efficiency|D1, D2, D6: bajo acoplamiento entre servicios a costa de consistencia eventual (AC2-E3, AC5-E4)|
|ADR-007|Transactional Outbox + idempotencia|AC5 Reliability|AC7 Maintainability|D6: confiabilidad ante fallos de Kafka y eventos duplicados (AC5-E4, AC5-E5) — ambos escenarios son Reliability; la idempotencia por `eventId` protege consistencia de datos, no trazabilidad de quién hizo qué (eso lo sustenta AC6-E5/E6, Accountability, sin ADR propio) — a costa de más lógica en cada escritura|
|ADR-009|Tokenización de pagos|AC6 Security|AC7 Maintainability (dependencia de la pasarela externa)|K2: reduce el alcance de cumplimiento PCI-DSS (AC6-E1), a costa de menor control directo sobre el flujo de pago|

**Síntesis (corregida — la versión anterior de esta frase se contradecía con su propia tabla):** AC7 (Maintainability) es el atributo que más veces se sacrifica en los ADRs existentes (5 de 7), aunque no de forma exclusiva — también aparece priorizado en ADR-002 y ADR-005, cuando la simplicidad de una sola base de código o de un solo esquema compartido es en sí misma la forma más mantenible de resolver el problema. Cuatro atributos no aparecen en ningún ADR: AC1 (Functional Suitability) y AC9 (Safety) son adiciones nuevas de esta versión, consistente con no tener todavía decisiones de arquitectura en torno a ellas; AC3 (Compatibility) y AC4 (Interaction Capability) en cambio existen desde la v2.0 y llevan una versión entera sin que ninguna decisión de arquitectura los haya priorizado o sacrificado — vale la pena que el equipo lo tenga presente al sustentar el documento.

---

## 7. Arquitectura de Negocio

Esta sección describe el modelo de negocio en términos conceptuales — actores, relaciones y flujos — independiente de cómo se implementa técnicamente.

### 7.1 Modelo de tenants

QUICKPATCH se concibe como una plataforma SaaS multi-tenant. La empresa dueña original de la plataforma es el primer tenant, pero el diseño contempla que otras empresas se sumen como tenants independientes a futuro, cada una con sus propios datos, usuarios, técnicos y catálogo de servicios aislados entre sí (ver D5, sustento de AC6 y AC8).

### 7.2 Actores y roles

|Rol|Descripción|Relación|
|---|---|---|
|`PLATFORM_ADMIN`|Administra la plataforma completa y los tenants (visión SaaS)|—|
|`TENANT_ADMIN`|Dueño/administrador de la empresa (el tenant)|—|
|`EMPLOYEE`|Colaborador directo de la plantilla del tenant, presta servicios|B2E|
|`ALLY`|Profesional o empresa subcontratada, no nómina directa, presta servicios|B2E / B2B2E|
|`SUPPLIER`|Proveedor de repuestos y materiales|B2B|
|`BUSINESS_CLIENT`|Empresa que contrata directo, o que subcontrata y revende|B2B / B2B2C|
|`CLIENT`|Persona natural — servicios para el hogar|B2C|

### 7.3 Relaciones entre actores

```mermaid
flowchart LR
    SUP["Proveedores de materiales<br/>(Suppliers)"] -->|venden materiales| TEN
    TEN["EMPRESA<br/>(Tenant)"] -->|coordina| ALL["Aliados / Técnicos<br/>(empleados + subcontratados)"]
    ALL -->|prestan servicio| TEN
    TEN -->|presta servicio a| HOG["Clientes hogar<br/>(B2C)"]
    TEN -->|presta servicio a| EMP["Empresas aliadas<br/>(B2B2C, subcontratan y revenden)"]
    TEN -->|presta servicio a| DIR["Empresas directas<br/>(B2B)"]
```

Las tres relaciones de negocio coexisten en el mismo sistema:

- **B2B**: el tenant le compra materiales a Suppliers, y presta servicios directos a empresas (`BUSINESS_CLIENT`).
- **B2E / B2B2E**: el tenant coordina técnicos propios (`EMPLOYEE`) y subcontratados (`ALLY`) para prestar los servicios.
- **B2C / B2B2C**: el tenant presta servicios a clientes hogar (`CLIENT`) directamente, o a través de empresas aliadas que subcontratan y revenden a sus propios clientes.

### 7.4 Ciclo de vida del servicio

Todo servicio contratado pasa por tres etapas, sin excepción (requisito explícito del negocio):

```mermaid
flowchart LR
    A[REQUESTED] --> B[QUOTED]
    B --> C[ACCEPTED]
    C --> D[IN_PROGRESS]
    D --> E[DELIVERED]
    E --> F[EVALUATED]
    A -.-> X[CANCELLED]
    B -.-> X
    C -.-> X
```

|Etapa|Qué ocurre|
|---|---|
|Cotización (`QUOTED`)|El cliente describe la necesidad; se sugieren aliados/empleados vía matching; se cotiza mano de obra + materiales|
|Prestación del servicio (`IN_PROGRESS` → `DELIVERED`)|Seguimiento en tiempo real, evidencias fotográficas, chat, checklist de trabajo|
|Evaluación de la entrega (`EVALUATED`)|El cliente valida lo entregado, califica (ranking), se libera el pago/factura|

`CANCELLED` es posible desde cualquier estado previo a `IN_PROGRESS`.

### 7.5 Modelo de facturación

La plataforma opera como **merchant of record** (modelo tipo Uber, ver D4): factura ella misma al cliente final a nombre propio, y a su vez exige a proveedores/aliados un contrato de prestación de servicios (no una factura tradicional) como soporte del pago que se les hace. Este modelo se eligió explícitamente en contraste con el modelo tipo Mercado Libre/Mercado Pago, donde cada vendedor factura por su cuenta.

```mermaid
flowchart TB
    C[Cliente paga en la app] --> P["Plataforma (tenant)<br/>= Merchant of Record"]
    P -->|genera| F[Factura al cliente<br/>a nombre de la plataforma]
    P -->|solicita| CP["Contrato de prestación de servicios<br/>al proveedor/aliado"]
```

---

## 8. Arquitectura de Datos

Esta sección describe las entidades principales del sistema agrupadas por dominio de negocio, y la estrategia de aislamiento multi-tenant. El modelo de datos completo (diagrama entidad-relación, atributos, tipos de dato y contratos de API) se desarrolla en el Documento de Datos (DD) V1, no en esta sección.

### 8.1 Dominios de entidades

|Dominio|Entidades|
|---|---|
|Identidad y Tenancy|`Tenant`, `User`, `Role`|
|Actores de negocio|`Supplier`, `Ally`, `Employee`, `Client`, `BusinessClient`|
|Catálogo|`Specialty`, `ServiceCatalogItem`, `CoverageZone`|
|Ciclo de vida del servicio|`ServiceRequest`, `Quote`, `MatchingAttempt`, `Evidence`, `Evaluation`|
|Reputación|`Ranking` (servicio), `SupplierRanking` (materiales)|
|Pagos y facturación|`Payment`, `Invoice`, `PayoutRecord`|
|Comunicación y soporte|`Conversation`, `Message`, `Notification`, `Complaint`|
|Auditoría|`AuditLog`|

### 8.2 Aislamiento multi-tenant a nivel de datos

Consistente con D5 y ADR-005 (shared-schema con `tenant_id` + Row-Level Security), toda tabla de negocio debe incluir la columna `tenant_id`, no solo las tablas raíz (`Tenant`, `User`). Esto sustenta directamente el escenario de calidad de aislamiento multi-tenant definido en AC6 — sin `tenant_id` en cada tabla, la política RLS no puede aplicarse de forma simple y consistente en todo el esquema.

---

## 9. Control de versiones

|Versión|Fecha|Descripción del cambio|
|---|---|---|
|1.0|Ago 2026|Versión inicial del SAD: drivers, killers, AC1–AC6 (21 escenarios), arquitectura de alto nivel, infraestructura, ADRs, arquitectura de negocio y de datos.|
|2.0|13 sep 2026|Se corrige el nombre del proyecto (TécnicoCerca → QUICKPATCH). Se agregan AC7–AC9 (ISO/IEC 25010:2023) y sus 6 escenarios — 27 escenarios en total. Se corrige inconsistencia entre AC1-E1 y AC4-E2 (3 segundos en ambos). Se confirma el alcance de evidencia fotográfica obligatoria (RF-15) y su storage en MinIO (VM7); se descarta Cloudflare R2 por K5. Se recortan las secciones 5.4–5.6 (Ansible, contenedores, CI/CD) para referenciar el Documento de Infraestructura V1 en vez de duplicar contenido operativo, evitando que ambos documentos se desincronicen. Se corrige una referencia rota a "ADR-008" (inexistente) en la sección 7.5.|
|2.1|13 sep 2026|Reestructuración completa de Atributos de Calidad para mapear 1:1 con las 9 características de ISO/IEC 25010:2023 (el profesor exige mínimo 1 escenario por subcaracterística, 40 en total). Se fusionan Seguridad+Trazabilidad → **AC6 Security** y Escalabilidad+Flexibilidad → **AC8 Flexibility**; se agregan **AC1 Functional Suitability** y **AC9 Safety** (esta última sin driver de negocio todavía, pendiente de decisión del equipo). Se reubican 2 escenarios mal clasificados (idempotencia → Reliability, no Accountability; cambio de proveedor de geolocalización → Flexibility, no Compatibility) y se retiran 2 escenarios redundantes. Cobertura real verificada: 18 de 40 subcaracterísticas, con etiqueta explícita en cada escenario (antes se creía 9). Se corrige la "Síntesis" de la sección 6, que se contradecía con su propia tabla. **Pendiente:** ~22 escenarios nuevos por redactar (el bloque mayor es Interaction Capability, 7 escenarios) y la votación ATAM de todo lo nuevo con cliente/stakeholders de negocio.|
|2.2|13 sep 2026|Revisión adversarial de la aplicación de 2.1: se corrige la frase de "escenarios oro" (usaba el criterio equivocado), ADR-007 (priorizaba Security sin escenario que lo sustente), el conteo de Interaction Capability (7 pendientes, no 6), se agregan etiquetas de subcaracterística faltantes (Integrity, Recoverability) para que el "18 de 40" sea auditable, se corrige la Síntesis (solo AC1 y AC9 son adiciones nuevas, no AC3/AC4), se recupera RIE-02 en el sustento de AC8 y se documenta que RIE-03 queda sin AC, y se agrega la fila de AC9 a la tabla ATAM.|
|2.3|13 sep 2026|Se corrige la nota de RIE-03 (afirmaba que Notifications es un módulo transversal; en realidad es uno de los 8 microservicios de dominio, y sí hay escenarios que lo tocan indirectamente vía AC2-E3 y AC5-E5). `CLAUDE.md` se sincroniza: ADR-007 ya no marca AC6 Security (quedó desincronizado del SAD en la corrección anterior) y la celda de AC7 en ADR-011 queda vacía en vez de asignarle un `−` que el SAD no afirma.|
|2.4|13 sep 2026|Se corrige la regla de "escenario oro" (sección 3.10): exigía Alta/Alta pero la lista real de 11 prioritarios incluye 6 Alta/Media; se reescribe para reflejar que basta con importancia de negocio Alta, y se documenta honestamente que 4 de los 11 no sustentan ningún ADR todavía y que ADR-003 se apoya también en un escenario Media/Media (AC8-E2). Se reubica la etiqueta *Integrity* de AC5-E5 (Reliability, mal ubicada) a AC6-E3 (Security, donde corresponde estructuralmente), dejando ese escenario como Confidentiality/Integrity según si la acción es de lectura o escritura. La cobertura de 18/40 subcaracterísticas se reafirma sin cambios en el conteo, ahora auditable sin la excepción estructural.|
|2.5|21 sep 2026|Se recupera la versión 2.4 (reestructuración ISO/IEC 25010:2023) tras un reset del repositorio que la había descartado (nunca se había commiteado) y se combina con el estado actual del remoto. Se actualizan las 5 referencias al Documento de Infraestructura a su nueva numeración de 14 secciones: presupuesto de recursos 4.2 → 5.6 (2 menciones), observabilidad 8 → 7, Ansible 3 → 5.1, CI/CD 5 → 6. El cambio 4.2 → 5.6 en la nota del punto único de falla de VM3 coincide con el que ya había hecho el remoto.|
|2.6|21 sep 2026|Se agregan 10 escenarios de calidad (AC1-E2, AC1-E3, AC2-E5, AC3-E2, AC5-E6, AC6-E7, AC6-E8, AC7-E4, AC7-E5, AC8-E5) que cubren *Functional Completeness*, *Functional Appropriateness*, *Resource Utilization*, *Co-existence*, *Faultlessness*, *Non-repudiation*, *Resistance*, *Analysability*, *Reusability* e *Installability*. La cobertura sube de 26 a 36 escenarios y de 18 a 28 de las 40 subcaracterísticas. Las medidas se apoyan en las herramientas del plan (k6, Loki, `kubectl`, Ansible) y son una primera propuesta pendiente de validación del equipo y de votación ATAM; AC6-E7 y AC7-E5 requieren confirmación de Backend, y la medición de AC5-E6 y AC7-E4 requiere que el access log del gateway incluya `$status` y `$request_time` y que todos los logs lleven `correlationId`. Pendiente: AC4 Interaction Capability (7) y AC9 Safety (5).|
|2.7|21 sep 2026|Se define Safety para QUICKPATCH (riesgo físico y patrimonial por fallos de la plataforma, no protección de datos, que es Security) y se agregan 8 escenarios (AC9-E1 a AC9-E8) que cubren *Operational Constraint*, *Risk Identification*, *Fail Safe*, *Hazard Warning* y *Safe Integration*. La cobertura sube de 36 a 44 escenarios y de 28 a 33 de las 40 subcaracterísticas. Los escenarios que dependen de decisiones de diseño sin cerrar (cómo el Matching conoce el estado de verificación del técnico y un atributo de riesgo en las categorías de servicio) quedan como pendiente en la sección 3.9. Pendiente: AC4 Interaction Capability (7).|
|2.8|21 sep 2026|Se agregan 7 escenarios (AC4-E3 a AC4-E9) que cubren las subcaracterísticas de Interaction Capability que faltaban: *Appropriateness Recognizability*, *Learnability*, *User Error Protection*, *User Engagement*, *User Assistance*, *Inclusivity* y *Self-descriptiveness*. La cobertura sube de 44 a 51 escenarios y de 33 a las 40 subcaracterísticas de ISO/IEC 25010:2023. Las medidas son una primera propuesta pendiente de validación del equipo y de votación ATAM; tres se miden con pruebas de usabilidad con 5 usuarios y una requiere que Frontend defina el dispositivo de referencia de gama baja.|
|2.9|22 sep 2026|Revisión de los killers tras una revisión adversarial independiente. K5 queda con una sola lectura (producción, cómputo y datos de usuarios, corre en las 7 VMs; se permiten herramientas gratuitas de desarrollo y CI; la pasarela de pagos es la única dependencia externa de producción), lo que resuelve por qué se descarta Cloudflare R2 y se usa GitHub Container Registry. K2 pasa a "Cumplimiento PCI-DSS" con el origen corregido (la norma prohíbe guardar el CVV; no guardar el PAN es decisión del equipo). K6 se precisa a Bogotá D.C. y reconoce la normativa colombiana. K7 aclara qué permite. Se agregan K9 (red privada del laboratorio sin dominio público), K10 (hardware fijo de 7 VMs) y K11 (un solo administrador de infraestructura). K4 y K8 se retiran como killers y pasan a una nueva sección 1.3 de fuera de alcance (FA1, FA2), junto con chat y reclamos (FA3). Las citas que usaban K5 como "sin VMs adicionales" pasan a K10, y se corrigen las justificaciones de ADR-005 (K3 en vez de K5) y ADR-007 (sin K5). Queda pendiente la decisión sobre cómo alcanzar el sistema desde fuera de la red del laboratorio (nota en la sección 1.2).|
|2.10|22 sep 2026|Revisión de los drivers tras una revisión adversarial independiente. D1 deja de mencionar la ventana horaria, que el modelo de datos nunca tuvo: la asignación es inmediata (se corrigen AC1-E3 y AC9-E4, que la citaban). El antiguo D4 (PCI-DSS) pasa al killer K2. Se agregan dos drivers que ya daban forma a la arquitectura sin estar declarados: seguimiento en tiempo real y evidencia fotográfica obligatoria. El doble ranking se reduce a la reputación del técnico, porque el ranking de materiales está fuera del MVP, y el merchant of record se acota a emitir el comprobante del pago. Los orígenes citan requisitos o decisiones del equipo en vez de afirmar hechos sin respaldo. Equivalencias de numeración: D1 → D1; D2 (doble ranking) → D3; D3 (merchant of record) → D4; D4 (PCI-DSS) → K2; D5 → D5; D8 → D6; nuevos D2 (tiempo real) y D7 (evidencia fotográfica). Cada driver sustenta ahora al menos un atributo de calidad o un ADR.|
|2.11|22 sep 2026|Los orígenes de los drivers y de K1 dejan de citar la presentación del Sprint 1 y citan los requisitos y las funciones del SRS. Se agrega la sección 1.4, una matriz que traza cada driver hasta sus requisitos, escenarios de calidad y ADR.|