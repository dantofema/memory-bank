# Flujo: Upload de archivo LAS (Backoffice - Filament)

Objetivo
- Permitir a un usuario (owner con tenant_id) subir un archivo LAS y disparar el pipeline de parsing en background.

Pasos
1. Usuario en Filament selecciona tenant/context y sube `.las` (archivo único). Validaciones mínimas: tamaño máximo configurable (ej. 50MB), extensión `.las`.
2. Backend almacena el archivo en `FILESYSTEM_DISK` (local en dev, S3 en staging/production) y crea un registro en `las_files` con `status = PENDING`, `tenant_id`, `user_id`, `storage_path`, `size_bytes`.
3. Tras crear `las_files`, Filament hook despacha `ParseLasFileJob` (o un evento que sea escuchado por job dispatcher) y responde al usuario con el id del archivo y estado.
4. UI muestra estado y cola de jobs; permite acciones: reprocess, download original, view metadata.

Errores y fallback
- Validación de extensión/size devuelve error inmediato.
- Si el job no se puede despachar, `las_files.status` debe permanecer `PENDING` y registrar error en `error_log`.

Criterios de aceptación
- Diagrama y payloads (ejemplo JSON) documentados.
- Hook de Filament definido como contrato (sin código en esta etapa).

