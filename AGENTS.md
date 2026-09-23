# Instrucciones para agentes de IA — QUICKPATCH

Este archivo es el punto de entrada general para agentes como Codex.

## Contexto obligatorio

Lee, en este orden:

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. El archivo del rol aplicable en `.ai/roles/`
5. Solo después, los documentos formales necesarios para la tarea.

No cargues todos los documentos largos del repositorio por defecto.

## Fuentes de verdad

Prioridad documental:

1. Requisitos vigentes: `Documentos/SRS.md`
2. Arquitectura vigente: `Documentos/SAD.md`
3. Diseño vigente: `Documentos/SDD.md`
4. Datos/contratos: `Documentos/DDv2.md` hasta completar la migración documental
5. Infraestructura: `Documentos/Documento de Infraestructura.md`
6. Políticas: `Documentos/Documento Politicas y Herramientas.md`

Si dos documentos se contradicen, no inventes una resolución silenciosa. Identifica la contradicción y propón el cambio mínimo.

## Límites

- No modifiques áreas fuera del rol salvo que la tarea lo exija explícitamente.
- No cambies contratos compartidos de forma implícita.
- No agregues dependencias sin justificar su necesidad.
- No elimines requisitos, ADR, controles de seguridad o aislamiento multi-tenant para simplificar una implementación.
- No hagas cambios directos a `main`.
- Usa ramas derivadas de `develop`.
- Mantén commits pequeños y coherentes.
