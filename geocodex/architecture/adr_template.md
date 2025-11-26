# ADR Template — Architecture Decision Record
---

- Los jobs deben exponer `MIN_VALID_RATIO` como configuración; tests deben cubrir casos límite.
Consequences

- Seleccionar 0.90 por defecto; permitir configuración por tenant.
Decision

- 0.90: estricto, reduce datos inconsistentes
- 0.75: tolerante, puede producir más partials
Options considered

- Definir umbral mínimo de filas válidas para marcar un archivo como COMPLETED.
Context / Problema

* Autor: equipo
* Estado: Proposed
## [2025-11-26] MIN_VALID_RATIO para LasParser

Ejemplo

- Fecha y referencia a cualquier PR/ticket que implemente la decisión
Notas de seguimiento

- Impacto, limitaciones, pasos siguientes (migrations, cambios en infra)
Consequences / Justificación

- Qué se decidió (claro y conciso)
Decision

- Opción B: breve descripción y pros/cons
- Opción A: breve descripción y pros/cons
Options considered

- Breve descripción del problema que requiere decisión.
Context / Problema

* Autor: <nombre>
* Estado: Proposed | Active | Deprecated

## [YYYY-MM-DD] Título corto

Formato recomendado

Usar este template para cada decisión técnica importante (ADR). Guardar cada ADR como un bloque independiente en `architecture/decisions.md` o como archivos separados `architecture/adr-YYYY-MM-DD-<short-title>.md`.


