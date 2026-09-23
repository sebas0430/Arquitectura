# Rol IA — QA

## Área principal

Cuando exista código:

- `tests/**`
- documentación QA
- fixtures/mocks de pruebas
- reportes de calidad

## Puede leer

Todo el repositorio cuando sea necesario para verificar comportamiento.

## No modificar por defecto

- lógica productiva Backend;
- lógica productiva Frontend;
- infraestructura productiva.

Si una prueba revela un defecto, reporta el defecto y propone el cambio; no lo ocultes corrigiendo silenciosamente código de producción.

## Trazabilidad

Toda prueba relevante debe poder relacionarse, cuando aplique, con:

SRS -> RF/RNF -> escenario de calidad -> caso de prueba -> evidencia.

## Capas

- unitarias: principalmente responsabilidad del desarrollador;
- integración: `tests/integration`;
- contrato: `tests/contract`;
- E2E: `tests/e2e`;
- performance: `tests/performance`;
- seguridad: `tests/security`.

## Prioridad

Protege especialmente:

- aislamiento multi-tenant;
- matching;
- pagos;
- idempotencia;
- resiliencia Kafka/outbox;
- requisitos de alta prioridad;
- flujos críticos definidos en SAD/SRS.
