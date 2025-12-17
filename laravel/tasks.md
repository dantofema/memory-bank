---
name: "Laravel Task Planning"
version: "2.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "AI-optimized task planning templates and tag system for Laravel development"
default_tags:
  - must-run-sail
  - requires-db
  - creates-tests
  - module-scoped
  - strict-typing
related_docs:
  - conventions: "./conventions.md"
---

# Planificación de Tareas Laravel

## Resumen

Framework de planificación optimizada para agentes IA que ejecutan tareas de desarrollo Laravel. Proporciona templates estandarizados, sistema de tags y ejemplos prácticos para generar planes estructurados y reproducibles.

**Nota**: Para convenciones técnicas y arquitectónicas del proyecto, consultar [conventions.md](conventions.md).

---

# Alcance del Documento

## 🚨 IMPORTANTE: Este es un Framework de Planificación

Este documento **NO implementa código**. Su propósito es:

✅ **Permitido**:

- Describir planes estructurados y reproducibles
- Especificar archivos que deben crearse/modificarse
- Documentar comandos sugeridos (sin ejecutarlos)
- Definir tests necesarios y estrategias de validación
- Proporcionar templates y ejemplos de estructura

❌ **NO Permitido**:

- Editar o crear archivos PHP/Blade/JavaScript
- Ejecutar comandos de migración/seed/artisan
- Aplicar cambios directamente al código fuente
- Modificar archivos de configuración o rutas

## Flujo de Uso

1. **Agente IA o Developer** → Lee este documento y [conventions.md](conventions.md)
2. **Genera Plan** → Usando templates y convenciones
3. **Otro Proceso** → Implementa el plan (humano o automatizado)
4. **Validación** → Ejecuta checklist de calidad definido en conventions.md

---

# Sistema de Tags

Las tags ayudan a los agentes IA a determinar automáticamente requisitos, pasos y validaciones del plan.

## Formato

- Escribir en `kebab-case` y minúsculas
- Usar múltiples tags para describir completamente la tarea
- Documentar nuevas tags en esta sección antes de usarlas

## Tags Disponibles

### Ejecución y Entorno

- **must-run-sail** — Ejecutar comandos obligatoriamente vía Sail (`./vendor/bin/sail`)
- **requires-db** — Requiere base de datos disponible para pruebas o migraciones
- **seeder-required** — Se requiere un seeder para datos iniciales
- **async-job** — Operación que debe ejecutarse como Job en cola

### Testing y Calidad

- **creates-tests** — La tarea debe añadir tests (Pest)
- **strict-typing** — Requiere tipado fuerte y pasar PHPStan al nivel indicado

### Arquitectura

- **module-scoped** — Cambios confinados a un módulo (`Modules/{ModuleName}`)
- **dto-required** — Inputs/outputs deben representarse con DTOs o Value Objects
- **filament** — Implica Filament/Livewire UI

### Base de Datos

- **migration-affects-production** — Migración que impacta datos en producción (revisión adicional)
- **db-index-required** — La migración/tabla requiere índices explícitos
- **postgis** — Uso de PostGIS/Postgres espacial (tests contra pgsql/postgis)

### Contexto de Proyecto

- **quick-mvp** — Solución orientada a MVP (priorizar rapidez)
- **ai-optimizable** — Paso pensado para que un agente IA lo automatice

## Añadir Nuevas Tags

1. Documentar la semántica en esta sección
2. Actualizar `default_tags` en el frontmatter si es recurrente
3. Usar en al menos un template de ejemplo

---

# Templates de Plan

Plantillas estandarizadas que debe seguir cualquier task plan. Los identificadores de campos están en inglés; los valores y descripciones en español.

## Template: MigrationTask

```yaml
title: ""
description: "Descripción breve en español sobre lo que hace la migración"
files:
  - "database/migrations/xxxx_create_xxx_table.php"
sail_commands:
  - "./vendor/bin/sail artisan migrate --path=database/migrations/xxxx_create_xxx_table.php --no-interaction"
tests:
  - "tests/Feature/Module/ExampleMigrationTest.php"
tags:
  - migration-affects-production
  - requires-db
notes: "Notas operacionales y precauciones en español"
```

## Template: FeatureTask

```yaml
title: ""
description: "Descripción breve en español de la feature a implementar"
files:
  - "Modules/ModuleName/app/Models/Model.php"
  - "Modules/ModuleName/app/Services/Service.php"
  - "Modules/ModuleName/database/factories/ModelFactory.php"
  - "Modules/ModuleName/tests/Feature/FeatureNameTest.php"
  - "Modules/ModuleName/http/FeatureName.http" # archivo para pruebas manuales/automáticas
  - "Modules/ModuleName/http/FeatureName.bru"  # archivo Bruno para testing de API
http_file: "Modules/ModuleName/http/FeatureName.http"
bruno_file: "Modules/ModuleName/http/FeatureName.bru"
sail_commands:
  - "./vendor/bin/sail artisan migrate:fresh --seed --no-interaction"
  - "./vendor/bin/sail test --filter=FeatureNameTest"
tests:
  - "Modules/ModuleName/tests/Feature/FeatureNameTest.php"
tags:
  - module-scoped
  - creates-tests
  - strict-typing
notes: "Instrucciones y supuestos técnicos en español"
```

## Template: RefactorTask

```yaml
title: ""
description: "Descripción breve en español del refactor"
files:
  - "app/SomeClass.php"
  - "tests/Feature/SomeClassTest.php"
sail_commands:
  - "./vendor/bin/sail bin pint --dirty"
  - "./vendor/bin/sail composer test --filter=SomeClassTest"
tests:
  - "tests/Feature/SomeClassTest.php"
tags:
  - strict-typing
  - ai-optimizable
notes: "Checklist: Pint, Rector, PHPStan, Pest"
```

---

# Ejemplos

## Ejemplo 1 — Migración simple (uso de template MigrationTask)

```yaml
title: "Agregar tabla locations con geom"
description: "Crear tabla 'locations' con columna geom PostGIS y un índice espacial"
files:
  - "Modules/Geo/database/migrations/2025_12_07_000000_create_locations_table.php"
sail_commands:
  - "./vendor/bin/sail artisan migrate --path=Modules/Geo/database/migrations/2025_12_07_000000_create_locations_table.php --no-interaction"
tests:
  - "Modules/Geo/tests/Feature/CreateLocationTest.php"
tags:
  - postgis
  - requires-db
  - db-index-required
notes: "Asegurarse de que la migración enable_postgis_extension se haya ejecutado primero"
```

## Ejemplo 2 — Feature (module-scoped)

```yaml
title: "Crear endpoint API para listar resources"
description: "Endpoint api/v1/resources que retorna ResourceResource collection paginada"
files:
  - "Modules/Resource/app/Http/Controllers/Api/V1/ResourceController.php"
  - "Modules/Resource/app/Services/ListResourceService.php"
  - "Modules/Resource/database/factories/ResourceFactory.php"
  - "Modules/Resource/tests/Feature/ListResourceTest.php"
  - "Modules/Resource/http/ListResources.http" # archivo para pruebas manuales/automáticas
  - "Modules/Resource/http/ListResources.bru"  # archivo Bruno para testing de API
http_file: "Modules/Resource/http/ListResources.http"
bruno_file: "Modules/Resource/http/ListResources.bru"
sail_commands:
  - "./vendor/bin/sail artisan migrate:fresh --seed --no-interaction"
  - "./vendor/bin/sail test --filter=ListResourceTest"
tests:
  - "Modules/Resource/tests/Feature/ListResourceTest.php"
tags:
  - module-scoped
  - creates-tests
  - strict-typing
notes: "Usar DTO para inputs y Resource API para outputs; Controllers con un solo método público" 
```

---

# Mantenimiento

- Para añadir una nueva tag, editar la sección `# Sistema de Tags` y actualizar `default_tags` en el frontmatter si es recurrente.
- Documentar cualquier cambio en templates en este archivo.
- Mantener sincronizadas las referencias a [conventions.md](conventions.md).

