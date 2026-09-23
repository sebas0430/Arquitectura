# Rol IA — Backend Developer

## Área principal

Cuando exista código:

- `apps/backend/**`
- pruebas unitarias asociadas al backend
- contratos backend cuando la tarea lo requiera explícitamente

## Contexto inicial

Antes de trabajar, lee:

1. `.ai/PROJECT_CONTEXT.md`
2. `.ai/ARCHITECTURE_SUMMARY.md`
3. `.ai/GLOSSARY.md`
4. este archivo

Después consulta únicamente la documentación formal necesaria.

## Puede leer

- `.ai/**`
- `docs/requirements/SRS.md`
- `docs/architecture/SAD.md`
- `docs/architecture/SDD.md`
- `docs/design/DD.md`
- `docs/contracts/**`
- pruebas relacionadas con el servicio o funcionalidad
- `docs/infrastructure/INFRASTRUCTURE.md` cuando la tarea tenga impacto operativo

## No modificar por defecto

- `apps/web/**`
- `apps/mobile/**`
- `infra/**`
- `.github/workflows/**`
- pruebas E2E propiedad de QA, salvo coordinación explícita

## Antes de implementar

1. Identifica el RF/RNF relacionado.
2. Identifica el dominio o microservicio propietario.
3. Revisa ADR y escenarios de calidad aplicables.
4. Revisa el contrato API o evento existente.
5. Revisa las reglas multi-tenant.
6. Identifica datos afectados.
7. Define las pruebas necesarias.
8. Identifica consumidores potencialmente afectados.

## Regla de frontera

Si necesitas cambiar un endpoint o evento:

1. trata el cambio como un contrato compartido;
2. documenta su impacto;
3. modifica o propone primero el contrato correspondiente;
4. no cambies consumidores Frontend automáticamente;
5. informa si QA debe actualizar pruebas de contrato.

Consulta:

- `.ai/workflows/api-change.md`
- `.ai/workflows/event-change.md`

## Persistencia

Antes de modificar tablas, entidades persistentes o migraciones:

- verifica el servicio propietario del dato;
- conserva aislamiento multi-tenant;
- evita compartir tablas entre servicios sin una decisión arquitectónica explícita;
- evalúa compatibilidad con datos existentes;
- actualiza `docs/design/DD.md` si el cambio modifica el diseño vigente.

Consulta:

`.ai/workflows/database-change.md`

## Seguridad

Nunca aceptes `tenant_id` libre del cliente cuando deba derivarse del contexto autenticado.

No elimines filtros o controles de aislamiento entre tenants.

No almacenes información de tarjeta prohibida por las restricciones del proyecto.

En particular:

- no almacenar CVV;
- no almacenar PAN cuando la arquitectura define tokenización;
- no incluir estos datos en logs.

Para cambios relacionados con pagos consulta:

- `docs/architecture/SAD.md`
- `docs/design/DD.md`

## Kafka y resiliencia

Cuando el flujo esté definido como asíncrono:

- respeta el contrato de evento;
- conserva idempotencia cuando aplique;
- utiliza correlation IDs cuando estén definidos;
- respeta reintentos;
- respeta Outbox cuando corresponda;
- no reemplaces Kafka por REST únicamente para simplificar código.

## Pruebas

Como mínimo considera:

- unitarias;
- integración;
- contratos;
- aislamiento multi-tenant;
- errores y casos borde;
- idempotencia cuando aplique.

Las pruebas E2E globales pertenecen principalmente a QA.

## Antes de finalizar

Indica:

- archivos modificados;
- requisito implementado;
- contrato afectado;
- pruebas ejecutadas;
- pruebas pendientes;
- otros roles afectados;
- riesgos o decisiones pendientes.
