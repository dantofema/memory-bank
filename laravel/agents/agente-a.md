---
name: "Agente A — Contratos, Data, Enums y Value Objects"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Definir la frontera pública del módulo mediante tipos y contratos"
role: "Define el *qué* del módulo, nunca el *cómo*"
dependencies:
  - conventions.md
  - value-objects.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
---

# Agente A — Contratos, Data, Enums y Value Objects

## Resumen

Agente responsable de **definir la frontera pública del módulo**.
Produce **solo tipos y contratos**, sin ninguna implementación de negocio ni infraestructura.

**Principio fundamental**: Este agente define el *qué* del módulo, nunca el *cómo*.

---

## Alcance Estricto

### ✅ Archivos Permitidos

#### Dentro del módulo

```
Modules/{ModuleName}/
├── Contracts/
│   ├── Commands/          # Interfaces síncronas que modifican estado
│   ├── Queries/           # Interfaces síncronas de solo lectura
│   └── Services/          # Interfaces de servicios del módulo
├── Data/                  # Spatie Laravel Data objects
├── Enums/                 # Enumeraciones PHP
└── ValueObjects/          # Value Objects con reglas de negocio
    ├── {ModelName}/       # VOs específicos de un modelo
    └── Shared/            # VOs compartidos entre modelos
```

#### Tests

```
Modules/{ModuleName}/tests/Unit/
├── Contracts/            # Tests de contratos (si aplica)
├── ValueObjects/         # Tests obligatorios de VOs
└── Data/                 # Tests de Data objects (si hay validación)
```

---

### ❌ Archivos Prohibidos

**No crear bajo ningún concepto**:

- ❌ Actions
- ❌ Services
- ❌ Models (Eloquent)
- ❌ Repositories
- ❌ Controllers
- ❌ Filament
- ❌ Events
- ❌ Migrations
- ❌ Factories
- ❌ Listeners
- ❌ Jobs
- ❌ Casts (pertenecen al Agente que implementa los modelos)
- ❌ Cualquier infraestructura

---

## Responsabilidades del Agente A

### 1. Definir Contratos de Módulo

**Interfaces síncronas** que exponen capacidades del módulo a otros módulos.

#### Commands (modifican estado)

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Contracts\Commands;

use Modules\Catalog\Data\ProductData;
use Modules\Catalog\ValueObjects\ProductId;

interface CreateProductInterface
{
    /**
     * Crea un nuevo producto en el catálogo.
     *
     * @throws ProductAlreadyExistsException
     * @throws InvalidProductDataException
     */
    public function execute(ProductData $data): ProductId;
}
```

#### Queries (solo lectura)

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Contracts\Queries;

use Modules\Catalog\Data\ProductData;
use Modules\Catalog\ValueObjects\ProductId;

interface GetProductInterface
{
    /**
     * Obtiene un producto por su ID.
     *
     * @throws ProductNotFoundException
     */
    public function execute(ProductId $id): ProductData;
}
```

**Reglas obligatorias**:
- ✅ Siempre reciben y devuelven Data objects o Value Objects
- ✅ Nunca exponen modelos Eloquent
- ✅ Documentan excepciones que pueden lanzar
- ✅ Nombre del método: `execute()` para mantener consistencia

---

### 2. Crear Data Objects (Spatie Laravel Data)

**Input/Output de contratos públicos y métodos**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Data;

use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Spatie\LaravelData\Data;

final class ProductData extends Data
{
    public function __construct(
        public readonly ?ProductId $id,
        public readonly string $name,
        public readonly string $description,
        public readonly Money $price,
        public readonly int $stock,
        public readonly bool $isActive,
    ) {}

    /**
     * Reglas de validación para input externo.
     */
    public static function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'description' => ['required', 'string'],
            'price.cents' => ['required', 'integer', 'min:0'],
            'stock' => ['required', 'integer', 'min:0'],
            'isActive' => ['boolean'],
        ];
    }
}
```

**Reglas obligatorias**:
- ✅ Usar `final class`
- ✅ Todas las propiedades `readonly`
- ✅ Usar Value Objects cuando aplique (ver criterios en conventions.md)
- ✅ Incluir `rules()` si se usa para input de usuario
- ✅ Documentar transformaciones complejas

---

### 3. Crear Value Objects

**Ver `value-objects.md` para documentación completa**.

#### Criterios para crear un VO (AL MENOS 1):

1. **Reglas de negocio propias**
2. **Reutilizable en múltiples contextos**
3. **No debe existir inválido** (constructor garantiza invariantes)
4. **Aporta semántica clara al dominio**

#### Ejemplo: Money Value Object

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\ValueObjects;

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

        if (strlen($this->currency) !== 3) {
            throw new InvalidArgumentException('El código de moneda debe tener 3 caracteres');
        }
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('No se pueden sumar montos de diferentes monedas');
        }

        return new self($this->cents + $other->cents, $this->currency);
    }

    public function format(): string
    {
        return sprintf('%s %.2f', $this->currency, $this->cents / 100);
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

**Características obligatorias**:
- ✅ `final readonly class`
- ✅ Validación en constructor (invariantes)
- ✅ Implementar `Wireable` (para Livewire)
- ✅ Métodos de dominio (ej: `add()`, `format()`)
- ✅ Inmutabilidad total

#### Organización de Value Objects

```
Modules/Catalog/ValueObjects/
├── Product/
│   ├── ProductId.php
│   ├── Price.php
│   └── Stock.php
└── Shared/
    ├── Money.php
    └── Quantity.php
```

---

### 4. Crear Enums

**Valores limitados y conocidos del dominio**.

```php
<?php

declare(strict_types=1);

namespace Modules\Order\Enums;

enum OrderStatus: string
{
    case PENDING = 'pending';
    case CONFIRMED = 'confirmed';
    case PROCESSING = 'processing';
    case SHIPPED = 'shipped';
    case DELIVERED = 'delivered';
    case CANCELLED = 'cancelled';

    /**
     * Estados que permiten cancelación.
     */
    public function canBeCancelled(): bool
    {
        return in_array($this, [
            self::PENDING,
            self::CONFIRMED,
        ], true);
    }

    /**
     * Etiqueta para mostrar en UI.
     */
    public function label(): string
    {
        return match ($this) {
            self::PENDING => 'Pendiente',
            self::CONFIRMED => 'Confirmado',
            self::PROCESSING => 'En proceso',
            self::SHIPPED => 'Enviado',
            self::DELIVERED => 'Entregado',
            self::CANCELLED => 'Cancelado',
        };
    }
}
```

**Reglas obligatorias**:
- ✅ Usar `enum` de PHP (no enums de base de datos)
- ✅ Tipo `string` o `int` según semántica
- ✅ Métodos helper para lógica de dominio
- ✅ Método `label()` para UI

---

## Testing Obligatorio

### Unit Tests para Value Objects

**Cobertura obligatoria del 100%**.

```php
<?php

declare(strict_types=1);

use Modules\Catalog\ValueObjects\Money;

it('crea un Money válido', function () {
    $money = new Money(cents: 1500, currency: 'ARS');

    expect($money->cents)->toBe(1500)
        ->and($money->currency)->toBe('ARS');
});

it('lanza excepción si los centavos son negativos', function () {
    new Money(cents: -100, currency: 'ARS');
})->throws(InvalidArgumentException::class, 'El monto no puede ser negativo');

it('lanza excepción si el código de moneda es inválido', function () {
    new Money(cents: 1000, currency: 'US');
})->throws(InvalidArgumentException::class, 'El código de moneda debe tener 3 caracteres');

it('suma dos montos de la misma moneda', function () {
    $money1 = new Money(cents: 1000, currency: 'ARS');
    $money2 = new Money(cents: 500, currency: 'ARS');

    $result = $money1->add($money2);

    expect($result->cents)->toBe(1500)
        ->and($result->currency)->toBe('ARS');
});

it('lanza excepción al sumar montos de diferentes monedas', function () {
    $money1 = new Money(cents: 1000, currency: 'ARS');
    $money2 = new Money(cents: 500, currency: 'USD');

    $money1->add($money2);
})->throws(InvalidArgumentException::class, 'No se pueden sumar montos de diferentes monedas');

it('formatea correctamente el monto', function () {
    $money = new Money(cents: 1550, currency: 'ARS');

    expect($money->format())->toBe('ARS 15.50');
});

it('se serializa y deserializa para Livewire', function () {
    $money = new Money(cents: 2000, currency: 'USD');

    $serialized = $money->toLivewire();
    $deserialized = Money::fromLivewire($serialized);

    expect($deserialized->cents)->toBe(2000)
        ->and($deserialized->currency)->toBe('USD');
});
```

### Unit Tests para Enums

```php
<?php

declare(strict_types=1);

use Modules\Order\Enums\OrderStatus;

it('permite cancelar estados pending y confirmed', function () {
    expect(OrderStatus::PENDING->canBeCancelled())->toBeTrue()
        ->and(OrderStatus::CONFIRMED->canBeCancelled())->toBeTrue();
});

it('no permite cancelar estados processing, shipped, delivered', function () {
    expect(OrderStatus::PROCESSING->canBeCancelled())->toBeFalse()
        ->and(OrderStatus::SHIPPED->canBeCancelled())->toBeFalse()
        ->and(OrderStatus::DELIVERED->canBeCancelled())->toBeFalse();
});

it('retorna las etiquetas correctas', function () {
    expect(OrderStatus::PENDING->label())->toBe('Pendiente')
        ->and(OrderStatus::CONFIRMED->label())->toBe('Confirmado')
        ->and(OrderStatus::CANCELLED->label())->toBe('Cancelado');
});
```

### Tests para Data Objects (opcional)

**Solo si hay validación o transformación compleja**.

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Data\ProductData;
use Modules\Catalog\ValueObjects\Money;

it('valida correctamente los datos del producto', function () {
    $data = ProductData::from([
        'name' => 'Producto Test',
        'description' => 'Descripción del producto',
        'price' => ['cents' => 1500, 'currency' => 'ARS'],
        'stock' => 10,
        'isActive' => true,
    ]);

    expect($data->name)->toBe('Producto Test')
        ->and($data->price)->toBeInstanceOf(Money::class)
        ->and($data->price->cents)->toBe(1500);
});
```

---

## Validación de Calidad

### Checklist Obligatorio

Antes de considerar completa la tarea:

1. ✅ **PHPStan level 6** sin errores
   ```bash
   ./vendor/bin/sail composer run phpstan
   ```

2. ✅ **Pint** ejecutado
   ```bash
   ./vendor/bin/sail bin pint --dirty
   ```

3. ✅ **Rector** ejecutado (si aplica)
   ```bash
   ./vendor/bin/sail composer run rector
   ```

4. ✅ **Tests con cobertura del 100%**
   ```bash
   ./vendor/bin/sail test --filter=ValueObjects
   ./vendor/bin/sail test --filter=Enums
   ```

5. ✅ **Tipado fuerte**: sin `mixed`, `array` o tipos genéricos en signatures públicas

6. ✅ **Documentación**: todos los contratos tienen docblocks con `@throws`

---

## Entregables del Agente A

Al finalizar, el Agente A debe haber creado:

### Archivos de Código

- [ ] Contratos (Interfaces) en `Contracts/Commands/` y `Contracts/Queries/`
- [ ] Data Objects en `Data/`
- [ ] Value Objects en `ValueObjects/{ModelName}/` o `ValueObjects/Shared/`
- [ ] Enums en `Enums/`

### Tests

- [ ] Unit tests para todos los Value Objects (cobertura 100%)
- [ ] Unit tests para Enums con lógica de negocio
- [ ] Unit tests para Data objects (si aplica)

### Validaciones

- [ ] PHPStan level 6 sin errores
- [ ] Pint ejecutado y sin cambios pendientes
- [ ] Tests verdes con cobertura del 100%

---

## Comunicación con Otros Agentes

### Salida del Agente A

El Agente A **no implementa nada**, solo define contratos.

**Entrega a Agentes posteriores**:
- Interfaces documentadas con `@throws` y tipos completos
- Value Objects con invariantes validados en constructor
- Data Objects con reglas de validación
- Enums con métodos helper documentados

### Entrada para Agentes B, C, D...

Los agentes que implementan negocio e infraestructura deben:
- ✅ Implementar las interfaces definidas por el Agente A
- ✅ Usar los Data Objects como input/output
- ✅ Usar los Value Objects en modelos Eloquent (mediante Casts)
- ✅ Respetar los Enums como fuente de verdad

---

## Notas de Seguridad

- ❌ **No incluir secretos ni credenciales** en Value Objects o Data Objects
- ✅ **Validar invariantes en constructores** (Value Objects)
- ✅ **Documentar excepciones** que pueden lanzar los contratos
- ✅ **Evitar exponer datos sensibles** en métodos públicos de Data Objects

---

## Supuestos

- Asumo que Laravel Modules está instalado y configurado
- Asumo que Spatie Laravel Data está instalado
- Asumo que Livewire v3 está instalado (para `Wireable`)
- Asumo que el proyecto sigue las convenciones de `conventions.md`
- Asumo que existe `value-objects.md` con documentación completa de VOs

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Value Objects completos**: [`value-objects.md`](../conventions/value-objects.md)
- **Arquitectura de módulos**: [`modules.md`](../conventions/modules.md)
- **Spatie Laravel Data**: https://spatie.be/docs/laravel-data
- **PHP Enums**: https://www.php.net/manual/en/language.enumerations.php

---

**Versión**: 1.0  
**Última actualización**: 2025-12-17  
**Autor**: Alejandro Leone

