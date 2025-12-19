---
name: "Laravel Technical Requirements"
version: "3.1"
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
phpstan_level: 6
context:
  project_type: "Laravel 12 with Filament v4, Livewire v3, Pest v4"
  team_size: 1
  priority: "MVP - speed and simplicity"
  quality_gates: "PHPStan level 6+, Pint, Rector"
value_objects:
  criteria: "Use VO if meets at least 1: business rules, reusability, invariants, semantic clarity"
  organization: "app/ValueObjects/{ModelName}/ or app/ValueObjects/ for shared"
  implementation: "Wireable interface, Eloquent Cast, validation in constructor"
  see_also: "./value-objects.md"
casts:
  organization: "app/Casts/{ModelName}/ or app/Casts/ for shared"
---

# Requerimientos Técnicos Laravel

## Resumen

Convenciones técnicas y arquitectónicas para desarrollo Laravel. Define estándares de código, arquitectura, base de
datos, APIs y testing que deben aplicarse en todos los proyectos.

**Versión 3.1**: criterios de decisión para Value Objects, tabla comparativa, ejemplo mínimo y referencia a
documentación completa en [`value-objects.md`](value-objects.md).

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

---

## Arquitectura y Diseño

### Estructura de Carpetas

- Las de Laravel standard son suficientes para este proyecto.
- Las de Filament son suficientes para este proyecto.
- Las de Livewire son suficientes para este proyecto.
- Las de Laravel Modules son suficientes para este proyecto.

### Livewire Volt

**Referencia completa**: ver [`livewire-volt.md`](livewire-volt.md) para convenciones detalladas sobre desarrollo con Volt.

**Reglas clave**:
- Sin Controllers: rutas llaman directamente a Volt
- Máximo 150 líneas por componente
- Extraer Actions para lógica de negocio
- Dividir en sub-componentes cuando sea necesario
- Feature tests obligatorios para cada componente

### Arquitectura de Módulos

**Referencia completa**: ver [`modules.md`](modules.md) para la especificación detallada.

Los módulos se comunican mediante **dos mecanismos complementarios** con roles semánticos claros:

#### 1. Interfaces (Commands / Queries) - Comunicación Síncrona

**Cuándo usar**: cuando un módulo necesita una **respuesta inmediata y determinista**.

- Representan capacidades, validaciones o decisiones del negocio
- Son sincrónicas, fuertemente tipadas y explícitas
- **No exponen modelos internos** ni detalles de persistencia
- Fallan inmediatamente y propagan el error al llamador

**Ejemplo**:

```php
// Catalog expone capacidad de verificar stock
interface CheckProductAvailabilityInterface
{
    public function check(ProductId $productId, Quantity $quantity): AvailabilityResult;
}
```

#### 2. Eventos de Dominio - Comunicación Asíncrona

**Cuándo usar**: para **notificar que un hecho relevante del negocio ya ocurrió**.

- Son señales inmutables, unidireccionales y sin valor de retorno
- **No influyen en decisiones de negocio** ni en el control del flujo
- Habilitan efectos secundarios: notificaciones, integraciones, reportes, auditoría
- Los consumidores deben ser idempotentes y tolerar duplicados

**Ejemplo**:

```php
// Order emite evento después de confirmar la orden
final readonly class OrderConfirmedEvent
{
    public function __construct(
        public OrderId $orderId,
        public Carbon $confirmedAt,
    ) {}
}
```

#### Reglas Obligatorias

1. ✅ **El estado del negocio se decide mediante interfaces, nunca mediante eventos**
2. ✅ **Los eventos comunican hechos del pasado; las interfaces gobiernan el presente**
3. ✅ **Los módulos core emiten eventos; los módulos periféricos los consumen**
4. ✅ **Los eventos no deben incluir datos sensibles**
5. ✅ **Interfaces nunca devuelven modelos Eloquent** (usar Data objects o Value Objects)

### Clases y Métodos

- Todas las clases deben ser `final` (sin métodos `protected`)
- Controllers y Actions: **un solo método público** (single responsibility)
- Métodos públicos: **nunca reciben ni devuelven arrays**, solo objetos Spatie Laravel Data/Value Objects
- Lógica de negocio: reside en clases Action, no en Controllers
- Nunca utilizamos clases Service

### Modelos Eloquent

- Declarar siempre las relaciones explícitamente
- Incluir siempre Factory para cada modelo
- Preferir enums de PHP sobre enums de base de datos

### Value Objects y Spatie Laravel Data

**📖 Para ejemplos completos y patrones de implementación, ver [`value-objects.md`](value-objects.md)**

- **Implementar Value Objects de forma temprana** en el desarrollo (sugerir o implementar desde el inicio)
- Value Objects deben implementar `Wireable` (para uso en Livewire)
- En Eloquent: usar Cast para mapear Value Objects (Eloquent → Cast → Value Object)
- Spatie Laravel Data obligatorios para inputs/outputs de métodos públicos
- **Organización**: Value Objects se organizan en `app/ValueObjects/{ModelName}/`
    - Ejemplo: `app/ValueObjects/Product/PriceValueObject.php`
    - Value Objects compartidos (ej: `Money`, `Quantity`) van directamente en `app/ValueObjects/`
- **Organización de Casts**: Los Casts se organizan en `app/Casts/{ModelName}/`
    - Ejemplo: `app/Casts/Product/PriceCast.php`
    - Casts compartidos (ej: `MoneyCast`, `QuantityCast`) van directamente en `app/Casts/`

#### Criterios para Usar Value Objects

**Usar Value Object si cumple AL MENOS 1 de estos criterios:**

1. **Tiene reglas de negocio propias**: la validación o comportamiento va más allá de tipos primitivos
    - ✅ `Money` (no permite negativos, maneja redondeo, formatea con moneda)
    - ✅ `PhoneNumber` (valida formato, normaliza, extrae código de país)
    - ❌ `string $name` (simple validación de longitud, no requiere VO)

2. **Se reutiliza en múltiples contextos**: aparece en varios modelos o servicios
    - ✅ `Address` (usado en Order, User, Merchant)
    - ✅ `Stock` (usado en Product, ProductVariant)
    - ❌ `product_description` (solo Product lo usa, simple string)

3. **No debe existir inválido**: la construcción debe garantizar estado válido siempre
    - ✅ `Email` (constructor valida formato, garantiza email válido)
    - ✅ `OrderStatus` (enum como VO garantiza valores permitidos)
    - ❌ `int $quantity` (puede ser negativo temporalmente durante validación)

4. **Aporta semántica clara al dominio**: el tipo primitivo no expresa suficiente significado
    - ✅ `PromotionPeriod` (expresa vigencia con start/end, no solo dos dates)
    - ✅ `DiscountValue` (distingue porcentaje vs monto fijo)
    - ❌ `bool $is_active` (el booleano es semánticamente claro)

**Regla práctica**: si dudás, empezá con tipo primitivo. Convertí a VO cuando el código muestre duplicación de
validaciones o lógica relacionada.

---

## Value Objects: Ejemplos Prácticos

### Tabla Comparativa: Usar VO vs No Usar VO

| Caso                 | Primitivo             | ¿Usar VO?                | Razón                                                                   |
|----------------------|-----------------------|--------------------------|-------------------------------------------------------------------------|
| Precio de producto   | `int $price_cents`    | ✅ **Sí - `Money`**       | Reglas de negocio (formato, redondeo, comparación), reutilizable        |
| Stock disponible     | `int $stock`          | ✅ **Sí - `Stock`**       | Reglas (no negativo, reservado vs disponible), reutilizable             |
| Teléfono de contacto | `string $phone`       | ✅ **Sí - `PhoneNumber`** | Validación compleja (formato internacional), normalización              |
| Email de usuario     | `string $email`       | ✅ **Sí - `Email`**       | No debe existir inválido, validación en constructor                     |
| Estado del pedido    | `string $status`      | ✅ **Sí - `OrderStatus`** | Valores limitados, transiciones con reglas, semántica clara             |
| Nombre de producto   | `string $name`        | ❌ **No**                 | Simple validación de longitud, no hay lógica de negocio                 |
| Descripción          | `string $description` | ❌ **No**                 | Solo almacenamiento, sin reglas propias                                 |
| Flag activo          | `bool $is_active`     | ❌ **No**                 | Booleano es semánticamente claro                                        |
| Cantidad en carrito  | `int $quantity`       | ⚠️ **Depende**           | Si solo valida > 0 → No. Si tiene lógica de conversión de unidades → Sí |

### Implementación Mínima de un Value Object

```php
<?php

declare(strict_types=1);

namespace App\ValueObjects;

use InvalidArgumentException;
use Livewire\Wireable;

final readonly class Money implements Wireable
{
    public function __construct(
        public int $cents,
        public string $currency = 'ARS'
    ) {
        if ($this->cents < 0) {
            throw new InvalidArgumentException('El monto no puede ser negativo');
        }
    }

    public function toLivewire(): array
    {
        return ['cents' => $this->cents, 'currency' => $this->currency];
    }

    public static function fromLivewire($value): self
    {
        return new self($value['cents'], $value['currency']);
    }
}
```

**Características clave:**

- `final readonly`: inmutabilidad garantizada
- Constructor valida invariantes (no permite estado inválido)
- `Wireable`: compatibilidad con Livewire
- **Para implementación completa con Cast, testing y métodos de dominio**: ver [`value-objects.md`](value-objects.md)

---

### Validación

- Usar FormRequest o clases con `Illuminate\Validation`
- Lanzar excepciones (`throw`) cuando los datos no cumplen requisitos
- Preferir excepciones sobre valores `null` o `''` por defecto

---

## Base de Datos

### Migraciones

- Incluir **siempre índices** para optimizar consultas
- Documentar impacto en producción (tag: `migration-affects-production`)

### Estructura de Tests

- Repetir estructura de `app/` en `tests/Feature/`
- Mantener mismos namespaces y estructura de carpetas
- **Smoke Tests obligatorios** para todas las páginas → `tests/Browser/SmokeTest.php`

---

## API REST

### Versionado y Estructura

- Rutas siempre versionadas: `api/v1/*`
- Carpeta de controllers replica estructura: `app/Http/Controllers/Api/V1/`
- Endpoints **siempre** retornan `Resource` o `ResourceCollection` (Eloquent API Resources)

### Testing de APIs

- Incluir archivo `.http` para pruebas manuales/automatizadas
- Incluir archivo `.bru` (Bruno) para testing de APIs
- Documentar ruta del `.http` y `.bru` en sección `files` del plan

---

## Testing y Debug

### Estrategia

- Priorizar testeabilidad: especificar fixtures, mocks, log points
- Logs detallados: indicar niveles (info/debug/error) para debugging
- Crear Interfaces cuando mejore simplicidad y aislamiento en tests
- **Value Objects**: testear validaciones en Unit tests, comportamiento en Feature tests con Eloquent Casts

### Cobertura

- Tests obligatorios para toda funcionalidad nueva
- **Cobertura de tests debe ser prácticamente del 100%**
- Feature tests preferidos sobre Unit tests
- Smoke tests para todas las páginas con UI
- Smoke test en `tests/Feature/SmokeTest.php`
- **Unit tests obligatorios para Value Objects**: validar constructor, invariantes, métodos de dominio

---

## Documentación

- Planes extensos: dividir en `steps` numerados con precondiciones y postcondiciones
- **Cuando se implementa código: NO generar documentación ni scripts adicionales**
- Documentación adicional: solo archivos `.md` en carpeta `docs/` con formato optimizado para IA
- Cambios en módulos: auto-contenimiento - todos los cambios dentro del mismo módulo

---

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

```bash
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
- Asumo que Laravel Modules (`nwidart/laravel-modules`) está instalado y sigue la convención `Modules/{ModuleName}/...`.
- Asumo que las reglas de comunicación entre módulos definidas en [`modules.md`](modules.md) son obligatorias para todo
  el proyecto.

---

# Mantenimiento

- Documentar cualquier cambio de convención en este archivo.
- Mantener sincronizado el frontmatter con las herramientas y versiones actuales del proyecto.

