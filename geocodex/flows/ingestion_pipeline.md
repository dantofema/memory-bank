# Flujo: Pipeline de ingestión y parsing (alto nivel)

Objetivo
- Definir las etapas del pipeline: upload → parsing streaming → persistencia por chunks → estado final.

Estados de un `las_file`
- PENDING: archivo subido, job no despachado o en cola
- PROCESSING: job activo
- COMPLETED: parsing satisfactorio (>= MIN_VALID_RATIO)
- PARTIAL: parsing con porcentaje de filas válidas menor que MIN_VALID_RATIO pero > 0
- FAILED: parsing abortado por error crítico

Etapas
1. Job `ParseLasFileJob` se ejecuta con parámetros (`las_file_id`, `tenant_id`, `storage_path`).
2. Reader abre el archivo en streaming y detecta secciones (~V, ~C, ~A). Se parsean headers y metadatos y se guardan en `header_meta`.
3. Cuando se llega a `~ASCII Log Data` (`~A`), el reader emite filas parseadas una a una; el job acumula un buffer.
4. Al alcanzar `chunk_size` (1000 por defecto) o al finalizar, hacer insert masivo en `las_curve_data` y reset buffer.
5. Registrar rows_inserted incrementales y métricas.
6. Al finalizar, calcular `valid_ratio = rows_inserted / rows_expected` (si rows_expected conocido) o inferir ratio por filas parseadas. Aplicar MIN_VALID_RATIO (0.90) para definir estado final.

Errores
- Error de header crítico → abortar job y marcar `FAILED` con `error_log`.
- Error en fila → registrar fila corrupta en `error_log` y continuar.

Retries / Idempotency
- Job idempotente: si se re-intenta, evitar duplicados (p.ej. limpiar filas previas si `version` cambia o usar deduplicación por unique constraint si aplica).

Métricas a exponer
- time_ms_total
- rows_inserted
- chunks_processed
- error_count

Criterios de aceptación
- Documento con pseudocódigo y ejemplos de parsing para fixtures.

