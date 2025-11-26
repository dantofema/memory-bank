# Ops checks & runbook (mínimo) — LasParser

Objetivo: documentar variables de entorno, responsables y pasos operativos básicos para desplegar y operar LasParser.

Ambientes
- local
  - FILESYSTEM_DISK=local
  - QUEUE_CONNECTION=database
  - Uso: desarrollo y pruebas locales
- staging
  - FILESYSTEM_DISK=s3
  - QUEUE_CONNECTION=redis
  - Uso: preproducción
- production
  - FILESYSTEM_DISK=s3
  - QUEUE_CONNECTION=redis
  - Deploy via: Laravel Forge

Variables de entorno mínimas (ejemplos)
- AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_DEFAULT_REGION, AWS_BUCKET (staging/production)
- QUEUE_CONNECTION
- FILESYSTEM_DISK
- DB_CONNECTION, DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD
- SENTRY_DSN (staging/production)

Secrets & rotation
- No guardar secretos en Memory Bank. Documentar solo las claves de configuración (nombres de variables).
- La rotación y provisioning de secretos corre por cuenta del equipo de infra (Forge / Vault / AWS Secrets Manager).

Deploy checklist (staging/production)
- [ ] Variables de entorno provisionadas y validadas
- [ ] S3 bucket con lifecycle policy (365 days) configurado
- [ ] Workers redis activos (cantidad según throughput esperado)
- [ ] Backups de Postgres configurados

Incidentes y rollback
- Si un job masivo falla y causa carga: pause workers, investigar logs, revertir a snapshot DB si hay corrupción.
- Para problemas con parsing: re-process desde feature branch con cambios; nunca aplicar fixes rápidos en main.

Runbook mínimo para un archivo que queda en FAILED
1. Revisar `las_files.error_log` y logs de la job en Sentry
2. Validar si el error es de header (abort) o de filas (partial)
3. Si es header corrupto: marcar el archivo y notificar al user; si es recoverable, crear ticket para re-procesado
4. Si es por infra (timeout/db): re-dispatch job con parámetros limitados y monitorizar

Contactos
- Equipo infra / DevOps: nombre@dominio (documentado aquí como placeholder)
- Owner del módulo: developer_principal (assign in tickets)

Nota: este runbook es mínimo y debe ser extendido en staging antes de producción.

