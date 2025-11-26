# Sprint 3 — Parser & Persistencia — Deliverables y subtasks

Duración estimada: 5 días
Owner: Developer

Deliverables
- `LasReader` pseudocódigo (streaming, detection of sections, tokenization)
- Persistence strategy document (buffer, chunked inserts, error handling)

Subtickets
- S3-1: `Reader Spec` — pseudocode, examples with fixtures. DoD: unit-testable pseudocode.
- S3-2: `Persistence Spec` — chunk_size, batch insert strategy, metrics to emit. DoD: pseudocode and failure modes.
- S3-3: `Final States` — definition of COMPLETED/PARTIAL/FAILED and `MIN_VALID_RATIO` logic. DoD: examples and expected outcomes for fixtures.

Acceptance Criteria
- Documents present in `geocodex/architecture` and `geocodex/tickets` and approved by architect.

