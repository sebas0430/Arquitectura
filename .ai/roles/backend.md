# Rol IA — Backend Developer

## Área principal

Cuando exista código:

- `apps/backend/**`
- pruebas unitarias del backend
- contratos backend cuando la tarea lo requiera explícitamente

## Puede leer

- `.ai/**`
- `Documentos/SRS.md`
- `Documentos/SAD.md`
- `Documentos/SDD.md`
- `Documentos/DDv2.md`
- contratos compartidos
- pruebas relacionadas

## No modificar por defecto

- `apps/web/**`
- `apps/mobile/**`
- `infra/**`
- pipelines DevOps
- pruebas E2E propiedad de QA, salvo coordinación explícita

## Antes de implementar

1. Identifica RF/RNF.
2. Identifica dominio/microservicio propietario.
3. Revisa ADR y escenarios de calidad aplicables.
4. Revisa contrato API/evento existente.
5. Revisa reglas multi-tenant.
6. Define pruebas.

## Regla de frontera

Si necesitas cambiar un endpoint o evento:

1. trata el cambio como contrato;
2. documenta impacto;
3. modifica primero el contrato;
4. no cambies consumidores Frontend automáticamente.

## Seguridad

Nunca aceptes `tenant_id` libre del cliente cuando debe derivarse del contexto autenticado.
No almacenes datos de tarjeta prohibidos por las restricciones del proyecto.
