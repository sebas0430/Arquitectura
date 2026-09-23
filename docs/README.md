# Índice — Documentación QUICKPATCH

Plataforma Digital Multi-tenant de Servicios Técnicos para el Hogar y las Empresas — Arquitectura de Software.

Este archivo es el punto de entrada a la documentación formal del proyecto.

## Documentos vigentes

| Documento | Propósito |
|---|---|
| `SRS.md` | Requisitos funcionales, no funcionales, interfaces y alcance. |
| `SAD.md` | Arquitectura, drivers, killers, atributos de calidad, escenarios y ADR. |
| `SDD.md` | Diseño por vistas y relación entre componentes, procesos y despliegue. |
| `design/DD.md` | Documento de Diseño vigente temporal: datos y contratos. |
| `Documento de Infraestructura.md` | Topología, ambientes, despliegue, observabilidad y operación. |
| `Documento Politicas y Herramientas.md` | GitFlow, colaboración, herramientas, calidad y uso de IA. |

## Documento histórico

`archive/DD-v1.md` corresponde a la versión histórica anterior del Documento de Diseño. La versión vigente y única fuente de verdad para nuevas decisiones es `design/DD.md`.

## Precedencia

Cuando exista una aparente contradicción:

1. El SRS define **qué debe hacer** el producto.
2. El SAD define **qué arquitectura y restricciones** gobiernan la solución.
3. El SDD define **cómo se diseña** la solución respetando SRS y SAD.
4. El DD define **cómo se representan datos y contratos concretos**.
5. Infraestructura define **cómo se despliega y opera**.
6. Políticas define **cómo colabora el equipo**.

Una contradicción real no se resuelve silenciosamente: debe corregirse en la fuente correspondiente y mantener trazabilidad.

## Contexto para asistentes de IA

Los agentes no deben cargar toda esta carpeta por defecto.

Punto de entrada:

- `../AGENTS.md`
- `../CLAUDE.md`
- `../.ai/PROJECT_CONTEXT.md`
