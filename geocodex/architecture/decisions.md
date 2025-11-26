# DECISIONS — LasParser

**Formato ADR aplicado:** cada decisión debe seguir el esquema:

## [YYYY-MM-DD] Título corto

* Problema:
* Opciones evaluadas:
* Decisión:
* Justificación:
* Estado: (Propuesto | Activo | Deprecado)
* Autor: <nombre>

---

Autor: Equipo LasParser
Fecha creada: 2025-11-25
Propósito: Registrar decisiones clave priorizadas (P0/P1/P2) para el desarrollo del módulo `LasParser`.

Resumen rápido

- Este documento lista las decisiones críticas (P0), importantes (P1) y de menor prioridad (P2).
- Para cada decisión se incluye: pregunta, opciones, recomendación (sugerida), pros/contras, campo para selección,
  razón, dueño y fecha.

Índice rápido

- P0 — Decisiones críticas
- P1 — Decisiones de alta prioridad
- P2 — Decisiones de prioridad media
- Siguiente pasos tras responder P0
- Historia de cambios

---

## P0 - Decisiones críticas (Responder primero)

Cada ítem P0 debe responderse antes de iniciar la implementación para evitar retrabajo.

### P0-01: Objetivo de negocio principal

- Pregunta: ¿Cuál es el objetivo de negocio principal del módulo LAS Parser?
- Opciones:
    - Ingesta masiva para análisis batch
    - Visualización interactiva simple (Filament)
    - Análisis avanzado (procesamientos derivados, ML)
- Recomendación (sugerida): Ingesta masiva + visualización básica (priorizar ingesta fiable).
- Pros/Contras:
    - Ingesta masiva: (Pros) escalable para grandes volúmenes; (Contras) requiere optimizaciones de persistencia.
    - Visualización: (Pros) rápido valor al usuario; (Contras) puede ser costoso si se requiere downsampling en tiempo
      real.
- Selección: Ingesta masiva + visualización básica (priorizar ingesta fiable).

### P0-02: SLA / Latencia objetivo por archivo

- Pregunta: ¿Cuál es el SLA esperado para procesar un archivo desde que se sube hasta que queda listo?
- Opciones: <5 min, <30 min, <1h, sin SLA (best-effort)
- Recomendación (sugerida): <30 min para archivos de tamaño razonable (<=50MB).
- Pros/Contras:
    - SLA bajo (<5 min): precisa infra adicional (paralelismo, drivers rápidos).
    - <30 min: buen balance entre coste y experiencia.
- Selección: <30 min para archivos de tamaño razonable (<=50MB).

### P0-03: Volumen esperado (files/day y tamaño medio)

- Pregunta: ¿Qué volumen de archivos y tamaño medio esperamos por usuario / por sistema?
- Opciones: (ejemplos) 0-10/day, 10-100/day, 100+/day — Tamaño medio: <5MB, 5-50MB, >50MB
- Recomendación (sugerida): Definir rango real por producto; si desconocido elegir 5-50MB y 10-100/day como punto de
  partida.
- Selección: Definir rango real por producto; si desconocido elegir 5-50MB y 10-100/day como punto de
  partida.

### P0-04: Versiones LAS a soportar

- Pregunta: ¿Soportamos solo LAS 2.0 o también LAS 1.x / variantes?
- Opciones: 2.0 únicamente; 1.x y 2.0; soporte gradual (2.0 ahora, 1.x después)
- Recomendación (sugerida): Empezar con 2.0 y diseñar el parser con extensibilidad para 1.x.
- Selección: Empezar con 2.0 y diseñar el parser con extensibilidad para 1.x.

### P0-05: Soporte para archivos comprimidos y uploads multipart

- Pregunta: ¿Aceptamos archivos comprimidos (.gz) y uploads multipart/resumables?
- Opciones: No; Sí (.gz solo); Sí (.gz y multipart/resumable)
- Recomendación (sugerida): Aceptar .gz (descompresión en streaming). Multipart/resumable opcional según UX.
- Selección: Empezar con NO y diseñar para extender a .gz y luego a multipart/resumable.

### P0-06: Almacenamiento de archivos originales

- Pregunta: ¿Dónde se guardan los originales de .las?
- Opciones: Disk local (`local`), S3 (u otro object storage), MinIO (S3 compatible auto-hosted)
- Recomendación (sugerida): S3 (privado) en producción; `local` para desarrollo.
- Pros/Contras: S3 — alta durabilidad y escalabilidad; local — simple pero no escalable.
- Selección: S3 (privado) en producción; `local` para desarrollo.

### P0-07: Driver de colas en producción

- Pregunta: ¿Qué driver de colas utilizaremos en producción?
- Opciones: `database`, `redis`, `sqs` (u otro)
- Recomendación (sugerida): `redis` para rendimiento; `database` aceptable para MVP.
- Selección: redis

### P0-08: Lista blanca de curvas obligatorias / mínimas

- Pregunta: ¿Qué curvas se consideran obligatorias y deben mapearse a columnas en BD?
- Opciones (ejemplos): `GR`, `DT`, `RHOB`, `NPOR`, `DEPT`(Depth) — lista editable
- Recomendación (sugerida): Incluir `DEPT`, `GR`, `DT`, `RHOB`, `NPOR` como conjunto mínimo.
- Selección: Incluir `DEPT`, `GR`, `DT`, `RHOB`, `NPOR` como conjunto mínimo.

---

## P1 - Decisiones de alta prioridad (Resolver tras P0)

Estas decisiones dependen de las P0 y afectan la arquitectura y robustez del parser.

### P1-01: Tolerancia a errores en parsing

- Pregunta: ¿Fail-fast (falla todo) o best-effort (salvar lo procesable) cuando hay errores en el archivo?
- Opciones: Fail-fast; Best-effort con registro de errores por fila; Híbrido (error crítico aborta, errores menores se
  ignoran)
- Recomendación (sugerida): Híbrido — abortar en errores críticos, salvar registros válidos y almacenar `error_log`.
- Selección:  Híbrido — abortar en errores críticos, salvar registros válidos y almacenar `error_log`.
- Razonamiento: "Híbrido: abortar en errores críticos de header/estructura (archivo inválido) y, para errores por fila,
  salvar las filas válidas y registrar las filas corruptas en `las_files.error_log`. Esto reduce pérdida de datos reales
  y protege el pipeline. Definir umbral configurable `MIN_VALID_RATIO` para decidir `completed` / `partial` / `failed`."

### P1-02: Estrategia de inserción masiva

- Pregunta: ¿Usar inserts por chunks con `DB::table()->insert()` o usar COPY/pg_copy para Postgres?
- Opciones: Chunked inserts (1000 filas); COPY (Postgres) con archivo temporal; Mezcla (COPY para producción, inserts
  para dev/tests)
- Recomendación (sugerida): Soporte para COPY en producción, fallback a chunked inserts.
- Selección: Chunked inserts (1000 filas)
- Razonamiento: "Chunked inserts (chunk_size = 1000) como MVP porque es simple, portable y rápido de implementar por un
  único desarrollador; evita complejidad operativa de COPY. Diseñar la arquitectura para poder incorporar COPY/pg_copy
  más adelante si las pruebas de carga lo requieren (archivo CSV temporal en S3 o tmpfs)."

### P1-03: Manejo de columnas desconocidas

- Pregunta: ¿Ignorar columnas desconocidas, guardarlas en JSON (`header_meta`) o añadir columnas dinámicas?
- Opciones: Ignorar; Guardar en JSON blob; Crear esquema extendido (migraciones dinámicas)
- Recomendación (sugerida): Guardar en `header_meta`/`curves_meta` como JSONb para flexibilidad.
- Selección: Guardar en `header_meta`/`curves_meta` como JSONb para flexibilidad.
- Razonamiento: "Guardar curvas no reconocidas en `header_meta`/`curves_meta` y almacenar por-fila las curvas extras en
  `extras JSONB` en `las_curve_data`. Mantiene esquema minimalista con columnas físicas solo para la lista blanca y
  preserva todos los datos para consultas/transformaciones futuras sin migraciones dinámicas."

### P1-04: Indexación / particionamiento en `las_curve_data`

- Pregunta: ¿Necesitamos particionar la tabla por `las_file_id` o por rangos de `depth`? ¿Índices compuestos?
- Opciones: No particionamiento; Particionamiento por `las_file_id`; Particionamiento por rango de fechas/profundidad;
  Índices compuestos
- Recomendación (sugerida): Empezar sin particionamiento, añadir índices sobre (`las_file_id`, `depth`) y evaluar tras
  pruebas.
- Selección: Empezar sin particionamiento, añadir índices sobre (`las_file_id`, `depth`) y evaluar tras
  pruebas.
- Razonamiento: "No particionamiento en el MVP; añadir índices compuestos `(las_file_id, depth)` y `las_file_id` para
  acelerar queries comunes. El particionamiento añade overhead operativo; solo introducirlo si las pruebas de carga
  muestran problemas de escala."

### P1-05: Modelo de tenancy (aislamiento)

- Pregunta: ¿`user_id` es suficiente o necesitamos `tenant_id` (multi-tenant con múltiples usuarios por tenant)?
- Opciones: `user_id` simple; `tenant_id` + `user_id`; scopes globales multi-tenant
- Recomendación (sugerida): `user_id` si es SaaS por usuario; si existen organizaciones, añadir `tenant_id`.
- Selección: scopes globales multi-tenant
- Razonamiento: "Scopes globales multi-tenant: implementar desde el inicio filtros/global scopes que permitan soportar
  niveles user + tenant, manteniendo flexibilidad. Campos `user_id` y opcional `tenant_id` en el modelo permiten crecer
  sin migraciones disruptivas."

### P1-06: Retención y versioning de archivos

- Pregunta: ¿Política de retención y versioning para archivos subidos?
- Opciones: Mantener para siempre; Retención configurable (30/90/365 días); Versioning (guardar cada re-procesado)
- Recomendación (sugerida): Retención configurable por cliente; versionado opcional (marcar re-procesos con nueva fila
  en `las_files`).
- Selección: Retención configurable por cliente; versionado opcional (marcar re-procesos con nueva fila
  en `las_files`).
- Razonamiento: "Retención configurable por cliente (por defecto 365 días) y versioning simple: cada reprocesado crea
  una nueva fila en `las_files` con `parent_id` o incrementando `version`. Mantener originales en S3 con lifecycle
  policy opcional y exponer comando artisan `lasparser:cleanup` para aplicar la retención."

---

## P2 - Decisiones de prioridad media (Operaciones, UX, observabilidad)

### P2-01: Downsampling para visualización de curvas

- Pregunta: ¿Implementar downsampling automático para gráficas grandes? ¿Qué algoritmo?
- Opciones: No; Sí (Largest-Triangle-Three-Buckets, decimation por factor, agregación por buckets)
- Recomendación (sugerida): Sí, usar LTTB o agregación por buckets para mantener forma de la curva.
- Selección: No. Por el momento no se hara en esta etapa
- Razonamiento: "Selección actual: No (no se implementa en esta etapa). Priorizar ingestión y parsing. Documentar el
  endpoint esperado (`downsample_points`) para facilitar adición futura (bucket aggregation o LTTB)."

### P2-02: Logging y observabilidad

- Pregunta: ¿Qué stack de observabilidad usar? (Telescope, Sentry, métricas Prometheus/StatsD)
- Opciones: Solo logs + DB; Telescope + Sentry; Añadir métricas (Prometheus)
- Recomendación (sugerida): Sentry para errores, métricas clave (jobs processed, rows inserted) y logs básicos;
  Telescope en staging.
- Selección: Sentry para errores, métricas clave (jobs processed, rows inserted) y logs básicos;
  Telescope en staging.
- Razonamiento: "Sentry para errores críticos + logs estructurados locales + métricas básicas (jobs_processed,
  rows_inserted, avg_time_per_file). Telescope habilitado en staging. Esto aporta detección de errores y métricas
  mínimas sin complejidad excesiva."

### P2-03: Cobertura mínima de tests y fixtures

- Pregunta: ¿Nivel mínimo de tests para aceptar una PR del módulo?
- Opciones: Solo tests unitarios; Unit + feature básicos; Unit + feature + tests de carga (fixtures)
- Recomendación (sugerida): Unit + Feature para flujos críticos (upload + job dispatch + parsing) y fixtures para casos
  comunes/broken.
- Selección: Unit + Feature para flujos críticos (upload + job dispatch + parsing) y fixtures para casos
  comunes/broken.
- Razonamiento: "Cobertura mínima: Unit + Feature (PEST) para flujos críticos. Fixtures mínimos: `valid_2.0.las`,
  `broken_header.las`, `file_with_extra_curves.las`, `file_with_bad_rows.las`. Tests de carga ligeros (~5MB) para
  validar timeouts y uso de memoria."

### P2-04: Política de re-procesado y permisos

- Pregunta: ¿Quién puede re-disparar el parseo / eliminar / descargar archivos?
- Opciones: Solo dueño del `LasFile`; Reglas de equipo/roles; Admins globales
- Recomendación (sugerida): Solo dueño + roles con permiso explícito (admin/support)
- Selección: Solo dueño + roles con permiso explícito (admin/support)
- Razonamiento: "Solo owner + roles con permiso explícito (admin/support) pueden re-disparar, eliminar o descargar.
  Implementar en `LasFilePolicy` y añadir logs de auditoría (user_id, action, timestamp)."

---
