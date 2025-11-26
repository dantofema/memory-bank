# CI checks proposal — detectar código en rama de planificación

Objetivo: Proveer un documento con la lógica de verificación que debe implementarse en CI para evitar que código fuente se suba al repo de planificación.

Regla básica
- El CI debe fallar si en el commit/PR aparece al menos un archivo con extensión de código (`.php`, `.js`, `.py`, `.go`, `.java`, etc.) bajo la carpeta `geocodex/Modules/`.

Ejemplo de script (bash) — propuesta
```bash
# exit non-zero if any code-like file is present under geocodex/Modules
set -e
found=$(git diff --name-only $BASE_SHA $HEAD_SHA | grep -E '^geocodex/Modules/.*\.(php|js|py|java|go)$' || true)
if [ -n "$found" ]; then
  echo "ERROR: Found code files in geocodex/Modules/:";
  echo "$found";
  exit 1
fi
```

Integración
- Añadir job en la pipeline que ejecute este script en PR validation en repositorios que usen Memory Bank como docs.
- Opcional: reportar la lista de archivos encontrados en el ticket creado automáticamente.

Limitaciones
- No sustituye revisiones humanas (owner/reviewer). Es un safety net.

Siguiente paso
- Crear ticket de infra para instalar la job de CI en el repositorio central o en el template de organizaciones.
```
