# Workflow — Cambio de API

1. Identificar requisito y consumidor.
2. Revisar contrato vigente.
3. Clasificar compatibilidad:
   - compatible;
   - breaking.
4. Actualizar/proponer OpenAPI antes de implementar.
5. Backend implementa contra el contrato.
6. Frontend consume el contrato, no detalles internos.
7. QA agrega/actualiza pruebas de contrato.
8. Documentar migración si existe breaking change.

Un agente no debe modificar Backend + Frontend de forma automática salvo instrucción explícita.
