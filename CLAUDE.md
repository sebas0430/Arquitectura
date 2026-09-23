# Claude Code — QUICKPATCH

Usa este archivo únicamente como punto de entrada.

## Antes de trabajar

Lee:

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. `.ai/roles/<rol>.md` según la tarea

Después abre únicamente los documentos formales y el código estrictamente necesarios.

## Política de contexto

No cargues automáticamente todo `Documentos/`.

El objetivo es mantener una ventana de contexto pequeña y relevante.

## Política de cambios

- Respeta los límites del rol.
- Un cambio de API, evento o esquema compartido debe tratarse como cambio de contrato.
- No implementes cambios en Frontend y Backend a la vez salvo que el usuario lo solicite explícitamente.
- No modifiques infraestructura desde tareas de producto.
- No modifiques lógica de producto desde tareas DevOps.
- Antes de terminar, enumera archivos modificados, pruebas ejecutadas y riesgos pendientes.

Las reglas globales completas están en `AGENTS.md`.
