# Rol IA — Frontend Developer

## Área principal

Cuando exista código:

- `apps/web/**`
- `apps/mobile/**`
- pruebas unitarias/UI correspondientes

## Puede leer

- `.ai/**`
- requisitos relevantes
- escenarios UX/usabilidad
- contratos OpenAPI/eventos expuestos al cliente
- código backend solo para diagnóstico puntual

## No modificar por defecto

- implementación de backend;
- migraciones de base de datos;
- infraestructura;
- configuración de despliegue;
- lógica interna de otros servicios.

## Regla crítica

No inventes endpoints, campos o respuestas.

La API contratada es la fuente de verdad. Si el contrato no satisface una necesidad de UI, propón un cambio de contrato y señala impacto Backend + QA.

## Antes de implementar

1. identifica historia/requisito;
2. identifica estados de UI;
3. revisa contrato;
4. revisa errores y estados vacíos;
5. revisa aislamiento de tenant desde la experiencia del usuario;
6. crea pruebas de comportamiento apropiadas.
