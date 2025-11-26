# Plan de planificación — Módulo LasParser (sin código)

Este documento contiene el plan, tareas, decisiones y artefactos que deben existir antes de comenzar la implementación. No incluye código: todos los cambios de implementación se harán en una rama y dentro del módulo del proyecto, pero el repo principal en esta etapa debe contener únicamente la planificación.

Resumen ejecutivo
- Objetivo: Crear un plan completo y verificable para el desarrollo del módulo `LasParser` que implemente ingestión streaming de archivos LAS 2.0, persistencia por chunks y aislamiento multi-tenant. Se trabajará con `local` storage en `local` y S3 en `staging/production`.
- Restricción: NO crear código en la rama principal durante la fase de planificación. Todos los artefactos técnicos deben describirse aquí y organizarse como tickets.

Decisiones confirmadas (fijas)
- Tenancy: obligatorio (`tenant_id` y `user_id` en todos los recursos). Usar package de tenancy en implementación (p.ej. `spatie/laravel-multitenancy`).
- Storage: `local` en `local`, S3 en `staging` y `production`.
- Queue: `database` en `local`, `redis` en `staging` y `production`.
- Curvas "whitelist" mínimas: `DEPT`, `GR`, `DT`, `RHOB`, `NPOR`.
- chunk_size: 1000 (configurable).
- MIN_VALID_RATIO: 0.90 (90%) por defecto — si menos, marcar `PARTIAL` o `FAILED` según umbral; esto debe revisarse en revisión de producto.
- Retención por defecto: 365 días.

Artefactos que debe documentar este plan
- Esquema de BD (tablas + columnas + índices) para `las_files` y `las_curve_data`.
- Contratos API/eventos: estructura de la fila `las_files` y eventos de job (PENDING→PROCESSING→COMPLETED|PARTIAL|FAILED).
- Estructura de módulos y paths (donde se colocará cada artefacto una vez que comencemos a implementar):
  - `Modules/LasParser/database/migrations`
  - `Modules/LasParser/Models`
  - `Modules/LasParser/Jobs`
  - `Modules/LasParser/Services`
  - `Modules/LasParser/tests/fixtures`
  - `Modules/LasParser/docs` (README y runbooks)

Backlog y tickets mínimos (para crear en GitHub)

Sprint 1 — Cimientos (2-3 días)
- T1: Documento de migraciones y esquema final (este ticket entrega SQL/migration pseudo-code y diagramas). Criterio: aprobada por arquitecto y lista para codificar.
- T2: Documento de modelos y políticas de acceso (detallar GlobalScopes por tenant y excepciones de admin). Criterio: políticas definidas y ejemplos de uso.
- T3: Fixtures listadas y ejemplos (ej: `valid_2.0.las`, `broken_header.las`, `file_with_extra_curves.las`). Criterio: fixtures versionadas y aprobadas.

Sprint 2 — Ingesta & Colas (3-5 días)
- T4: Flujo de subida (Filament) definido: validaciones, almacenamiento, creación de `las_files` y dispatch del job. Criterio: diagramas y request/response JSONs.
- T5: Contrato del job `ParseLasFileJob`: inputs, outputs, retries, idempotency, timeouts, memory limits. Criterio: documento con ejemplos.

Sprint 3 — Parser (4-6 días)
- T6: Especificación de `LasReader` streaming: detección de secciones (~V, ~C, ~A), tokenización, manejo de nulos. Criterio: pseudocódigo y ejemplos de parseo para fixtures.
- T7: Contrato de persistencia: buffer, chunked insert (1000), error handling por fila y `MIN_VALID_RATIO`. Criterio: pseudocódigo y flujos de error.

Sprint 4 — Tests & Hardening (3-5 días)
- T8: Matriz de tests (Unit + Feature + Integration multi-tenant) y tablas de fixtures para cada caso. Criterio: matriz aprobada y lista para codificar.
- T9: Runbook de despliegue y variables de entorno para `local/staging/production` (S3, redis, forge). Criterio: runbook revisado por ops.

Items transversales (no-blocking pero necesarios)
- Observabilidad: definir eventos/métricas que instrumentar en jobs (rows_inserted, time_ms, error_count).
- Security: definir policies + auditoría (logs de re-procesos, descargas, eliminaciones).
- Cleanup: definir `lasparser:cleanup` y política de lifecycle en S3.

Criterios de aceptación generales (Done)
- Cada ticket tiene: descripción, DoD (tests/fixtures), owner, estimación en días/hours, y dependencias.
- No hay código en la rama principal: si algún PR añade código, debe referenciar el ticket y pasar por revisión.

Checklist de revisión de planificación (por arquitecto/producto)
- [ ] ¿MIN_VALID_RATIO=0.90 aceptado? (si no, especificar nuevo valor).
- [ ] ¿chunk_size=1000 aceptado?
- [ ] ¿Retención=365 días aceptada?
- [ ] ¿Paths y módulos confirmados para implementación cuando se inicie? (Modules/LasParser/...)
- [ ] Fixtures mínimas aprobadas: `valid_2.0.las`, `broken_header.las`, `file_with_extra_curves.las`.

Próximos pasos inmediatos
1. Crear los tickets en GitHub con el contenido de este plan y asignarlos al desarrollador.
2. Revisar la checklist anterior y marcar las decisiones finales.
3. Sólo tras aprobación: crear la branch `feat/lasparser-mvp` e implementar los tickets en orden.

Document history
- 2025-11-26: Plan inicial generado por equipo de planificación.


