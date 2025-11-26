# Auditoría mensual — checklist (ADRs, flows y tickets)

Objetivo: asegurar que la documentación del proyecto se mantenga relevante y que las decisiones importantes estén actualizadas.

Checklist (mensual)
- [ ] Revisar `architecture/decisions.md` y marcar ADRs que se deprecatearon o cambiaron.
- [ ] Verificar que `flows/` refleja la implementación actual (upload, ingestion pipeline).
- [ ] Confirmar que `tickets/` contiene incidentes relevantes y que los tickets abiertos tienen owner y ETA.
- [ ] Revisar `knowledge/` para nuevos términos o cambios (p. ej. cambio en `MIN_VALID_RATIO`).
- [ ] Ejecutar una búsqueda rápida por `geocodex/Modules/` para detectar código accidental y abrir ticket si existe.
- [ ] Ejecutar CI check (propuesta en `tickets/ci_checks.md`) si está disponible.

Resultados esperados
- Un resumen de acciones (1 page) con decisiones: "keep", "update", "deprecate".
- Crear tickets para cualquier acción requerida.

Owner: persona asignada en `tickets/owners.md` (revisar cada mes)

