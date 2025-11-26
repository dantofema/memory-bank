# Database conventions — LasParser & Core
Nota: Este documento es normativo — cualquier cambio en estas convenciones debe registrarse en `architecture/decisions.md` como ADR antes de implementarlo.

- Considerar constraints y unique keys según disponibilidad de identificadores (por ejemplo unique on las_file_id+depth si aplica).
- Registrar `error_log` en `las_files` con estructura { row: <int|null>, message: <string>, severity: <string> }
Auditing and integrity

- S3 lifecycle: recomendada para producción (configurable mediante infra).
- Política por defecto: 365 días; exponer variable/flag por tenant para modificaciones.
Data retention

- Consider COPY/pg_copy como mejora para producción en el futuro.
- Inserts masivos: usar chunked inserts (configurable `chunk_size`, default 1000). Diseñar migraciones y pruebas teniendo en cuenta índices: crear índices después de cargas masivas cuando sea posible.
Performance notes

  - created_at, updated_at
  - extras (jsonb)
  - npor (double)
  - rhob (double)
  - dt (double)
  - gr (double)
  - depth (numeric/decimal)
  - user_id (unsignedBigInteger)
  - tenant_id (unsignedBigInteger)
  - las_file_id (unsignedBigInteger) — FK -> las_files.id
  - id (bigint PK)
- `las_curve_data`:

  - created_at, updated_at
  - rows_inserted (bigint, default 0)
  - rows_expected (bigint, nullable)
  - error_log (jsonb)
  - header_meta (jsonb)
  - version (integer)
  - status (string) — enum-like: PENDING/PROCESSING/COMPLETED/PARTIAL/FAILED
  - size_bytes (bigint)
  - storage_path (string)
  - filename (string)
  - user_id (unsignedBigInteger) — FK
  - tenant_id (unsignedBigInteger) — FK
  - id (bigint PK)
- `las_files`:
Schemas sugeridos (resumen)

- Soft deletes: usar `deleted_at` solo si hay requerimiento explícito (no default en MVP)
- Timestamps: `created_at`, `updated_at`
- Foreign keys: `<resource>_id` (unsignedBigInteger)
- Primary key: `id` (big increment)
Naming conventions

- Indexes: definir índices compuestos en las consultas críticas (p.ej. (`las_file_id`, `depth`)).
- FK y ON DELETE: usar `ON DELETE CASCADE` para datos dependientes (`las_curve_data` -> `las_files`).
- Campos obligatorios en recursos multi-tenant: `tenant_id` (unsignedBigInteger) y `user_id` (unsignedBigInteger). Estos deben existir en todas las tablas de recursos del módulo.
- Todos los nombres de tablas en el módulo seguirán `snake_case` y prefijo funcional cuando aplique (ej.: `las_files`, `las_curve_data`).
Principios generales

Propósito: documentar convenciones y decisiones de diseño para la base de datos que deben permanecer estables y guiar las migraciones del módulo LasParser.


