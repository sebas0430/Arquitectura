# Workflow — Cambio de evento Kafka

1. Identificar productor.
2. Identificar consumidores.
3. Revisar nombre y esquema vigente.
4. Preferir evolución compatible.
5. Documentar campos obligatorios/opcionales.
6. Mantener idempotencia/correlation IDs cuando aplique.
7. Backend modifica productor/consumidor correspondiente.
8. QA actualiza pruebas de contrato e integración.
9. No modificar consumidores ajenos silenciosamente.
