---
name: "Laravel Task Plan"
version: "1.1"
author: "Alejandro Leone"
last_updated: "2025-12-07"
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
phpstan_level: 6 or higher
---

# Plan para tareas de Laravel

## Resumen

Documento de referencia para planificar y ejecutar tareas de desarrollo en este proyecto Laravel. Está optimizado para
uso humano y para agentes IA que necesiten extraer metadatos, tags y plantillas de acción.

# Convenciones

- Prefiero los enums de PHP a los de las bases de datos.
- Es un MVP, la solución debe ser rápida y sencilla.
- El desarrollo está a cargo de un solo programador; priorizar soluciones que minimicen coordinación y mantenimiento.
- Utilizar Pest v4 para los tests.
- Los tests deben repetir la estructura de la app: `app/` ↔ `tests/Feature/` (mismos namespaces y estructura de carpetas
  cuando aplique).
- El análisis estático es muy estricto: PHPStan, Rector y Pint. Siempre validar al final.
- Usamos Laravel Sail: todos los comandos se deben ejecutar dentro del contenedor con `vendor/bin/sail`.
- Usamos Laravel Modules; está instalado y configurado.
- Ya están instalados y configurados: PHPStan, Rector, Pint, Laravel Module, Tailwind, Pest v4, Livewire, AlpineJS,
  FilamentPHP v4.
- PHPStan como mínimo está en level 6 (configurable en frontmatter).
- Los métodos públicos nunca reciben arrays; reciben objetos DTO.
- La lógica de negocio reside en una clase Service.
- Cuando trabajamos en un módulo, todos los cambios se deben hacer en ese módulo (auto-contenimiento).
- Si el plan necesita documentar, solo puede crear un archivo `.md` dentro de la carpeta `docs/` con formato prompt
  optimizado para IA.
- Los modelos Eloquent deben declarar las relaciones.
- Los modelos Eloquent siempre tienen Factory.
- Las migraciones siempre deben incluir índices para optimizar las consultas.
- Las clases deben ser siempre `final`, sin métodos `protected`.
- Todo el código debe estar fuertemente tipado.
- Siempre se deben tipar los elementos de los arrays (usar shapes/array types donde corresponda).
- Validar tipos y lanzar excepciones (`throw`) cuando el dato no es el esperado.
- Cuando un dato es requerido, preferimos lanzar una excepción en lugar de usar `null` o `''` por defecto.
- Las rutas de la API siempre deben estar versionadas: `api/v1`; la carpeta de los controllers debe replicar esta
  estructura.
- Los Controllers y Services tienen solo un método público (API: single responsibility por clase).
- Cuando un endpoint REST API se diseña, SIEMPRE debe devolver un `Resource` o una `ResourceCollection` (usar Eloquent
  API Resources).
- Priorizar la testeabilidad y la capacidad de debug: todo plan debe especificar cómo se probará y depurará (fixtures,
  mocks, log points).
- Logs detallados: los steps que impliquen ejecución deben indicar qué logs añadir y qué niveles (info/debug/error) se
  requieren para facilitar el debugging en tests y en entorno local.
- Crear `Interfaces` cuando beneficie la simplicidad y el aislamiento en los tests (incluir contract/interface en el
  plan y cómo mockearlo).
- Si hay validación de datos, usar una clase de validación de Laravel (FormRequest o una clase que utilice
  `Illuminate\Validation`), documentar reglas y mensajes en el plan.
- Cuando haya endpoints, el plan debe incluir un archivo `.http` (o colección Postman) para pruebas manuales y
  automatizadas; indicar la ruta del `.http` en la sección `files` del template.
- Cuando un plan es extenso, dividirlo en `steps` numerados y describir precondiciones y postcondiciones para cada step.

# Alcance: SOLO PLAN (NO IMPLEMENTACIÓN)

- Este documento SOLO contiene indicaciones para crear un plan. NO implementa código.
- NO debe editar ni crear archivos PHP ni aplicar cambios al código fuente.
- El objetivo es armar un plan reproducible y testeable que otro proceso (humano o automatizado) pueda ejecutar
  posteriormente.
- Nunca ejecutar comandos de migración/seed/fixtures directamente desde este documento; el plan puede sugerir comandos
  pero no ejecutarlos.
- Si un plan requiere cambios en archivos, describirlos exhaustivamente en `files` y `changes_expected` sin crear o
  editar los archivos aquí.

# Tags

Las `tags` ayudan a decidir pasos automáticos y requisitos del plan. Deben escribirse en `kebab-case` y en minúsculas.

Tags recomendadas y su semántica:

- must-run-sail — Ejecutar comandos obligatoriamente vía Sail (./vendor/bin/sail ...).
- requires-db — Requiere base de datos disponible para pruebas o migraciones.
- creates-tests — La tarea debe añadir tests (Pest).
- module-scoped — Cambios confinados a un módulo (Modules/{ModuleName}).
- strict-typing — Requiere tipado fuerte y pasar PHPStan al nivel indicado.
- dto-required — Inputs/outputs deben representarse con DTOs o Value Objects.
- migration-affects-production — Migración que impacta datos en producción (revisión adicional).
- db-index-required — La migración/tabla requiere índices explícitos.
- quick-mvp — Solución orientada a MVP (priorizar rapidez).
- ai-optimizable — Paso pensado para que un agente IA lo automatice.
- postgis — Uso de PostGIS/Postgres espacial (tests contra pgsql/postgis).
- filament — Implica Filament/Livewire UI.
- seeder-required — Se requiere un seeder para datos iniciales.
- async-job — Operación que debe ejecutarse como Job en cola.

Añadir nuevas tags: documentar la semántica en esta sección y añadirlas en `default_tags` del frontmatter si son
recurrentes.

# Templates de Plan

A continuación hay plantillas estandarizadas que debe seguir cualquier task plan. Los identifiers de campos están en
inglés; los valores y descripciones en español.

Template: MigrationTask

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

Template: FeatureTask

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

Template: RefactorTask

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

# Ejemplos

Ejemplo 1 — Migración simple (uso de template MigrationTask)

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

Ejemplo 2 — Feature (module-scoped)

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

# Seguridad

- Nunca incluir secretos ni credenciales en este archivo o en ejemplos.
- Marcar migraciones sensibles con `migration-affects-production` y pedir revisión manual antes de deploy.
- Para cambios que exponen datos, documentar claramente los riesgos y los checks necesarios (autorizaciones, scopes).

# Supuestos

- Asumo que Sail está instalado y configurado en este repositorio y que `vendor/bin/sail` es el entrypoint correcto.
- Asumo que Laravel Modules sigue la convención `Modules/{ModuleName}/...`.

# Mantenimiento

- Para añadir una nueva tag, editar la sección `# Tags` y actualizar `default_tags` en el frontmatter si corresponde.
- Documentar cualquier cambio de convención en la sección `# Convenciones`.
