# Instrucciones para agentes de IA — QUICKPATCH

Este archivo es el punto de entrada general para agentes de IA como Codex y herramientas equivalentes.

## 1. Contexto obligatorio

Antes de modificar código o documentación, lee en este orden:

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. El contexto correspondiente al rol en `.ai/roles/`
5. Los documentos formales estrictamente necesarios para la tarea
6. El código relacionado directamente con la tarea

No cargues todos los documentos del repositorio por defecto.

## 2. Fuentes de verdad

### Requisitos
`docs/requirements/SRS.md`

### Arquitectura
`docs/architecture/SAD.md`

### Diseño
`docs/architecture/SDD.md`

### Datos y contratos
`docs/design/DD.md`

Versión histórica:
`docs/archive/DD-v1.md`

### Infraestructura
`docs/infrastructure/INFRASTRUCTURE.md`

### Políticas de trabajo
`docs/governance/WORKING_AGREEMENTS.md`

## 3. Precedencia documental

1. `SRS.md` define qué debe hacer el producto.
2. `SAD.md` define arquitectura y restricciones.
3. `SDD.md` define cómo se diseña la solución.
4. `DD.md` define datos y contratos concretos.
5. `INFRASTRUCTURE.md` define despliegue y operación.
6. `WORKING_AGREEMENTS.md` define cómo trabaja el equipo.

No resuelvas contradicciones silenciosamente.

## 4. Contexto por rol

- Backend: `.ai/roles/backend.md`
- Frontend: `.ai/roles/frontend.md`
- QA: `.ai/roles/qa.md`
- DevOps: `.ai/roles/devops.md`

## 5. Límites entre roles

### Backend
No modificar por defecto Frontend, infraestructura, pipelines o E2E de QA.

### Frontend
No modificar por defecto lógica backend, migraciones o infraestructura.

### QA
No modificar por defecto lógica productiva Backend/Frontend ni infraestructura productiva.

### DevOps
No modificar por defecto reglas de negocio, endpoints, entidades o comportamiento funcional del Frontend.

## 6. Contratos compartidos

### API REST
`docs/contracts/openapi/`

Frontend no debe inventar endpoints.
Backend no debe cambiar contratos silenciosamente.

### Eventos Kafka
`docs/contracts/events/`

Todo cambio debe identificar productor, consumidores, campos, compatibilidad e impacto.

## 7. Cambios compartidos

Consulta cuando aplique:

- `.ai/workflows/api-change.md`
- `.ai/workflows/event-change.md`
- `.ai/workflows/database-change.md`
- `.ai/workflows/pull-request.md`

No modifiques automáticamente todos los consumidores afectados salvo instrucción explícita.

## 8. Arquitectura crítica

QUICKPATCH es multi-tenant.

Ningún cambio puede debilitar el aislamiento entre tenants.

Cuando el diseño indique que el tenant proviene del contexto autenticado, no aceptes `tenant_id` libremente desde el cliente.

## 9. Pagos

Respeta las restricciones PCI-DSS documentadas.

No almacenar CVV.
No almacenar PAN cuando la arquitectura define tokenización.
No registrar estos datos en logs.

Consulta `docs/architecture/SAD.md` y `docs/design/DD.md` antes de modificar Payments.

## 10. Kafka y procesamiento asíncrono

No reemplaces Kafka por llamadas REST solo para simplificar una implementación.

Respeta contratos, idempotencia, correlation IDs, reintentos y Outbox cuando aplique.

## 11. GitFlow

Las ramas normales deben salir de `develop`.

Convención recomendada:

```text
feature/backend/<descripcion>
feature/frontend/<descripcion>
feature/qa/<descripcion>
feature/devops/<descripcion>
```

Los Pull Requests normales deben apuntar a `develop`.

No realizar push directo a `main`.

## 12. Commits

Utiliza Conventional Commits.

Ejemplos:

```text
feat(matching): agregar filtro por disponibilidad
fix(payments): corregir procesamiento idempotente
test(auth): agregar pruebas de aislamiento multi-tenant
docs(ai): actualizar contexto del backend
chore(devops): ajustar pipeline de integracion
```

## 13. Contexto progresivo

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. `.ai/roles/<rol>.md`
5. Solo las secciones relevantes de la documentación
6. Código directamente relacionado

No escanees todo el repositorio sin una razón concreta.

## 14. Antes de implementar

Identifica:

1. requisito relacionado;
2. rol responsable;
3. módulo o servicio propietario;
4. atributo de calidad;
5. ADR aplicable;
6. contrato existente;
7. datos afectados;
8. pruebas necesarias;
9. otros roles afectados.

## 15. Antes de finalizar

Revisa:

- archivos modificados;
- pruebas ejecutadas;
- contratos afectados;
- documentación afectada;
- riesgos pendientes;
- revisiones de otros roles necesarias.

## 16. Reglas generales

- No inventes arquitectura.
- No inventes endpoints.
- No inventes eventos.
- No inventes tablas.
- No cambies tecnologías aprobadas sin justificación.
- No elimines restricciones para simplificar código.
- No elimines validaciones multi-tenant.
- No agregues secretos al repositorio.
- No modifiques áreas no relacionadas con la tarea.
- No cargues documentación irrelevante en contexto.
- Mantén trazabilidad entre requisitos, arquitectura, implementación y pruebas.
