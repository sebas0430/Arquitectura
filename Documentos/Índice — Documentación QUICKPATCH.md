

Plataforma Digital Multi-tenant de Servicios Técnicos para el Hogar y las Empresas — Arquitectura de Software

Este es el punto de entrada a toda la documentación formal y operativa del proyecto. Cada entregable vive en su propia nota y se referencia desde aquí.

---

## Entregables de rúbrica

|Documento|Contenido|
|---|---|
|[[SAD]]|Software Architecture Document — arquitectura general, escenarios de calidad, ADRs, diagramas C4.|
|[[SRS]]|Software Requirements Specification — requisitos funcionales y no funcionales, features, trazabilidad a Jira.|
|[[DD]]|Documento de Diseño — diccionario de datos, modelo ER, contratos REST y eventos Kafka.|
|[[Documento Politicas y Herramientas]]|GitFlow, políticas de equipo, versionamiento, backlog, estilo de código, uso de IA, documentación técnica.|

---

## Cómo se relacionan

- **[[SAD]]** define la arquitectura (módulos, ADRs) que **[[DD]]** traduce a modelo de datos y contratos concretos.
- **[[SRS]]** define los requisitos (RF/RNF) que tanto **[[SAD]]** como **[[DD]]** deben satisfacer — el DD referencia explícitamente los códigos RF-XX de cada tabla.
- **[[Documento Politicas y Herramientas]]** define cómo trabaja el equipo (GitFlow, commits, CI) para construir lo que describen los tres documentos anteriores.

---

_Última actualización: 3 de septiembre de 2026._