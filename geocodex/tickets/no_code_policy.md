# Política: No código en rama de planificación (`main`)

Propósito: dejar constancia formal de la regla operativa que prohíbe la inclusión de código en la rama principal durante la fase de planificación.

Regla
- En la rama principal (`main` o `master` del Memory Bank) **no debe haber código**. Solo documentación, decisiones y artefactos de planificación.

Qué se considera "código"
- Archivos bajo `Modules/` que contengan PHP/JS/Python/compilables.
- Migraciones, modelos, jobs, controllers, o cualquier archivo con extensión de lenguaje fuente (ej. `.php`, `.js`, `.py`, `.java`) dentro de `geocodex/`.

Excepciones
- Snippets muy pequeños en documentación (pseudocódigo) que no sean ejecutables.

Responsables
- Owner del proyecto y reviewer arquitecto deben vigilar que se cumpla la regla.

Acciones si se detecta código
1. Eliminar inmediatamente del `main` y mover a branch de feature (`feat/...`) referenciando el ticket correspondiente.
2. Crear un ticket en `tickets/` con análisis y razones.
3. Validar que el autor realice PR apropiado desde una rama feature.

Verificación manual
- Antes de mergear PRs de documentación, revisar que no contengan `Modules/` con código.

Automatización (ver `tickets/ci_checks.md`) se propone un chequeo CI para detectar artefactos de código y fallar la pipeline en caso de encontrar alguno.

