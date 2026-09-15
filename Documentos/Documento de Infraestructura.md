# Documento de Infraestructura V1 — QUICKPATCH

| | |
|---|---|
| **Tipo de documento** | Manual operativo de infraestructura |
| **Versión** | 1.2 |
| **Curso** | Arquitectura de Software |
| **Proyecto** | QUICKPATCH |

---

## 1. Introducción y alcance

### 1.1 Propósito

Este documento describe cómo se construye, despliega y opera la infraestructura de QUICKPATCH sobre las 7 máquinas virtuales propias del proyecto. Es el manual operativo: el "cómo" concreto de llevar a producción lo que el SAD ya decidió a nivel de arquitectura.

### 1.2 Qué cubre

- Arquitectura de despliegue: qué corre en cada VM y cómo se conectan los componentes entre sí.
- Inventario detallado de las 7 VMs.
- Configuración de los ambientes Local, Dev, QA/Staging y Producción, y requisitos del ambiente local del equipo.
- Provisionamiento con Ansible, orquestación y despliegue (Docker Compose + k3s).
- Pipeline de CI/CD.
- Observabilidad (métricas y logs).
- Gestión de secretos.
- Backup y recuperación.
- Seguridad de infraestructura (TLS, puertos entre VMs, acceso administrativo).
- Dominio y DNS (y por qué el proyecto no usa un dominio público).
- Presupuesto.

### 1.3 Qué no cubre

- Decisiones de arquitectura de software y sus trade-offs (ver SAD).
- Modelo de datos y contratos de API/eventos (ver DD).
- Políticas de equipo, GitFlow y estilo de código (ver Políticas y Herramientas).
- El diagrama de despliegue conceptual y el benchmarking de infraestructura (ver Vista Física, SDD).

---

## 2. Arquitectura de despliegue

### 2.1 Vista general

Esta sección presenta la arquitectura de despliegue: qué componente de software corre en cada VM y cómo se conectan entre sí a nivel lógico. El inventario de hardware (specs, IPs) está en la sección 3; el detalle de puertos exactos y reglas de firewall está en la sección 10.2.

```mermaid
flowchart TB
    CLIENTE(["Cliente — web / móvil<br/>(dentro de la red de campus)"]) --> VM1

    subgraph VM1["VM1 — Gateway"]
        NGINX["Nginx"]
        GW["API Gateway"]
    end
    subgraph VM2["VM2 — Frontend web"]
        WEB["Next.js (panel admin)"]
    end
    subgraph VM3["VM3 — Backend (k3s, nodo único)"]
        MS["8 microservicios:<br/>Identity, Actors, Catalog, Matching,<br/>ServiceRequest, Ranking, Payments, Communication"]
    end
    subgraph VM4["VM4 — Base de datos"]
        PG["PostgreSQL + PostGIS"]
    end
    subgraph VM5["VM5 — Cache"]
        REDIS["Redis"]
    end
    subgraph VM6["VM6 — Mensajería"]
        KAFKA["Apache Kafka + Kafka UI"]
    end
    subgraph VM7["VM7 — Storage y observabilidad"]
        MINIO["MinIO"]
        OBS["Prometheus + Loki + Grafana"]
    end

    VM1 --> VM2
    VM1 --> VM3
    VM3 --> VM4
    VM3 --> VM5
    VM3 --> VM6
    VM3 -.->|"evidencias"| VM7
    VM1 -.->|"métricas / logs"| VM7
    VM2 -.->|"métricas / logs"| VM7
```

### 2.2 Principios de la arquitectura de despliegue

- **Sin servicios administrados en la nube (K5):** todo el sistema corre en las 7 VMs propias asignadas por el laboratorio de la Javeriana; no hay bases de datos, colas ni cómputo administrado por un proveedor cloud.
- **Un rol productivo por VM, salvo VM3 y VM7:** cada VM aloja un componente principal (gateway, frontend, base de datos, cache, mensajería); VM3 concentra los 8 microservicios vía k3s (ADR-011) y VM7 combina storage (MinIO) y observabilidad (Prometheus/Loki/Grafana) porque no hay presupuesto para una octava VM (K5).
- **Compose fuera de VM3, Kubernetes solo dentro de VM3:** el resto de VMs usa Docker Compose por simplicidad operativa; solo el backend justifica la complejidad de k3s, por necesitar rolling updates y auto-healing independientes por microservicio (ver sección 5).
- **Automatizado, no manual:** el aprovisionamiento de las 7 VMs y el despliegue de cada componente se hacen con Ansible (sección 5), no a mano, dado que una sola persona (DevOps) administra las 7 máquinas (SAD, sección 5.3).
- **Red cerrada por defecto:** ninguna VM acepta tráfico entrante salvo las rutas explícitas de la sección 10.2; el único punto de entrada desde fuera de las VMs es VM1.

---

## 3. Inventario de infraestructura (VMs)

### 3.1 Especificación base

Las 7 VMs son asignadas por el laboratorio de virtualización de la Pontificia Universidad Javeriana para el curso, sobre hipervisor VMware. Las 7 comparten exactamente la misma especificación de recursos; solo difieren en la IP asignada y el software que cada una aloja.

| Ítem | Valor |
|---|---|
| Proveedor | Laboratorio de virtualización, Pontificia Universidad Javeriana |
| Hipervisor | VMware |
| Sistema operativo | Ubuntu 22.04.5 LTS (Jammy Jellyfish) |
| Kernel | 6.8.0-90-generic |
| vCPU | 4 (Intel Xeon Gold 5520) |
| RAM | 11 GiB |
| Disco | 68 GB (50 GB disponibles al aprovisionar) |

> Las 7 VMs tienen la misma asignación de recursos independientemente de su rol. En particular, VM3 aloja los 8 microservicios sobre k3s con la misma RAM disponible que VM5, que solo corre Redis. El presupuesto de CPU/RAM asignado a cada microservicio (sección 5.6) es una estimación de diseño, no una medición real — todavía no existen manifiestos ni datos de carga contra el hardware real. Se vigila con las métricas de Prometheus/`node_exporter` (sección 7) contra los umbrales de AC1-E4 y AC4-E3 del SAD, y se valida de forma definitiva con el benchmarking del entregable "PoC + ADR" (sección 10).

### 3.2 Detalle por VM

| VM | Rol | IP | Software instalado |
|---|---|---|---|
| VM1 | Gateway | `10.43.100.168` | Nginx + API Gateway |
| VM2 | Frontend web | `10.43.98.15` | Next.js (panel admin) |
| VM3 | Backend | `10.43.98.205` | k3s (nodo único) — 8 microservicios: Identity, Actors, Catalog, Matching, ServiceRequest, Ranking, Payments, Communication |
| VM4 | Base de datos | `10.43.98.209` | PostgreSQL + PostGIS |
| VM5 | Cache | `10.43.98.29` | Redis |
| VM6 | Mensajería | `10.43.99.12` | Apache Kafka + Kafka UI |
| VM7 | Storage y observabilidad | `10.43.99.8` | MinIO + Prometheus + Loki + Grafana |

---

## 4. Ambientes Dev/QA/Prod

### 4.1 Resumen

| Ambiente | Dónde vive | Persistencia | Qué corre ahí |
|---|---|---|---|
| Local | Laptop de cada integrante del equipo | Persistente mientras se desarrolla, no compartido | Desarrollo día a día, lint, pruebas unitarias |
| Dev | Runner de GitHub Actions (Testcontainers) | Efímero — se crea y se destruye en cada ejecución de CI | Pruebas de integración y de contrato (Pact) sobre `develop` |
| QA / Staging | Runner de GitHub Actions (`docker-compose.staging.yml`) + VM3 real para la prueba de carga | Efímero (funcional) / real y programado (carga) | E2E, UAT, seguridad (automático) — carga con k6 (manual, ventana programada) |
| Producción | Las 7 VMs | Persistente, siempre activo | El sistema completo, tráfico real |

Ningún ambiente además de Producción ocupa hardware dedicado y permanente — es la consecuencia directa de K5 (sin presupuesto para VMs adicionales): Dev y la parte funcional de Staging existen solo durante la ejecución del pipeline, no como servidores que alguien tiene que mantener corriendo.

### 4.2 Requisitos del ambiente local

_Pendiente de confirmar con Backend y Frontend: versión de Node.js, versión del SDK de Flutter, gestor de paquetes y forma de fijar la versión en el repo (`.nvmrc` u otro). Se completa cuando el equipo lo confirme._

---

## 5. Contenedores y orquestación

Esta sección documenta el **plan** de provisionamiento, orquestación y despliegue: ni los playbooks de Ansible ni los manifiestos de despliegue están implementados todavía; se desarrollan durante el Sprint 3, siguiendo la estructura definida aquí y en el SAD (sección 5.4). Los límites de recursos definidos en 5.6 surgieron de una revisión crítica de la capacidad real de VM3 (ver `Hallazgos - Capacidad de Infraestructura.md`) y quedan pendientes de confirmación por parte de Backend antes de implementarse en código.

### 5.1 Inventario de Ansible

```ini
[gateway]
vm1 ansible_host=10.43.100.168

[frontend]
vm2 ansible_host=10.43.98.15

[backend]
vm3 ansible_host=10.43.98.205

[database]
vm4 ansible_host=10.43.98.209

[cache]
vm5 ansible_host=10.43.98.29

[messaging]
vm6 ansible_host=10.43.99.12

[storage_observability]
vm7 ansible_host=10.43.99.8
```

### 5.2 Playbooks planeados

| Playbook | Alcance | Qué instala/configura |
|---|---|---|
| `setup-base.yml` | Las 7 VMs | Docker Engine + plugin de Compose, dependencias comunes, usuario de despliegue y llaves SSH, **`node_exporter` y Promtail** (agentes de métricas y logs) |
| `deploy-db.yml` | VM4 | PostgreSQL + extensión PostGIS, bases y roles iniciales |
| `deploy-kafka.yml` | VM6 | Apache Kafka + Kafka UI vía Docker Compose |
| `deploy-k3s.yml` | VM3 | Instalación de k3s (nodo único) con `--service-cidr=10.44.0.0/16 --cluster-cidr=10.42.0.0/16 --cluster-dns=10.44.0.10` (ver nota de red abajo), configuración de `kubeconfig`, y bootstrap inicial de los 8 microservicios (ver 5.3) |
| `deploy-cache.yml` | VM5 | Redis vía Docker Compose |
| `deploy-gateway.yml` | VM1 | Nginx + configuración del API Gateway |
| `deploy-runner.yml` | VM1 | Registro e instalación del runner self-hosted de GitHub Actions (servicio systemd), copia del `kubeconfig` de VM3 |
| `deploy-frontend.yml` | VM2 | Contenedor de Next.js vía Docker Compose |
| `deploy-storage-observability.yml` | VM7 | MinIO + Prometheus + Loki + Grafana vía Docker Compose (los componentes centrales; `node_exporter`/Promtail van en `setup-base.yml`, no aquí) |

`setup-base.yml` corre primero y en las 7 VMs por igual; los ocho restantes son específicos de cada rol y en general solo tocan su propia VM (vía los grupos del inventario), de forma que aplicar o repetir uno no afecta a las demás — la única excepción es `deploy-runner.yml`, que además copia el `kubeconfig` generado por `deploy-k3s.yml` (sección 5.4).

> **Nota crítica sobre el rango de red de k3s:** el `--service-cidr` por defecto de k3s es `10.43.0.0/16` — exactamente el rango en el que viven las 7 VMs del laboratorio (`10.43.98.x`, `10.43.99.x`, `10.43.100.x`). Sin cambiarlo, un `ClusterIP` que k3s asigne a un `Service` puede coincidir con la IP real de otra VM (por ejemplo, `10.43.98.209`, la IP de VM4): el tráfico de un pod hacia esa IP se enrutaría al Service en vez de a PostgreSQL, un fallo intermitente y dependiente del orden de asignación de IPs, no reproducible de forma confiable. Por eso `deploy-k3s.yml` instala k3s con `--service-cidr` y `--cluster-cidr` fijados fuera de `10.43.0.0/16` (ver tabla arriba) — este flag se fija en el momento de instalación y no se puede cambiar después sin reinstalar el clúster.

> **Nota sobre Pino:** Pino no se instala en ningún lado — es la librería de logging que cada microservicio NestJS usa internamente para escribir sus logs en formato JSON a la salida estándar del contenedor. **Loki** (en VM7) es el componente real que los agrega y almacena; **Promtail** (instalado en las 7 VMs desde `setup-base.yml`) es el agente que los recolecta desde cada contenedor y se los envía a Loki. Grafana consulta Loki igual que consulta Prometheus, en la misma interfaz. El detalle de este flujo se documenta en la sección 7 (Monitoreo y observabilidad).

### 5.3 Bootstrap inicial de los microservicios

Instalar k3s deja el clúster corriendo, pero vacío — ningún microservicio arranca solo. `deploy-k3s.yml` incluye, después de instalar k3s, los pasos para dejar los 8 microservicios corriendo por primera vez:

1. Crear el namespace del proyecto (`kubectl create namespace quickpatch`).
2. Cargar los `Secret` de Kubernetes con las credenciales necesarias (contraseñas de base de datos, llaves de firma JWT, credenciales de la pasarela de pagos) — nunca en texto plano en los manifiestos.
3. Cargar los `ConfigMap` con configuración no sensible (URLs internas de Kafka, Redis, etc.).
4. Crear el `imagePullSecret` con el token de acceso a `ghcr.io` (ver sección 5.8), para que k3s pueda descargar las imágenes de los 8 servicios.
5. Aplicar los 8 `deployment.yaml` + `service.yaml` de `infrastructure/vm3-k3s/` (sección 5.5) por primera vez.

Después de este bootstrap, cualquier actualización posterior usa el flujo normal de la sección 5.8 (`kubectl set image`), que no repite estos pasos.

### 5.4 Ejecución planeada

```bash
ansible-playbook -i inventory.ini setup-base.yml
ansible-playbook -i inventory.ini deploy-db.yml
ansible-playbook -i inventory.ini deploy-cache.yml
ansible-playbook -i inventory.ini deploy-kafka.yml
ansible-playbook -i inventory.ini deploy-k3s.yml
ansible-playbook -i inventory.ini deploy-gateway.yml
ansible-playbook -i inventory.ini deploy-runner.yml
ansible-playbook -i inventory.ini deploy-frontend.yml
ansible-playbook -i inventory.ini deploy-storage-observability.yml
```

El orden respeta las dependencias de arranque ya definidas en el SAD (sección 5.3): base de datos y cache antes que mensajería, mensajería antes que el backend, backend antes que frontend/gateway. `deploy-runner.yml` corre después de `deploy-k3s.yml` porque necesita copiar el `kubeconfig` que ese playbook genera. Storage y observabilidad (VM7) es independiente y puede correr en cualquier momento.

### 5.5 Estructura de archivos de despliegue

```
infrastructure/
├── vm1-gateway/docker-compose.yml
├── vm2-web/docker-compose.yml
├── vm3-k3s/
│   ├── identity/deployment.yaml + service.yaml
│   ├── actors/deployment.yaml + service.yaml
│   ├── catalog/deployment.yaml + service.yaml
│   ├── matching/deployment.yaml + service.yaml
│   ├── service-request/deployment.yaml + service.yaml
│   ├── ranking/deployment.yaml + service.yaml
│   ├── payments/deployment.yaml + service.yaml
│   └── communication/deployment.yaml + service.yaml
├── vm4-database/docker-compose.yml
├── vm5-redis/docker-compose.yml
├── vm6-kafka/docker-compose.yml
└── vm7-storage-observability/docker-compose.yml
```

Solo VM3 usa Kubernetes (k3s); el resto de VMs usa Docker Compose, consistente con ADR-011.

### 5.6 Presupuesto de recursos en VM3

VM3 tiene 4 vCPU y 11 GiB de RAM fijos (sección 3.1), compartidos entre el control plane de k3s y los 8 microservicios. Sin límites explícitos, Node/V8 y el JVM asumen que tienen toda la máquina disponible, lo que puede llevar a que el sistema operativo mate procesos por falta de memoria durante un pico de carga — y que la primera víctima sea, por cómo Kubernetes decide a quién desalojar, el propio Matching Service. Para evitarlo, cada microservicio declara `resources.requests` y `resources.limits` explícitos:

| Componente | CPU (request/limit) | RAM (request/limit) | Clase de QoS |
|---|---|---|---|
| k3s + containerd + Traefik + CoreDNS (sistema) | — | — | reservado, ~0.5 vCPU / ~2 GiB de margen |
| Matching Service (Spring Boot) | 1 / 1 vCPU | 1 / 1 GiB | Guaranteed |
| Identity, Actors, Catalog, ServiceRequest, Ranking, Payments, Communication (NestJS, c/u) | 0.1 / 0.35 vCPU | 256Mi / 512Mi | Burstable |

Con los 7 servicios NestJS en su límite máximo (2.45 vCPU / 3.5 GiB) más el Matching (1 vCPU / 1 GiB) más el overhead de sistema (0.5 vCPU / 2 GiB), el uso máximo teórico es de **3.95 vCPU / 6.5 GiB de 4 vCPU / 11 GiB disponibles** — deja margen para el pod adicional que se crea durante un rolling update y para escalar el Matching a una segunda réplica (AC4-E3) sin agotar la máquina.

```mermaid
pie title Presupuesto de RAM en VM3 (11 GiB)
    "Sistema (k3s, containerd, Traefik, CoreDNS)" : 2
    "Matching Service (Guaranteed)" : 1
    "7 servicios NestJS (límite máximo)" : 3.5
    "Margen libre (rolling update / escalado)" : 4.5
```

**Por qué el Matching queda en "Guaranteed":** cuando VM3 entra en presión de memoria, Kubernetes desaloja primero los pods en QoS "BestEffort" (sin límites) y luego los "Burstable" que más exceden su *request*; un pod "Guaranteed" (request = limit) es el último candidato a desalojo. AC1-E4 exige que el Matching siga respondiendo bajo pico de carga — dejarlo en la clase de QoS más protegida es lo que hace esa exigencia verificable, no solo declarada.

### 5.7 Límite de conexiones a PostgreSQL

VM4 corre PostgreSQL con `max_connections` por defecto (100). Sin restricción, cada instancia de un servicio NestJS con Prisma abre por defecto varias conexiones simultáneas, y sumadas a las del Matching (HikariCP) pueden acercarse al límite incluso sin haber escalado ningún servicio. Se fija explícitamente, vía variable de entorno en cada `deployment.yaml` (sin tocar el código de los servicios):

| Servicio | Variable | Valor |
|---|---|---|
| Los 7 servicios NestJS | `DATABASE_URL=...?connection_limit=5` | 5 conexiones c/u → 35 en total |
| Matching (Spring Boot) | `SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE` | 10 |
| Matching (Spring Boot) | `SERVER_TOMCAT_THREADS_MAX` | 50 |

Con esto, el uso base de conexiones queda en ~45 de 100, dejando margen real para escalar el Matching (AC4-E3) sin llegar al error `too many clients`. El límite de hilos de Tomcat (50, en vez del valor por defecto de 200) evita que el servicio abra más hilos de los que su pool de conexiones puede atender, que era la causa del *thrashing* bajo carga identificado en la revisión de capacidad.

### 5.8 Cómo se despliega o actualiza un microservicio

1. Se construye la imagen Docker del servicio modificado y se sube a **GitHub Container Registry (`ghcr.io`)** — gratuito e integrado con el repositorio del proyecto, sin credenciales adicionales que gestionar.
2. Se actualiza el `deployment.yaml` correspondiente (o se ejecuta `kubectl set image deployment/<servicio> <contenedor>=<nueva-imagen>`).
3. Kubernetes aplica un *rolling update*: crea el pod nuevo, espera a que pase el `readinessProbe`, y solo entonces retira el pod anterior — sin downtime para ese servicio ni para los demás 7.
4. El margen de RAM definido en la sección 5.6 es lo que permite que el pod adicional del rolling update quepa sin desalojar a otros servicios.

**Migraciones de base de datos — por qué deben ser *expand-contract*:** `kubectl rollout undo` (sección 6.2) revierte únicamente la imagen del contenedor a la versión anterior; **nunca revierte una migración de esquema** (Prisma/Flyway). Si un release incluye una migración que rompe compatibilidad hacia atrás (por ejemplo, elimina o renombra una columna que la versión anterior todavía usa), un rollback de imagen deja código viejo corriendo contra un esquema nuevo — el sistema queda roto, no recuperado. Por eso toda migración de esquema debe seguir el patrón **expand-contract**: agregar lo nuevo (columna, tabla) sin tocar ni eliminar lo que la versión anterior todavía necesita; solo se retira lo viejo en un release posterior, una vez que ningún código en producción depende de ello. Bajo esta regla, un `kubectl rollout undo` sí es seguro: la versión anterior del código sigue siendo compatible con el esquema, ya ampliado, de la base de datos. Si una migración no puede expresarse de forma expand-contract dentro del alcance de un sprint, no se despliega junto con el resto del release — se separa en su propio cambio, con downtime planeado y comunicado (escenario de mantenimiento planeado del SAD).

**Pendiente de definir con datos reales:** los parámetros de `readinessProbe`/`livenessProbe` (tiempos de espera, número de reintentos) y si se necesita un `startupProbe` separado para el arranque. No se fijan valores en esta versión del documento porque dependen del comportamiento real de cada servicio bajo carga, algo que todavía no se ha medido — fijar un número ahora sería una estimación sin sustento, el mismo problema señalado en la sección 3.1.

---

## 6. CI/CD

### 6.1 Estructura del pipeline

Pipeline en GitHub Actions, separado por aplicación (`web/`, `mobile/`) y por microservicio dentro del backend (`services/identity/`, `services/matching/`, etc.). Un cambio en un solo servicio dispara solo su propio pipeline, no el de los 8 — esto es lo que hace posible cumplir AC5-E1 (build + pruebas en menos de 10 minutos) incluso con pruebas de integración pesadas en el gate de `develop`.

### 6.2 Gates por rama

| Rama | Dónde corre | Qué corre | Gate |
|---|---|---|---|
| `feature/*` → PR a `develop` | Runner de GitHub Actions | Lint + unitarias (Jest, `flutter_test`) del servicio afectado | CI verde + 1 revisor, sin autoaprobación |
| `develop` | Runner de GitHub Actions | Integración con Testcontainers (PostgreSQL/PostGIS y Kafka reales en Docker) + contrato (Pact) | CI verde |
| `release/x.y.z` | Runner de GitHub Actions, levantando `docker-compose.staging.yml` | E2E (Playwright, Patrol), UAT contra criterios de aceptación, seguridad (OWASP ZAP) | Checklist de aceptación aprobado — bloquea el merge a `main` si falla (RNF-08) |
| `main` (despliegue) | Runner self-hosted (VM1) → k3s en VM3 | Rolling update del servicio modificado | Tag SemVer |
| `main` (post-despliegue) | VM3 real, ventana de mantenimiento programada, inmediatamente después del despliegue | Smoke tests + k6 — 150 matchings concurrentes (AC1-E4), contra tenant de prueba dedicado | Si no cumple el umbral, reversión de imagen (`kubectl rollout undo` — no revierte esquema, ver detalle abajo) antes de habilitar tráfico real |

```mermaid
flowchart LR
    F["feature/*"] --> G1{{"Lint + unitarias<br/>runner GH Actions"}}
    G1 --> D["develop"]
    D --> G2{{"Testcontainers + Pact<br/>runner GH Actions"}}
    G2 --> R["release/x.y.z"]
    R --> G3{{"E2E + UAT + OWASP ZAP<br/>docker-compose.staging.yml<br/>runner GH Actions — automático"}}
    G3 -->|"checklist aprobado"| M["main"]
    M --> SH{{"Runner self-hosted en VM1<br/>(dentro de la red privada)"}}
    SH --> P["k3s en VM3<br/>rolling update por servicio"]
    P --> G4{{"Smoke tests + k6<br/>150 concurrentes<br/>ventana de mantenimiento — MANUAL"}}
    G4 -->|"cumple umbral"| OK(["Tráfico real habilitado"])
    G4 -->|"no cumple"| RB{{"kubectl rollout undo"}}
    RB --> P
```

El gate de `release/x.y.z` funcional (E2E/UAT/seguridad) corre automáticamente dentro del runner de GitHub Actions: `docker compose -f docker-compose.staging.yml up -d`, se ejecutan las pruebas contra esa copia efímera, y el runner la apaga al terminar. No depende de que un miembro del equipo la tenga levantada en su laptop — eso queda como opción para depurar manualmente, no como el mecanismo del gate. Este es el único gate que bloquea el merge a `main` (RNF-08, parte funcional/seguridad).

La prueba de carga (k6) **no puede correr antes del despliegue** porque no existe una segunda instancia de VM3 donde probar la versión nueva sin desplegarla primero (K5). Por eso corre justo después del rolling update, dentro de la misma ventana de mantenimiento sin usuarios activos (RNF-07): si el reporte no cumple el umbral de AC1-E4, se revierte con `kubectl rollout undo` antes de que haya tráfico real — la reversión es casi instantánea **para la imagen del contenedor**, porque Kubernetes ya conoce la versión anterior y no hay que reconstruir nada.

**Alcance real de esta reversión:** `kubectl rollout undo` no revierte migraciones de esquema de base de datos — solo es "casi instantánea" si el release siguió el patrón expand-contract exigido en la sección 5.8. Si no lo siguió, la recuperación real es restaurar el backup de PostgreSQL (sección 9.3), con el RPO de hasta 24h que eso implica — no un rollback instantáneo. Por eso todo release con migración de esquema debe validar ese patrón antes de llegar a este gate.

**Datos de la prueba de carga:** los 150 matchings concurrentes de k6 corren contra la base de datos real de producción (VM4), dentro de una cuenta/tenant de prueba dado de alta solo para esta prueba — nunca contra datos de tenants reales. Las solicitudes, pagos y calificaciones que genera esa corrida se identifican por ese tenant de prueba y se limpian con un script inmediatamente después, para no contaminar reportes ni facturación real.

### 6.3 Despliegue a producción

Los runners de GitHub Actions corren en la nube de GitHub y no tienen ruta de red hacia `10.43.x.x` (red privada del laboratorio): ninguna dirección de ese rango es alcanzable desde fuera de la red de la Javeriana, sin importar el protocolo. Por eso el paso de despliegue no puede ejecutarse en un runner normal — se ejecuta en un **runner self-hosted instalado en VM1**, que sí está dentro de esa red y puede llegar a VM3 directamente.

Al mergear a `main` (tras pasar el gate funcional de `release/*`): build de la imagen Docker del servicio modificado, push al registro, y — ya dentro del runner self-hosted — `kubectl set image deployment/<servicio> ...` (o `kubectl apply -f`). Kubernetes hace el rolling update descrito en la sección 5.8 — reinicia únicamente el Deployment del servicio afectado, sin downtime para los demás 7. Inmediatamente después corre la validación de la sección 6.2 (smoke tests + k6); un fallo dispara `kubectl rollout undo` sobre ese mismo Deployment antes de habilitar tráfico real.

**Por qué en VM1 y no en VM3 ni en una laptop:** VM3 ya opera con el presupuesto de recursos ajustado de la sección 5.6, y sumarle el proceso del runner competiría por esa misma RAM/CPU justo durante el despliegue. VM1 solo corre Nginx y tiene margen de sobra; al estar en la misma red privada que VM3, no necesita ningún túnel ni VPN adicional para alcanzarla. Una laptop del equipo no sirve para esto porque el runner tiene que estar disponible siempre que se necesite desplegar, no solo cuando esa persona esté conectada.

**Restricción de seguridad obligatoria:** el repositorio es público. GitHub advierte explícitamente que un runner self-hosted en un repositorio público es un riesgo — cualquier persona puede abrir un Pull Request desde un fork, y si ese PR dispara un workflow que corre en el runner self-hosted, ese código ajeno se ejecuta dentro de la red de la Javeriana. El job que usa `runs-on: self-hosted` debe restringirse a eventos de `push` directo sobre `main`/`release/*` — nunca a `pull_request` proveniente de un fork externo.

---

## 7. Monitoreo y observabilidad

Dos tipos de información, cada uno con su propia cadena de recolección y almacenamiento, unificados en un solo panel:

- **Métricas** (números que cambian en el tiempo: CPU, RAM, peticiones por segundo, errores por minuto) — recolectadas por `node_exporter` en cada VM, almacenadas por Prometheus en VM7.
- **Logs** (mensajes de texto que describen eventos: "pago rechazado", "acceso denegado") — recolectados por Promtail en cada VM, almacenados por Loki en VM7.

Ninguno de los dos sustituye al otro: las métricas dicen *qué tan rápido u ocupado* está el sistema; los logs dicen *qué pasó exactamente*. Grafana consulta ambas fuentes desde el mismo panel, lo que permite cruzarlos — por ejemplo, ver qué decían los logs justo cuando una métrica de CPU se disparó.

```mermaid
flowchart LR
    subgraph VMs["Las 7 VMs"]
        NE["node_exporter<br/>(métricas de la VM)"]
        PT["Promtail<br/>(logs de los contenedores)"]
    end
    NE -->|"scrape cada ~15s"| PROM["Prometheus<br/>(VM7)"]
    PT -->|"envío continuo"| LOKI["Loki<br/>(VM7)"]
    PROM --> GRAF["Grafana<br/>(VM7)"]
    LOKI --> GRAF
```

### 7.1 Qué se instala y dónde

| Componente | Dónde | Rol |
|---|---|---|
| `node_exporter` | Las 7 VMs (`setup-base.yml`) | Expone las métricas de esa VM para que Prometheus las recolecte |
| Promtail | Las 7 VMs (`setup-base.yml`) | Recolecta los logs de los contenedores de esa VM y los envía a Loki |
| Prometheus | VM7 (`deploy-storage-observability.yml`) | Almacena el historial de métricas de las 7 VMs |
| Loki | VM7 (`deploy-storage-observability.yml`) | Almacena el historial de logs de las 7 VMs |
| Grafana | VM7 (`deploy-storage-observability.yml`) | Panel único para consultar Prometheus y Loki |

### 7.2 Qué cubre esto en el SRS y el SAD

- **RNF-04** (todo 403 queda en log): el 403 se escribe con Pino/el logger del servicio, Promtail lo recolecta, Loki lo guarda permanentemente — sin esta cadena, el log existiría solo mientras el contenedor no se reinicie, lo cual pasa en cada despliegue.
- **AC6-E1/E2** (reconstruir una disputa, trazabilidad de pagos): requieren historial persistente de eventos — Loki es lo que hace posible que ese historial sobreviva más allá de la vida de un contenedor.
- **AC3-E1/E3** (disponibilidad, "límite de capacidad a vigilar" de la sección 3.1): Prometheus + `node_exporter` son el instrumento real para vigilar esa capacidad — sin ellos, "vigilar" no tenía con qué hacerse.

---

## 8. Gestión de secretos

### 8.1 Principio general

Ningún secreto (contraseñas, tokens de la pasarela de pagos, credenciales de MinIO, llaves de firma JWT) se guarda en el repositorio ni en las imágenes Docker — consistente con la política del equipo de no compartir credenciales reales, ni siquiera con la IA (Políticas y Herramientas, sección de uso de IA).

### 8.2 Mecanismo por tipo de despliegue

- **VM1, VM2, VM4, VM5, VM6, VM7** (Docker Compose): secretos gestionados con Ansible Vault — archivos cifrados dentro del propio repositorio de Ansible, que se desencriptan solo al momento de ejecutar el playbook.
- **VM3** (k3s): secretos como objetos `Secret` de Kubernetes, referenciados desde cada `deployment.yaml` como variable de entorno — nunca escritos en texto plano en el manifiesto.
- **CI/CD:** credenciales del registro de imágenes y del `kubeconfig` de despliegue, como GitHub Actions Secrets del repositorio.

### 8.3 Inventario de secretos

| Secreto | Usado por | Dónde se gestiona |
|---|---|---|
| Credenciales de PostgreSQL | Los 8 microservicios (vía `DATABASE_URL`) | `Secret` de Kubernetes (VM3) |
| Llaves de firma JWT | Identity, y validación de token en el resto de servicios | `Secret` de Kubernetes (VM3) |
| Credenciales de la pasarela de pagos | Payments | `Secret` de Kubernetes (VM3) |
| Credenciales de MinIO (access/secret key) | Servicios que suben evidencias, y `pg_dump`/backup (sección 9.2) | Ansible Vault (VM7) + `Secret` de Kubernetes (VM3) |
| Token de acceso a `ghcr.io` | k3s (`imagePullSecret`, sección 5.3) y pipeline de CI/CD | GitHub Actions Secret + `Secret` de Kubernetes (VM3) |
| `kubeconfig` de VM3 | Runner self-hosted (VM1, sección 6.3) | GitHub Actions Secret, copiado a VM1 por `deploy-runner.yml` |

---

## 9. Backup y recuperación

### 9.1 Qué se respalda y qué no

| Dato | ¿Se respalda? | Por qué |
|---|---|---|
| PostgreSQL (VM4) | **Sí** | Es la fuente de verdad del negocio — tenants, usuarios, solicitudes, pagos, calificaciones, evidencias registradas. Perderla es perder la plataforma. |
| Evidencias en MinIO (VM7) | Pendiente de decisión — ver 9.4 | Son archivos, no filas de base de datos; respaldarlas requiere otro lugar donde guardarlas. |
| Kafka (VM6) | No | Es un bus de mensajes en tránsito, no un sistema de registro. El patrón Outbox (ADR-007) ya garantiza que un evento no se pierde entre que se genera y se publica — no hace falta guardar el historial completo de Kafka aparte. |
| Redis (VM5) | No | Cache y colas cortas — se reconstruye solo con el uso normal del sistema, no guarda información que no exista también en otro lado. |
| Configuración de infraestructura (Ansible, manifiestos k3s) | No hace falta un respaldo aparte | Ya vive versionada en el repositorio de Git — el repositorio es su respaldo. |

### 9.2 Respaldo de PostgreSQL

- **Mecanismo:** `pg_dump` programado por cron en VM4, una vez al día.
- **Destino:** un bucket de MinIO en VM7 — VM distinta a VM4, así un problema en VM4 no se lleva también su propio respaldo.
- **Retención:** 7 días. Suficiente para el alcance de un proyecto académico de 3 meses (K3); no se justifica una política de retención de nivel empresarial para este contexto.

### 9.3 Recuperación

Restaurar desde el dump más reciente con `pg_restore`. Dado K7 (sin operación 24/7), esta recuperación es **manual**, dentro de la misma ventana de 12–24h ya aceptada para el resto de fallos de infraestructura (ver SAD, sección 5.2) — no hay un mecanismo de restauración automática.

### 9.4 Evidencias en MinIO — sin respaldo, limitación aceptada

MinIO en VM7 es tanto el almacenamiento principal de las evidencias fotográficas como, si algo le pasa a esa VM, el único lugar donde existían — no hay una octava VM para duplicar el storage (K5), y un respaldo manual dependiente de una sola persona no es un mecanismo confiable ni reproducible por el resto del equipo.

**Se documenta como limitación aceptada**, con el mismo tratamiento que el SPOF de VM3 (SAD, sección 5.2): si VM7 falla por completo, las evidencias no respaldadas se pierden. El riesgo residual es la pérdida de evidencia fotográfica de servicios ya completados, no la caída de la plataforma — el ciclo de negocio (RF-15, pagos, calificaciones) no depende de que la evidencia siga disponible después de completado el servicio.

---

## 10. Seguridad de infraestructura

### 10.1 TLS / HTTPS

RNF-02 exige que toda comunicación cliente-servidor use HTTPS/TLS. Dado que el sistema no tiene un dominio público (ver sección 11, Dominio y DNS), **no es posible obtener un certificado de Let's Encrypt**: su proceso de validación (retos HTTP-01/DNS-01) exige que el dominio resuelva públicamente hacia el servidor, y las 7 VMs solo son alcanzables dentro de la red privada del laboratorio (`10.43.x.x`).

En su lugar, VM1 (Nginx) sirve HTTPS con un **certificado autofirmado**, generado e instalado desde `setup-base.yml`. Esto satisface RNF-02 literalmente — el tráfico cliente-servidor va cifrado — aunque el navegador del cliente muestre una advertencia de certificado no confiable la primera vez, dado que no proviene de una CA públicamente reconocida. Se documenta como limitación aceptada (sección 13): la norma exige cifrado, no una cadena de confianza pública, y no hay presupuesto (K5) para una CA comercial cuando el acceso ya está restringido a la red del laboratorio.

El tráfico interno entre VMs, dentro de la misma red privada de la Javeriana, no usa TLS: es tráfico que nunca sale a Internet, y cifrarlo agregaría gestión de certificados internos sin un beneficio real dado que la red ya es privada.

### 10.2 Puertos y comunicación entre VMs

```mermaid
flowchart TB
    INET(["Internet / red de campus"]) -->|"443 HTTPS"| VM1
    VM1["VM1 — Gateway<br/>Nginx + API Gateway"] -->|"3000"| VM2["VM2 — Next.js"]
    VM1 -->|"NodePort 30080/30443<br/>Traefik (k3s)"| VM3["VM3 — k3s<br/>8 microservicios"]
    VM1 -->|"6443 — API server<br/>(runner self-hosted, sección 6.3)"| VM3
    VM3 -->|"5432"| VM4["VM4 — PostgreSQL + PostGIS"]
    VM3 -->|"6379"| VM5["VM5 — Redis"]
    VM3 -->|"9092"| VM6["VM6 — Kafka"]
    VM3 -->|"9000"| VM7["VM7 — MinIO"]
    VM4 -->|"9000 — backup diario"| VM7
    VM7 -.->|"9100 — scrape node_exporter"| TODAS(["Las 7 VMs"])
    TODAS -.->|"3100 — Promtail → Loki"| VM7
    VM6 -.->|"8080 Kafka UI"| EQ(["Equipo — acceso interno"])
    VM7 -.->|"3000 Grafana / 9090 Prometheus"| EQ
```

| Origen | Destino | Puerto | Servicio |
|---|---|---|---|
| Internet / red de campus | VM1 | 443 | Nginx / API Gateway (TLS) |
| VM1 | VM2 | 3000 | Next.js |
| VM1 | VM3 | 30080 / 30443 | Ingreso a microservicios (Traefik, NodePort de k3s) |
| VM1 | VM3 | 6443 | API server de k3s — usado por `kubectl` desde el runner self-hosted (sección 6.3) para desplegar |
| VM3 | VM4 | 5432 | PostgreSQL |
| VM3 | VM5 | 6379 | Redis |
| VM3 | VM6 | 9092 | Kafka (productores/consumidores) |
| VM3 | VM7 | 9000 | MinIO (subir evidencias) |
| VM4 | VM7 | 9000 | MinIO — subida del backup diario de `pg_dump` (sección 9.2) |
| VM7 | Las 7 VMs | 9100 | Prometheus haciendo scrape de `node_exporter` en cada VM |
| Las 7 VMs | VM7 | 3100 | Promtail enviando logs a Loki |
| Equipo (interno) | VM6 | 8080 | Kafka UI |
| Equipo (interno) | VM7 | 3000 / 9090 | Grafana / Prometheus |
| Equipo (SSH) | Las 7 VMs | 22 | Administración |

Regla general: firewall por defecto en denegar todo el tráfico entrante (`ufw default deny incoming`), habilitando explícitamente solo los pares origen-destino-puerto de esta tabla. Ninguna VM acepta conexiones directas del exterior salvo VM1 (443) y el acceso SSH del equipo. Los puertos internos de k3s (VXLAN de flannel en 8472/UDP, API del kubelet en 10250) no aparecen en esta tabla porque VM3 es un clúster de un solo nodo: ese tráfico es local a la propia máquina, no cruza la red entre VMs, y `ufw` no bloquea tráfico de loopback por defecto.

### 10.3 Acceso administrativo (SSH)

Autenticación únicamente por llave pública — login por contraseña deshabilitado en las 7 VMs desde `setup-base.yml`.

---

## 11. Dominio y DNS

### 11.1 Alcance: sin dominio público

Las 7 VMs del proyecto viven en el rango privado `10.43.x.x` del laboratorio de virtualización de la Javeriana (sección 3), no enrutable desde Internet. No hay NAT, port-forwarding ni un gateway público administrado por el equipo que exponga VM1 hacia afuera de la red del laboratorio — eso está fuera del control del equipo y del alcance de este proyecto académico. En consecuencia, **el proyecto no usa un dominio público real ni un proveedor de DNS público** (Cloudflare, Route 53, GoDaddy, etc.).

### 11.2 Nombre lógico y resolución interna

Para referirse al sistema de forma legible en documentación, configuración y pruebas (en vez de escribir la IP `10.43.100.168` en todos lados), se usa el nombre lógico `quickpatch.internal`:

- Se configura como `server_name quickpatch.internal;` en la configuración de Nginx de VM1.
- Se resuelve únicamente mediante entradas manuales en el archivo `/etc/hosts` (o `C:\Windows\System32\drivers\etc\hosts`) de cada máquina del equipo que necesite acceder al sistema (`10.43.100.168 quickpatch.internal`), o mediante el DNS interno que el laboratorio de la Javeriana provea para la red `10.43.x.x`, si existe.
- **No es un registro DNS público** — nadie fuera de esas máquinas configuradas puede resolver `quickpatch.internal`, y el nombre no tiene validez ni existencia fuera de este proyecto.

### 11.3 Consecuencia sobre TLS

Al no existir un dominio público, no es posible tramitar un certificado válido emitido por una CA públicamente reconocida (ver sección 10.1). El certificado autofirmado servido por VM1 se emite para el nombre `quickpatch.internal` y para la IP `10.43.100.168`, consistente con la resolución de la sección 11.2.

### 11.4 Fuera de alcance: dominio real en un despliegue de producción

Si el proyecto pasara de entorno académico a un despliegue real de producción, se necesitaría: (1) un dominio público comprado a un registrador, (2) un proveedor de DNS público, y (3) salir de la red privada del laboratorio hacia infraestructura con IP pública o un túnel administrado — momento en el cual sí sería viable tramitar un certificado real de Let's Encrypt. Esta migración está fuera del alcance de este documento y de las restricciones académicas K3/K5.

---

## 12. Presupuesto

### 12.1 Nota metodológica

**Los valores monetarios de esta sección son estimaciones ilustrativas**, construidas por el equipo para efectos de este documento académico — no son cotizaciones reales solicitadas a ningún proveedor. Cada cifra estimada se marca explícitamente como **(estimado)**; las cifras sin esa marca son costos reales verificables (por ejemplo, "$0, plan gratuito").

### 12.2 Costo real del proyecto (académico)

| Rubro | Costo real | Detalle |
|---|---|---|
| 7 VMs (cómputo, red, almacenamiento) | $0 | Asignadas por el laboratorio de virtualización de la Pontificia Universidad Javeriana para el curso (sección 3.1) |
| Herramientas de colaboración (Jira, Figma, Miro) | $0 | Planes Free — ver Políticas y Herramientas, tabla de planes gratuitos |
| GitHub + GitHub Actions | $0 | Repositorio público; minutos de Actions ilimitados en repos públicos |
| Software de infraestructura (Docker, PostgreSQL, PostGIS, Redis, Kafka, MinIO, Prometheus, Loki, Grafana, Nginx, k3s, Ansible) | $0 | Todo open source, self-hosted |
| Registro de imágenes (GitHub Container Registry) | $0 | Incluido con el repositorio de GitHub |
| Dominio y DNS público | $0 | No aplica — el proyecto no usa dominio público (sección 11) |
| Certificado TLS | $0 | Autofirmado (sección 10.1) |
| **Total real** | **$0** | Todo el costo de infraestructura y herramientas queda cubierto por recursos gratuitos o asignados por la universidad, consistente con K5 |

### 12.3 Costo estimado si se contratara en la nube (referencia, no aplica al proyecto)

Solo para dimensionar qué costaría este mismo diseño fuera del contexto académico, si las 7 VMs se reemplazaran por instancias equivalentes en un proveedor cloud comercial (especificación aproximada por VM: 4 vCPU / 11 GiB):

| Rubro | Costo estimado (USD/mes) | Nota |
|---|---|---|
| 7 VMs equivalentes (~4 vCPU / 11 GiB c/u) | **(estimado)** ~$110 c/u → ~$770/mes | Tarifa aproximada de instancia de propósito general de gama media, on-demand, sin descuento por reserva |
| IP pública / balanceador en VM1 | **(estimado)** ~$20/mes | Necesario para exponer el sistema a Internet real |
| Dominio público (.com) | **(estimado)** ~$1/mes (~$12/año) | Registro anual con un registrador estándar |
| Certificado TLS (Let's Encrypt) | $0 | Gratuito una vez existe un dominio público válido |
| Almacenamiento de respaldo adicional (redundancia geográfica) | **(estimado)** ~$15/mes | Redundancia que hoy no existe (sección 9.4) |
| **Total estimado** | **(estimado) ~$816/mes** (~$9,800/año) | Cifra ilustrativa de orden de magnitud, no una cotización — sirve para contrastar contra el costo real de $0 del proyecto académico |

### 12.4 Resumen

El proyecto, dadas sus restricciones académicas (K3, K5), opera con **costo real de $0**: las 7 VMs las asigna la universidad y todo el software usado es open source o de plan gratuito. La comparación de la sección 12.3 es puramente ilustrativa, para dimensionar la diferencia frente a una operación comercial real, y no representa una decisión de presupuesto tomada por el equipo.

---

## 13. Limitaciones reconocidas

Índice de las limitaciones ya aceptadas a lo largo de este documento y del SAD — no se repiten aquí las justificaciones completas, solo dónde encontrarlas.

| Limitación | Origen | Detalle en |
|---|---|---|
| VM3 es punto único de falla (SPOF) | K5 | SAD, sección 5.2; este documento, sección 5.6 |
| Las 7 VMs tienen la misma especificación sin importar el rol | Laboratorio de la Javeriana | Sección 3.1 |
| Sin VM dedicada a staging — funcional en CI, carga contra producción | K5 | Sección 6.2; SRS, RNF-07 |
| La prueba de carga valida después del despliegue, no antes, con reversión si falla | K5 | Sección 6.2/6.3; SRS, RNF-08 |
| `kubectl rollout undo` solo revierte la imagen del contenedor, no migraciones de esquema — exige que todo cambio de esquema sea expand-contract | Diseño de Kubernetes | Sección 5.8; sección 6.2 |
| Evidencias en MinIO sin respaldo | K5 | Sección 9.4 |
| Recuperación manual entre 12 y 24 horas ante cualquier falla | K7 | SAD, sección 5.2 |
| QoS "Guaranteed" del Matching protege contra desalojo, pero le quita capacidad de ráfaga | Trade-off de diseño | Sección 5.6 |
| Parámetros de `readinessProbe`/`livenessProbe` sin definir | Falta de datos reales medidos | Sección 5.8 |
| La sección 3.1 describe un presupuesto de recursos asignado, no uno medido — la validación real llega con el benchmarking del entregable "PoC + ADR" | K3 (alcance académico) | Sección 3.1 |
| Certificado TLS autofirmado, sin CA públicamente reconocida, por no existir dominio público | K5 / red privada del laboratorio | Sección 10.1; sección 11 |

Ninguna de estas limitaciones se considera un defecto a corregir dentro del alcance de este documento — son restricciones aceptadas conscientemente, consistentes con K3, K5 y K7 del SAD.

---

## 14. Control de versiones del documento

| Versión | Fecha | Descripción del cambio |
|---|---|---|
| 1.0 | 13 sep 2026 | Versión inicial del Documento de Infraestructura. |
| 1.1 | 15 sep 2026 | Se reorganiza el documento para alinearlo con el desglose de Jira (SCRUM-167 a SCRUM-178): se agregan las secciones "Arquitectura de despliegue", "Dominio y DNS" y "Presupuesto"; se promueve "Gestión de secretos" a sección propia; se renumeran las referencias cruzadas internas. Se corrige la sección de TLS: se reemplaza Let's Encrypt (inviable sin dominio público) por certificado autofirmado. |
| 1.2 | *(este documento)* | Corrige tres hallazgos bloqueantes de una revisión crítica independiente: (1) el `--service-cidr` por defecto de k3s coincidía con la red del laboratorio (`10.43.0.0/16`) — se fija explícitamente fuera de ese rango en `deploy-k3s.yml` (sección 5.2); (2) se documenta que `kubectl rollout undo` no revierte migraciones de esquema y se exige el patrón expand-contract para toda migración (sección 5.8), y se aclara que la prueba de carga corre contra un tenant de prueba dedicado, no contra datos reales (sección 6.2); (3) se completa la tabla de puertos con las rutas que otras secciones ya requerían pero no estaban habilitadas (scrape de `node_exporter`, envío de logs a Loki, API server de k3s para el despliegue, subida del backup a MinIO — sección 10.2). |
