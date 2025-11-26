# Modules — Bounded contexts y capas

Propósito
- Definir los módulos (bounded contexts) previstos para Geocodex y las convenciones de ubicación de artefactos cuando se implemente.

Principios
- Cada módulo implementará un dominio vertical y contendrá su propio código, migraciones y tests bajo `Modules/<ModuleName>` **en ramas de implementación**.
- La rama principal del Memory Bank contiene solo documentación y planificación (no código). Ver `tickets/no_code_policy.md`.
- Paths de implementación sugeridos (cuando comience el desarrollo):
  - `Modules/<ModuleName>/database/migrations`
  - `Modules/<ModuleName>/Models`
  - `Modules/<ModuleName>/Jobs`
  - `Modules/<ModuleName>/Services`
  - `Modules/<ModuleName>/tests`
  - `Modules/<ModuleName>/docs`

Políticas de tenancy y usuarios
- Tenancy: usar la solución de tenancy provista por Filament (Filament v4 tenancy). No se utilizará un paquete externo.
- Single database: la aplicación usará una única base de datos. El aislamiento se implementará por scopes/columnas (ej. `tenant_id`).
- Usuarios:
  - Administradores (admin): no pertenecen a ningún tenant (tenant_id = NULL). Tienen permisos globales.
  - Usuarios normales: deben pertenecer a un tenant y `tenant_id` es obligatorio.

Módulos previstos (MVP)
- `LasParser`
  - Propósito: Ingesta y parsing de archivos LAS (2.0), persistencia y APIs básicas para lectura y administración.
  - Contendrá: migraciones, modelos, jobs, servicios (reader), tests y fixtures.
  - Paths en implementación: `Modules/LasParser/{database,Models,Jobs,Services,tests,docs}`

Core / Plataforma
- `Core` o la aplicación principal contendrá componentes compartidos: autenticación, tenancy bootstrap (Filament), configuración global y utilidades.

Notas
- Documentar cualquier nuevo módulo en este archivo antes de crear su código.
- Asegurarse de que las decisiones de tenancy y usuario estén representadas en ADRs (ver `architecture/decisions.md`).
