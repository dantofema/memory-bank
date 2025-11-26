# Modules — Bounded contexts y capas
- Cualquier módulo nuevo debe registrarse aquí antes de su implementación.
- Este archivo NO contiene código; define contrato de módulos y convenios para los desarrolladores.
Notas

  - Paths: `app/`, `database/migrations/` (cuando aplique en implementación).
  - Propósito: modelos compartidos, autenticación, tenancy, roles/permissions.
- Core (aplicación principal)

  - Paths en implementación: `Modules/LasParser/{database,Models,Jobs,Services,tests,docs}`
  - Contendrá: migraciones, modelos, jobs, servicios (reader), tests y fixtures.
  - Propósito: Ingesta y parsing de archivos LAS (2.0), persistencia y APIs básicas para lectura.
- LasParser
Módulos previstos (MVP)

- Durante la planificación, registrar decisiones, contratos y migraciones pseudo-código aquí.
- Cada módulo implementará un dominio vertical y contendrá su propio código, migraciones y tests bajo `Modules/<ModuleName>` (esto se aplicará solo en ramas de implementación; la rama `main` del Memory Bank mantiene sólo documentación).
Principios

Este documento describe los módulos (bounded contexts) previstos para el proyecto Geocodex y dónde residirán los artefactos al comenzar la implementación.


