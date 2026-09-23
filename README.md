# QUICKPATCH

Plataforma multi-tenant de servicios técnicos para hogares y empresas.

## Estado del repositorio

Este repositorio contiene la documentación arquitectónica y, progresivamente, el código fuente de QUICKPATCH.

## Punto de entrada para personas

1. Lee `Documentos/Índice — Documentación QUICKPATCH.md`.
2. Revisa `Documentos/SRS.md` para requisitos.
3. Revisa `Documentos/SAD.md` para arquitectura y atributos de calidad.
4. Revisa `Documentos/SDD.md` para diseño.
5. Revisa `Documentos/DDv2.md` como versión vigente temporal del Documento de Diseño.
6. Revisa `Documentos/Documento de Infraestructura.md` para despliegue.
7. Revisa `Documentos/Documento Politicas y Herramientas.md` para GitFlow y políticas.

> Mientras se completa la reorganización documental, `DDv2.md` es la versión vigente y `DD.md` se considera histórica. En una segunda etapa se renombrará la versión vigente a `DD.md` y la anterior se archivará.

## Punto de entrada para asistentes de IA

- Contexto global: `.ai/PROJECT_CONTEXT.md`
- Resumen arquitectónico: `.ai/ARCHITECTURE_SUMMARY.md`
- Glosario: `.ai/GLOSSARY.md`
- Reglas por rol: `.ai/roles/`
- Flujos de cambios compartidos: `.ai/workflows/`

Claude Code debe leer `CLAUDE.md`.
Codex y agentes compatibles deben leer `AGENTS.md`.

## Regla principal

Una tarea debe modificar únicamente el área de responsabilidad solicitada. Los contratos compartidos (API, eventos y persistencia pública entre servicios) se modifican de forma explícita y revisada por los roles afectados.
