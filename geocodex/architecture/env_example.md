# .env.example snippets — LasParser (por ambiente)

## Local
FILESYSTEM_DISK=local
QUEUE_CONNECTION=database
DB_CONNECTION=pgsql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=geocodex_local
DB_USERNAME=postgres
DB_PASSWORD=password

## Staging / Production
FILESYSTEM_DISK=s3
QUEUE_CONNECTION=redis
# AWS credentials (managed by infra / secrets manager)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=geocodex-staging-or-prod

# Optional (observability)
SENTRY_DSN=

# Feature flags / config
MIN_VALID_RATIO=0.90
CHUNK_SIZE=1000
RETENTION_DAYS=365

# Notes: No colocar valores secretos en este archivo público. Infra debe provisionar secretos en el entorno.

