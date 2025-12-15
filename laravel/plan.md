---
name: "Laravel Task Plan"
version: "1.2"
author: "Alejandro Leone"
last_updated: "2025-12-15"
purpose: "AI-optimized planning framework for Laravel development tasks"
default_tags:
  - must-run-sail
  - requires-db
  - creates-tests
  - module-scoped
  - strict-typing
tools:
  - sail
  - pest
  - phpstan
  - rector
  - pint
  - filament
phpstan_level: 5
context:
  project_type: "Laravel 12 with Filament v4, Livewire v3, Pest v4"
  team_size: 1
  priority: "MVP - speed and simplicity"
  quality_gates: "PHPStan level 6+, Pint, Rector"
---

# Plan para tareas de Laravel

## Resumen

Framework de planificación optimizado para agentes IA que ejecutan tareas de desarrollo Laravel. Proporciona metadatos
estructurados, plantillas reutilizables y convenciones estrictas para código mantenible y testeable.

# Convenciones

## Contexto del Proyecto

- **Tipo**: MVP - priorizar soluciones rápidas y sencillas
- **Equipo**: Un solo desarrollador - minimizar coordinación y mantenimiento
- **Stack**: Laravel 12, Filament v4, Livewire v3, Pest v4, Tailwind v4, Alpine.js
- **Infraestructura**: Laravel Sail (Docker) - todos los comandos vía `./vendor/bin/sail`
- **Arquitectura**: Laravel Modules instalado y configurado

## Calidad de Código (Obligatorio)

- **Tipado**: Fuertemente tipado en todo el código
- **Análisis estático**: PHPStan level 6+ (obligatorio al finalizar)
- **Formateo**: Pint (ejecutar siempre antes de commit)
- **Refactoring**: Rector (cuando aplique)
- **Testing**: Pest v4 (cobertura obligatoria para toda funcionalidad)

## Arquitectura y Diseño

### Clases y Métodos

- Todas las clases deben ser `final` (sin métodos `protected`)
- Controllers y Services: **un solo método público** (single responsibility)
- Métodos públicos: **nunca reciben arrays**, solo objetos DTO/Value Objects
- Lógica de negocio: reside en clases Service, no en Controllers

### Modelos Eloquent

- Declarar siempre las relaciones explícitamente
- Incluir siempre Factory para cada modelo
- Preferir enums de PHP sobre enums de base de datos

### Value Objects y DTOs

- Value Objects deben implementar `Wireable` (para uso en Livewire)
- En Eloquent: usar Cast para mapear Value Objects (Eloquent → Cast → Value Object)
- DTOs obligatorios para inputs/outputs de métodos públicos

### Validación

- Usar FormRequest o clases con `Illuminate\Validation`
- Lanzar excepciones (`throw`) cuando los datos no cumplen requisitos
- Preferir excepciones sobre valores `null` o `''` por defecto

## Base de Datos

### Migraciones

- Incluir **siempre índices** para optimizar consultas
- Documentar impacto en producción (tag: `migration-affects-production`)

### Estructura de Tests

- Repetir estructura de `app/` en `tests/Feature/`
- Mantener mismos namespaces y estructura de carpetas
- **Smoke Tests obligatorios** para todas las páginas → `tests/Browser/SmokeTest.php`

## API REST

### Versionado y Estructura

- Rutas siempre versionadas: `api/v1/*`
- Carpeta de controllers replica estructura: `app/Http/Controllers/Api/V1/`
- Endpoints **siempre** retornan `Resource` o `ResourceCollection` (Eloquent API Resources)

### Testing de APIs

- Incluir archivo `.http` para pruebas manuales/automatizadas
- Documentar ruta del `.http` en sección `files` del plan

## Testing y Debug

### Estrategia

- Priorizar testeabilidad: especificar fixtures, mocks, log points
- Logs detallados: indicar niveles (info/debug/error) para debugging
- Crear Interfaces cuando mejore simplicidad y aislamiento en tests

### Cobertura

- Tests obligatorios para toda funcionalidad nueva
- Feature tests preferidos sobre Unit tests
- Smoke tests para todas las páginas con UI

## Documentación

- Planes extensos: dividir en `steps` numerados con precondiciones y postcondiciones
- Documentación adicional: solo archivos `.md` en carpeta `docs/` con formato optimizado para IA
- Cambios en módulos: auto-contenimiento - todos los cambios dentro del mismo módulo

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

1. **Agente IA o Developer** → Lee este documento
2. **Genera Plan** → Usando templates y convenciones
3. **Otro Proceso** → Implementa el plan (humano o automatizado)
4. **Validación** → Ejecuta checklist de calidad

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

Plantillas estandarizadas que debe seguir cualquier task plan. Los identificadores de campos están en inglés; los
valores y descripciones en español.

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
http_file: "Modules/ModuleName/http/FeatureName.http"
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
http_file: "Modules/Resource/http/ListResources.http"
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

# Validación y uso

Checklist mínima para validar cualquier plan antes de implementarlo:

1. Ejecutar los comandos de Sail indicados en la plantilla.
2. Formatear y limpiar con Pint.
3. Ejecutar Rector si corresponde.
4. Ejecutar PHPStan al nivel indicado en el frontmatter.
5. Ejecutar los tests relevantes con Pest.

Comandos sugeridos (ejecutar desde el root del proyecto):

```zsh
./vendor/bin/sail up -d
./vendor/bin/sail artisan migrate:fresh --seed --no-interaction
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail composer run rector
./vendor/bin/sail composer run phpstan
./vendor/bin/sail test --filter=NombreDelTest
```

---

# Seguridad

- Nunca incluir secretos ni credenciales en este archivo o en ejemplos.
- Marcar migraciones sensibles con `migration-affects-production` y pedir revisión manual antes de deploy.
- Para cambios que exponen datos, documentar claramente los riesgos y los checks necesarios (autorizaciones, scopes).

---

# Supuestos

- Asumo que Sail está instalado y configurado en este repositorio y que `vendor/bin/sail` es el entrypoint correcto.
- Asumo que Laravel Modules sigue la convención `Modules/{ModuleName}/...`.

---

# Mantenimiento

- Para añadir una nueva tag, editar la sección `# Tags` y actualizar `default_tags` en el frontmatter si corresponde.
- Documentar cualquier cambio de convención en la sección `# Convenciones`.
