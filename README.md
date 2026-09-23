# QUICKPATCH

Plataforma digital multi-tenant de servicios técnicos para hogares y empresas.

QUICKPATCH conecta usuarios que requieren servicios técnicos con técnicos disponibles y previamente registrados, gestionando el ciclo completo de la solicitud, asignación, ejecución, evidencia, pago, calificación y trazabilidad.

## Estructura del repositorio

El repositorio está organizado para permitir desarrollo colaborativo entre Backend Developer, Frontend Developer, QA y DevOps, y para facilitar el uso controlado de asistentes de IA como Codex y Claude Code.

```text
Arquitectura/
├── README.md
├── AGENTS.md
├── CLAUDE.md
├── .gitignore
├── .ai/
│   ├── PROJECT_CONTEXT.md
│   ├── ARCHITECTURE_SUMMARY.md
│   ├── GLOSSARY.md
│   ├── roles/
│   │   ├── backend.md
│   │   ├── frontend.md
│   │   ├── qa.md
│   │   └── devops.md
│   └── workflows/
│       ├── api-change.md
│       ├── database-change.md
│       ├── event-change.md
│       └── pull-request.md
├── docs/
│   ├── README.md
│   ├── requirements/
│   │   └── SRS.md
│   ├── architecture/
│   │   ├── SAD.md
│   │   ├── SDD.md
│   │   └── diagrams/
│   ├── design/
│   │   └── DD.md
│   ├── infrastructure/
│   │   └── INFRASTRUCTURE.md
│   ├── governance/
│   │   └── WORKING_AGREEMENTS.md
│   ├── contracts/
│   │   ├── openapi/
│   │   └── events/
│   └── archive/
│       └── DD-v1.md
└── .github/
    ├── pull_request_template.md
    └── CODEOWNERS.example
```

Las carpetas de código fuente se agregarán progresivamente:

```text
apps/
├── backend/
├── web/
└── mobile/

tests/
infra/
```

## Documentación

Toda la documentación formal se encuentra en `docs/`.

El punto de entrada es `docs/README.md`.

### Requisitos

`docs/requirements/SRS.md`

### Arquitectura

`docs/architecture/SAD.md`

### Diseño

`docs/architecture/SDD.md`

Los diagramas asociados se encuentran en `docs/architecture/diagrams/`.

### Datos y contratos

`docs/design/DD.md`

La versión histórica anterior se conserva en `docs/archive/DD-v1.md`.

### Infraestructura

`docs/infrastructure/INFRASTRUCTURE.md`

### Políticas de trabajo

`docs/governance/WORKING_AGREEMENTS.md`

## Contratos compartidos

### API REST

`docs/contracts/openapi/`

Aquí se almacenarán las especificaciones OpenAPI que sirven como frontera entre Backend y Frontend.

### Eventos Kafka

`docs/contracts/events/`

Aquí se almacenarán los esquemas de eventos utilizados entre servicios.

## Desarrollo asistido por IA

### Codex y agentes compatibles

Punto de entrada: `AGENTS.md`

### Claude Code

Punto de entrada: `CLAUDE.md`

### Contexto compartido

```text
.ai/PROJECT_CONTEXT.md
.ai/ARCHITECTURE_SUMMARY.md
.ai/GLOSSARY.md
```

### Contexto por rol

```text
.ai/roles/backend.md
.ai/roles/frontend.md
.ai/roles/qa.md
.ai/roles/devops.md
```

## Distribución de responsabilidades

### Backend Developer

Área principal futura: `apps/backend/`

### Frontend Developer

Áreas principales futuras: `apps/web/` y `apps/mobile/`

### QA

Área principal futura: `tests/`

### DevOps

Áreas principales futuras: `infra/` y `.github/workflows/`

## GitFlow

QUICKPATCH utiliza GitFlow.

Ramas principales:

```text
main
develop
```

Las nuevas funcionalidades deben salir de `develop`.

Ejemplos:

```text
feature/backend/login-jwt
feature/backend/matching-disponibilidad
feature/frontend/service-request
feature/frontend/payment-screen
feature/qa/e2e-service-flow
feature/devops/backend-ci
```

Los Pull Requests normales deben dirigirse a `develop`, no directamente a `main`.

## Commits

Se utiliza Conventional Commits.

Ejemplos:

```text
feat(matching): agregar filtro por disponibilidad
fix(payments): corregir webhook duplicado
test(auth): validar aislamiento entre tenants
docs(architecture): actualizar diagrama de componentes
chore(devops): actualizar pipeline
```

## Principios para trabajo paralelo

1. Cada rol modifica principalmente su propio directorio.
2. Los contratos compartidos se modifican explícitamente.
3. Frontend no inventa APIs.
4. Backend no modifica Frontend automáticamente.
5. DevOps no modifica lógica de negocio.
6. QA no corrige silenciosamente código productivo.
7. Cualquier cambio transversal debe declararse en el Pull Request.

## Fuente de verdad

La documentación formal tiene la siguiente precedencia:

```text
SRS
 ↓
SAD
 ↓
SDD
 ↓
DD
 ↓
Infraestructura
 ↓
Políticas de trabajo
```

- SRS define qué debe hacer el sistema.
- SAD define qué arquitectura debe respetarse.
- SDD define cómo se diseña la solución.
- DD define cómo se representan datos y contratos.
- Infraestructura define cómo se ejecuta y despliega.
- Las políticas definen cómo trabaja el equipo.

Las contradicciones entre documentos deben corregirse explícitamente; no deben resolverse silenciosamente durante la implementación.

## Proyecto

**QUICKPATCH**

Curso: Arquitectura de Software.

Proyecto académico orientado al diseño e implementación de una plataforma distribuida de servicios técnicos siguiendo principios de arquitectura de software, atributos de calidad, trazabilidad de requisitos y desarrollo colaborativo asistido por IA.
