# Infrastructure — servicios y configuración por ambiente

Resumen
Este documento registra las decisiones y la configuración de infraestructura que impactan el diseño del módulo LasParser.

Ambientes
- local
  - Storage: `local` (filesystem)
  - Queue: `database`
  - Purpose: desarrollo y tests locales
- staging
  - Storage: S3 (provisionado por ops)
  - Queue: `redis`
  - Purpose: preproducción, pruebas de carga ligeras
- production
  - Storage: S3 (privado)
  - Queue: `redis`
  - Deploy: Laravel Forge

Servicios y credenciales
- GitHub: repositorio central y PRs
- Laravel Forge: despliegues para staging/production
- Sentry / Observabilidad: definir DSN en staging/production

Storage & lifecycle
- En producción se recomienda lifecycle policy en S3 para aplicar retención (default 365 días).
- En local, usar `storage/app` para facilitar tests y fixtures.

Queues
- Local: `database` por simplicidad
- Staging/Prod: `redis` por performance; asegurar configuración de workers y autoscaling si es necesario.

Backups y data retention
- Base: PostgreSQL con backups periódicos gestionados por infra
- Retención: `las_files` por defecto 365 días; `las_curve_data` sujeta a políticas del cliente.

Runbook breve
- Promover variables de entorno desde `.env.example` y documentar en `architecture/lasparser_plan.md`.

