# Issues ready to create (copy to GitHub)

A continuación se listan los tickets por sprint (deliverable + subtasks). Copiar cada bloque como issue en GitHub y asignar owner/estimation.

## Sprint 1 — Cimientos
- Title: S1-1 DB Schema (LasParser)
  - Description: Definir esquema final para `las_files` y `las_curve_data` (columnas, tipos, índices, FKs, constraints). Incluir notas sobre performance y retención.
  - DoD: SQL/pseudocode aprobado, documentado en `architecture/lasparser_plan.md` y `knowledge/database.md`.
  - Estimate: 1 day
  - Owner: @alejandro-leone

- Title: S1-2 Models & Policies
  - Description: Documentar modelos `LasFile` y `LasCurveData`, GlobalScopes por `tenant_id`, y `LasFilePolicy` para permisos (owner + admin/support).
  - DoD: especificaciones y ejemplos de queries, policy matrix.
  - Estimate: 0.5 day
  - Owner: @alejandro-leone

- Title: S1-3 Fixtures
  - Description: Definir fixtures `valid_2.0.las`, `broken_header.las`, `file_with_extra_curves.las` y `file_with_bad_rows.las`.
  - DoD: fixtures enumeradas y ejemplos en `tickets/`.
  - Estimate: 0.5 day
  - Owner: @alejandro-leone

## Sprint 2 — Ingesta & Colas
- Title: S2-1 Upload Contract (Filament)
  - Description: Contracto de upload: validations, storage rules, Filament hook behavior, response payloads.
  - DoD: JSON examples and UI expectations in `flows/upload.md`.
  - Estimate: 1 day
  - Owner: @alejandro-leone

- Title: S2-2 Job Contract (ParseLasFileJob)
  - Description: Inputs/outputs, retries, idempotency, memory/time limits, versioning strategy.
  - DoD: Document in `architecture`.
  - Estimate: 1 day
  - Owner: @alejandro-leone

- Title: S2-3 Environments `.env` snippets
  - Description: Provide `.env.example` snippets and variable mappings for local/staging/production.
  - DoD: `architecture/env_example.md` present and reviewed by infra.
  - Estimate: 0.5 day
  - Owner: @alejandro-leone

## Sprint 3 — Parser & Persistencia
- Title: S3-1 LasReader Spec
  - Description: Pseudocode and examples for streaming parser; handling ~V/~C/~A sections and nulls.
  - DoD: Pseudocode unit-testable in `architecture`.
  - Estimate: 2 days
  - Owner: @alejandro-leone

- Title: S3-2 Persistence Spec
  - Description: Buffering, chunked inserts (configurable), error handling per row, metrics to emit.
  - DoD: Pseudocode and failure modes documented.
  - Estimate: 1 day
  - Owner: @alejandro-leone

- Title: S3-3 Final States and MIN_VALID_RATIO
  - Description: Define logic for COMPLETED/PARTIAL/FAILED and edge case examples.
  - DoD: Documented logic and examples.
  - Estimate: 0.5 day
  - Owner: @alejandro-leone

## Sprint 4 — Tests & Hardening
- Title: S4-1 Tests Matrix & Integration
  - Description: Define Unit + Feature + Integration tests, multi-tenant scenarios and fixtures mapping.
  - DoD: Test matrix and example tests described.
  - Estimate: 2 days
  - Owner: @alejandro-leone

- Title: S4-2 Runbook & Ops
  - Description: Finalize `architecture/ops_checks.md` and define responsibilities for infra.
  - DoD: Runbook reviewed by infra.
  - Estimate: 0.5 day
  - Owner: @alejandro-leone

---

Notes:
- These are high-level tickets; each can be split into smaller tasks during refinement. Ensure each created issue links back to `architecture/lasparser_plan.md` and relevant docs.
