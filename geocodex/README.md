# Geocodex - Resumen Ejecutivo Técnico (Solo Desarrollo)

Breve objetivo ejecutivo
Crear un módulo SaaS fiable para la ingestión, parsing y visualización básica de archivos .las (LAS 2.0), priorizando la ingestión masiva y la persistencia eficiente para soportar archivos de tamaño razonable (hasta ~50MB) con un SLA objetivo <30 minutos por archivo en condiciones normales.

> Nota importante sobre tenancy
> Este proyecto será multi-tenant desde el diseño inicial. Todos los recursos persistentes tendrán `tenant_id` obligatorio y `user_id`. Además, por contrato: `user_id` siempre pertenece a un `tenant` (es decir, todos los users están asignados a un tenant). Los administradores (roles con permiso explícito) pueden acceder a datos de todos los tenants. Se planea integrar un paquete Laravel para tenancy (por ejemplo `spatie/laravel-multitenancy`) para gestionar identificación y contexto de tenant en requests.

## 1. Problema técnico
- Necesitamos un pipeline que permita subir archivos LAS, procesarlos en background de forma confiable y consultar resultados (metadatos + curvas) sin consumir memoria excesiva ni bloquear la base de datos.

## 2. Usuario objetivo
- Usuarios técnicos/operativos (ingenieros, geocientíficos) que suben y analizan datos de pozos/sondeos en un entorno SaaS. Casos principales: subir archivo, revisar estado, ver metadatos y descargar/visualizar curvas básicas.

## 3. Propuesta funcional (en una frase)
- Subida segura de .las → procesamiento en cola con parser por streaming → persistencia escalable de curvas → interfaz Filament para administración y visualización básica.

## 4. Alcance del MVP (priorizado)
- Subida de archivos (Backoffice / Filament) con aislamiento por tenant + usuario.
- Job en cola que parsea por streaming y persiste curvas por chunks.
- Base de datos: tablas para metadatos (`las_files`) y datos de curva (`las_curve_data`) optimizadas para lecturas; ambas contienen `tenant_id` y `user_id` (obligatorios).
- Recursos Filament: lista/tabla con estado, detalles y acciones (reprocesar, descargar); visualización limitada según `tenant_id` y `user_id`.
- Tests básicos (Unit + Feature) para upload, dispatch de job y parsing con fixtures, incluyendo tests de aislamiento por tenant.
(Excluir por ahora: soporte extenso para versiones antiguas, downsampling complejo en tiempo real y ingestión masiva sin pruebas de carga).

## 5. Arquitectura y stack (resumen)
- Laravel 12, PostgreSQL, Filament, Jobs (queue: redis en producción), S3 para almacenamiento de originales en producción (local en dev).
- Tenancy: usar un paquete de Laravel para gestionar contexto de tenant y middleware (por ejemplo `spatie/laravel-multitenancy` o similar). Aplicar `tenant_id` obligatorio en migraciones y scope global que filtre por tenant (salvo para admins con permiso explícito).
- Parser por streaming (fopen/fgets), buffer + inserts por chunks (1000 filas) para persistencia.
- Observabilidad mínima: Sentry para errores y métricas clave; Telescope en staging.

## 6. Riesgos técnicos
- Insert masivo sin chunking puede bloquear la DB.
- Parsing en memoria para archivos grandes puede agotar RAM.
- Multi-tenancy mal aplicado puede exponer datos de otros tenants o romper búsquedas si scopes son inconsistentes.
- SLA incumplido si las colas no están dimensionadas (workers/redis).
- Variaciones en formatos LAS requieren robustez y pruebas.

## 7. Métricas de calidad
- Tests: Unit + Feature para flujos críticos, más pruebas de aislamiento por tenant.
- Performance: uso de memoria por job, throughput rows/sec, tiempo total por archivo.
- Seguridad: policies y global scopes que garanticen acceso por owner/tenant; roles de admin con alcance global.
- Observabilidad: errores en Sentry, métricas jobs_processed / rows_inserted.

---

Decisiones críticas (P0) — resumen
- P0-01: Objetivo de negocio principal
  - Selección: Ingesta masiva + visualización básica (priorizar ingesta fiable).

- P0-02: SLA / Latencia por archivo
  - Selección: <30 minutos para archivos razonables (<=50MB).

- P0-03: Volumen esperado
  - Selección: Punto de partida 10–100 archivos/día por sistema, tamaño medio 5–50MB (ajustar con telemetría).

- P0-04: Versiones LAS a soportar
  - Selección: Empezar con LAS 2.0 y diseñar extensible a 1.x.

- P0-05: Soporte para archivos comprimidos / multipart
  - Selección: Empezar sin soporte; diseñar extensible para .gz y uploads resumables.

- P0-06: Almacenamiento de originales
  - Selección: S3 privado en producción; `local` en desarrollo.

- P0-07: Driver de colas en producción
  - Selección: `redis` (performance); `database` aceptable para MVP.

- P0-08: Lista blanca de curvas mínimas
  - Selección: `DEPT` (Depth), `GR`, `DT`, `RHOB`, `NPOR` como conjunto mínimo; extras en JSONB.

- P0-09: Tenancy obligatorio (nuevo)
  - Pregunta: ¿El sistema será multi-tenant desde el inicio y cómo se gestiona?
  - Selección: Sí — `tenant_id` obligatorio en todas las tablas de recursos; `user_id` siempre asociado a un `tenant`. Integrar un paquete Laravel para tenancy (ej. `spatie/laravel-multitenancy`) y aplicar middleware/global scopes para garantizar aislamiento por tenant. Los admins con permiso explícito saltan el scope.

Decisiones P1 (alta prioridad) — resumen
- Tolerancia a errores: Híbrido — abortar en errores críticos del header/estructura; salvar filas válidas y registrar errores por fila; usar umbral `MIN_VALID_RATIO`.
- Inserción masiva: Chunked inserts (chunk_size = 1000) como MVP; diseñar para poder añadir COPY/pg_copy en producción si la carga lo requiere.
- Columnas desconocidas: Guardar metadatos/curvas extras en JSONB (`header_meta` / `curves_meta`) y extras por fila cuando haga falta.
- Indexación/particionamiento: Empezar sin particionamiento; añadir índices compuestos (`las_file_id`, `depth`) y reevaluar tras pruebas de carga.
- Modelo de tenancy: `tenant_id` y `user_id` obligatorios; implementar GlobalScopes que filtren por `tenant_id` y `user_id` según corresponda; los `admin` con permiso explícito saltan el scope.
- Retención/versioning: Retención configurable por cliente (por defecto 365 días); reprocesos crean nuevas filas en `las_files` (versionado simple).

Resumen de Sprints (entregables clave por sprint — alto nivel)
- Sprint 1 — Cimientos: migraciones `tenants`, `las_files` & `las_curve_data` (con `tenant_id` + `user_id`), modelo `LasFile` con scope por tenant y policy básica, tests de aislamiento por tenant.
- Sprint 2 — Ingesta & Colas: Recurso Filament para upload, job `ParseLasFileJob`, dispatch y estado visual en Filament, tests de integración (incluyendo tests multi-tenant).
- Sprint 3 — Parser I: Stream reader, detección de secciones (~V, ~W, ~C, ~A) y extracción de header/metadatos.
- Sprint 4 — Parser II: Mapeo dinámico de curvas, parsing de `~A`, manejo de nulos, tests unitarios con fixtures.
- Sprint 5 — Persistencia optimizada: buffer + inserts por chunks, manejo de errores, estado COMPLETED/FAILED, pruebas de carga ligeras.
- Sprint 6 — UX & Visualización: RelationManager/tabla para curvas, acciones re-procesado/descarga, revisión final de seguridad.

Próximos pasos inmediatos (acciones concretas)
1. Crear branch de trabajo local y añadir pruebas mínimas: fixtures `valid_2.0.las` y `broken_header.las`.
2. Implementar migraciones básicas: `tenants` (si no existe), `las_files` y `las_curve_data` con `tenant_id` y `user_id` obligatorios y FK apropiadas.
3. Implementar modelo `Tenant` y `LasFile` con GlobalScopes por `tenant_id`, y `LasFilePolicy` (owner-only + admins con permiso global).
4. Añadir Recurso Filament para upload (hook que dispatch `ParseLasFileJob`) con validación de `tenant_id` al crear.
5. Implementar `LasReader` (streaming) y job `ParseLasFileJob` con buffer & chunked inserts (1000) que incluyan `tenant_id` en cada fila.
6. Añadir tests PEST: upload → job dispatched; parser unit tests con fixtures; test de aislamiento multi-tenant.
7. En dev: usar `local` storage y `database` queue si `redis` no está disponible; en producción habilitar S3 + redis.

Notas operacionales y recomendaciones rápidas
- Parser por streaming: usar fopen/fgets + `preg_split('/\s+/')` para separar columnas. Evitar cargar todo el archivo en RAM.
- Persistencia: usar `DB::table(...)->insert($buffer)` para performance; envolver por transacciones por chunk si es necesario.
- Observabilidad: instrumentar jobs con métricas (tiempo, filas insertadas, errores) y enviar errores a Sentry.
- Tests: correr las pruebas afectadas tras cada cambio (`php artisan test --filter=...`) y mantener fixtures en el módulo.

Metadatos
**Estado:** Borrador / En validación
**Última actualización:** 26/11/2025

---

## Estructura del proyecto (documentación)

Este repositorio contiene solo la documentación y planificación del módulo LasParser según la guía `guide.md` del Memory Bank. No contiene código implementado en la rama principal durante la fase de planificación.

Carpetas importantes:
- `architecture/` — decisiones arquitectónicas y contratos (ver `decisions.md`, `modules.md`, `infrastructure.md`, `lasparser_plan.md`).
- `flows/` — flujos funcionales críticos (`upload.md`, `ingestion_pipeline.md`).
- `tickets/` — tickets por sprint y entregables (`sprint_1_mvp.md`, `sprint_2_ingest.md`, `sprint_3_parser.md`).
- `knowledge/` — glosario y convenciones (`glossary.md`).

Reglas operativas
- No agregar código en esta rama: las implementaciones se harán en ramas de feature bajo `Modules/` y con PR vinculados a los tickets.
- Storage/queue por ambiente: `local` => local/disk + database queue; staging/prod => S3 + redis.
