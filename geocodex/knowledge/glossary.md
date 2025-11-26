# Glosario — términos y convenciones

- tenant_id: identificador de organización. Obligatorio en todas las tablas del módulo LasParser.
- user_id: identificador del usuario que realiza la acción. Siempre pertenece a un tenant.
- las_file: recurso que representa un archivo LAS subido. Tiene metadata y estado de procesamiento.
- las_curve_data: tabla que almacena filas de curvas (por depth) asociadas a un `las_file_id`.
- MIN_VALID_RATIO: umbral configurable para considerar `COMPLETED` (por defecto 0.90).
- chunk_size: número de filas para insert por batch (por defecto 1000).
- whitelist curves: `DEPT`, `GR`, `DT`, `RHOB`, `NPOR`.
- status: `PENDING` | `PROCESSING` | `COMPLETED` | `PARTIAL` | `FAILED`.

