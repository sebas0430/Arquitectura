# Claude Code — QUICKPATCH

Este archivo es el punto de entrada para Claude Code dentro del repositorio QUICKPATCH.

## 1. Antes de trabajar

Lee en este orden:

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. El archivo correspondiente al rol en `.ai/roles/`
5. Únicamente los documentos formales necesarios para la tarea
6. El código directamente relacionado

No cargues automáticamente todo `docs/`.

## 2. Documentación oficial

- Requisitos: `docs/requirements/SRS.md`
- Arquitectura: `docs/architecture/SAD.md`
- Diseño: `docs/architecture/SDD.md`
- Datos y contratos: `docs/design/DD.md`
- Versión histórica del DD: `docs/archive/DD-v1.md`
- Infraestructura: `docs/infrastructure/INFRASTRUCTURE.md`
- Políticas del equipo: `docs/governance/WORKING_AGREEMENTS.md`
- Índice documental: `docs/README.md`

## 3. Precedencia documental

1. SRS define qué debe hacer el producto.
2. SAD define arquitectura y restricciones.
3. SDD define el diseño.
4. DD define datos y contratos concretos.
5. Infraestructura define despliegue y operación.
6. Working Agreements define la forma de trabajo.

No resuelvas contradicciones silenciosamente.

## 4. Contexto por rol

### Backend
`.ai/roles/backend.md`

Área principal futura: `apps/backend/**`

### Frontend
`.ai/roles/frontend.md`

Áreas principales futuras: `apps/web/**` y `apps/mobile/**`

### QA
`.ai/roles/qa.md`

Área principal futura: `tests/**`

### DevOps
`.ai/roles/devops.md`

Áreas principales futuras: `infra/**` y `.github/workflows/**`

## 5. Política de contexto

Usa contexto progresivo:

1. contexto global;
2. resumen de arquitectura;
3. rol;
4. requisito/ADR/contrato relacionado;
5. código necesario.

Evita leer documentos completos si solo necesitas una sección concreta.

## 6. Límites de modificación

### Backend
No modificar Frontend, infraestructura o E2E de QA por defecto.

### Frontend
No modificar implementación backend, migraciones o infraestructura por defecto.

### QA
No corregir silenciosamente lógica productiva.

### DevOps
No modificar reglas de negocio, endpoints o modelos de dominio por defecto.

## 7. Contratos

### REST
`docs/contracts/openapi/`

Frontend no debe inventar endpoints.
Backend no debe modificar contratos implícitamente.

### Kafka
`docs/contracts/events/`

Los cambios deben identificar productor, consumidores, compatibilidad, campos e impacto.

Consulta:

- `.ai/workflows/api-change.md`
- `.ai/workflows/event-change.md`
- `.ai/workflows/database-change.md`

cuando corresponda.

## 8. Multi-tenancy

QUICKPATCH es multi-tenant.

No debilites el aislamiento entre tenants.

Cuando el tenant provenga del contexto autenticado:

- no aceptes `tenant_id` libremente desde el cliente;
- no confíes en identificadores enviados por la UI;
- conserva filtros y controles de aislamiento;
- crea pruebas contra fugas entre tenants.

## 9. Pagos

Respeta las restricciones PCI-DSS descritas en:

- `docs/architecture/SAD.md`
- `docs/design/DD.md`

No almacenes CVV.
No almacenes PAN cuando la arquitectura establece tokenización.
No incluyas estos datos en logs.

## 10. Kafka y resiliencia

No reemplaces comunicación asíncrona aprobada por llamadas síncronas solo para simplificar implementación.

Respeta cuando apliquen:

- contratos de eventos;
- idempotencia;
- correlation IDs;
- reintentos;
- patrón Outbox;
- trazabilidad.

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

No hagas push directo a `main`.

## 12. Commits

Usa Conventional Commits.

Ejemplos:

```text
feat(auth): agregar autenticacion jwt
fix(matching): corregir filtro de disponibilidad
test(payments): validar idempotencia del webhook
docs(ai): actualizar contexto de claude
chore(devops): ajustar health check
```

## 13. Antes de implementar

Determina:

1. requisito relacionado;
2. rol responsable;
3. servicio o módulo propietario;
4. escenario de calidad;
5. ADR relevante;
6. contrato afectado;
7. datos afectados;
8. pruebas necesarias;
9. impacto sobre otros roles.

## 14. Antes de terminar

Resume:

- archivos modificados;
- motivo del cambio;
- pruebas ejecutadas;
- pruebas pendientes;
- contratos afectados;
- documentación afectada;
- riesgos o decisiones pendientes.

## 15. Reglas generales

- No inventes arquitectura.
- No inventes endpoints.
- No inventes eventos.
- No inventes tablas.
- No cambies tecnologías aprobadas sin justificarlo.
- No elimines controles de seguridad para simplificar.
- No elimines multi-tenancy.
- No introduzcas secretos.
- No modifiques áreas ajenas sin indicarlo.
- No cargues documentación irrelevante.
- Mantén trazabilidad entre SRS, SAD, SDD, DD, implementación y pruebas.

Las reglas generales compartidas también están en `AGENTS.md`.
