---
name: "Laravel Technical Requirements"
version: "2.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "Technical conventions and architectural requirements for Laravel development"
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

# Requerimientos Técnicos Laravel

## Resumen

Convenciones técnicas y arquitectónicas para desarrollo Laravel. Define estándares de código, arquitectura, base de datos, APIs y testing que deben aplicarse en todos los proyectos.

---

# Convenciones

## Contexto del Proyecto

- **Tipo**: Priorizar soluciones rápidas y sencillas
- **Equipo**: Un solo desarrollador - minimizar coordinación y mantenimiento
- **Stack**: Laravel 12, Filament v4, Livewire v3 con Volt, Pest v4, Tailwind v4, Alpine.js
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
- Métodos públicos: **nunca reciben ni devuelven arrays**, solo objetos Spatie Laravel Data/Value Objects
- Lógica de negocio: reside en clases Service, no en Controllers

### Modelos Eloquent

- Declarar siempre las relaciones explícitamente
- Incluir siempre Factory para cada modelo
- Preferir enums de PHP sobre enums de base de datos

### Value Objects y Spatie Laravel Data

- **Implementar Value Objects de forma temprana** en el desarrollo (sugerir o implementar desde el inicio)
- Value Objects deben implementar `Wireable` (para uso en Livewire)
- En Eloquent: usar Cast para mapear Value Objects (Eloquent → Cast → Value Object)
- Spatie Laravel Data obligatorios para inputs/outputs de métodos públicos
- **Organización**: Value Objects se organizan en `app/ValueObjects/{ModelName}/`
    - Ejemplo: `app/ValueObjects/Product/PriceValueObject.php`
    - Value Objects compartidos (ej: `Money`, `Quantity`) van directamente en `app/ValueObjects/`

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
- Incluir archivo `.bru` (Bruno) para testing de APIs
- Documentar ruta del `.http` y `.bru` en sección `files` del plan

## Testing y Debug

### Estrategia

- Priorizar testeabilidad: especificar fixtures, mocks, log points
- Logs detallados: indicar niveles (info/debug/error) para debugging
- Crear Interfaces cuando mejore simplicidad y aislamiento en tests

### Cobertura

- Tests obligatorios para toda funcionalidad nueva
- **Cobertura de tests debe ser prácticamente del 100%**
- Feature tests preferidos sobre Unit tests
- Smoke tests para todas las páginas con UI

## Documentación

- Planes extensos: dividir en `steps` numerados con precondiciones y postcondiciones
- **Cuando se implementa código: NO generar documentación ni scripts adicionales**
- Documentación adicional: solo archivos `.md` en carpeta `docs/` con formato optimizado para IA
- Cambios en módulos: auto-contenimiento - todos los cambios dentro del mismo módulo

## División de Tareas

- **Cuando una tarea implica crear más de 3 archivos, separar en pasos**
- Cada paso debe utilizar un template apropiado (MigrationTask, FeatureTask, RefactorTask)
- Cada paso debe ser independiente y tener sus propios tests y validación
- Los pasos deben estar numerados con precondiciones y postcondiciones claras

---

# Validación de Calidad

Checklist mínima para validar cualquier implementación:

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

- Nunca incluir secretos ni credenciales en archivos de código o ejemplos.
- Marcar migraciones sensibles con `migration-affects-production` y pedir revisión manual antes de deploy.
- Para cambios que exponen datos, documentar claramente los riesgos y los checks necesarios (autorizaciones, scopes).

---

# Supuestos

- Asumo que Sail está instalado y configurado en este repositorio y que `vendor/bin/sail` es el entrypoint correcto.
- Asumo que Laravel Modules sigue la convención `Modules/{ModuleName}/...`.

---

# Mantenimiento

- Documentar cualquier cambio de convención en este archivo.
- Mantener sincronizado el frontmatter con las herramientas y versiones actuales del proyecto.

