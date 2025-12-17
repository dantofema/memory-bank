---
name: "Value Objects Examples"
version: "2.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "Detailed examples and patterns for implementing Value Objects in Laravel"
related_docs:
  - requirements: "./conventions.md"
  - testing: "./pest.md"
tags:
  - value-objects
  - domain-driven-design
  - strict-typing
  - livewire
---

# Value Objects: Ejemplos y Patrones

## TL;DR - Resumen Ejecutivo

- **Value Objects (VO)** encapsulan lógica de negocio, garantizan invariantes y aportan semántica al dominio
- **Patrón completo**: VO (`final readonly`, `Wireable`) + Eloquent Cast + Unit/Feature tests
- **Organización**: `app/ValueObjects/{ModelName}/` para VOs específicos, `app/ValueObjects/` para compartidos
- **Criterios de decisión**: consultar tabla comparativa en [`conventions.md`](conventions.md)
- **Tres patrones base**: `Money` (operaciones aritméticas), `PhoneNumber` (normalización), `Stock` (flujo de estado)

## Resumen

Ejemplos completos de implementación de Value Objects en Laravel con Eloquent Casts, Livewire Wireable y testing con
Pest. Complementa los criterios definidos en [`conventions.md`](conventions.md).

---

## Patrón Base: Money

**Complejidad**: 🟢 Básico | **Casos de uso**: Montos monetarios, operaciones aritméticas, comparaciones

### Value Object

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

        if (! in_array($this->currency, ['ARS', 'USD', 'EUR'], true)) {
            throw new InvalidArgumentException("Moneda no soportada: {$this->currency}");
        }
    }

    public static function fromAmount(float $amount, string $currency = 'ARS'): self
    {
        return new self((int) round($amount * 100), $currency);
    }

    public function format(): string
    {
        $symbol = match ($this->currency) {
            'ARS' => '$',
            'USD' => 'U$D',
            'EUR' => '€',
        };

        return $symbol . ' ' . number_format($this->cents / 100, 2, ',', '.');
    }

    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('No se pueden sumar montos de diferentes monedas');
        }

        return new self($this->cents + $other->cents, $this->currency);
    }

    public function multiply(int $factor): self
    {
        return new self($this->cents * $factor, $this->currency);
    }

    public function isGreaterThan(self $other): bool
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException('No se pueden comparar montos de diferentes monedas');
        }

        return $this->cents > $other->cents;
    }

    public function toLivewire(): array
    {
        return [
            'cents' => $this->cents,
            'currency' => $this->currency,
        ];
    }

    public static function fromLivewire($value): self
    {
        return new self($value['cents'], $value['currency']);
    }
}
```

### Eloquent Cast

```php
<?php

declare(strict_types=1);

namespace App\Casts;

use App\ValueObjects\Money;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

final class MoneyCast implements CastsAttributes
{
    /**
     * @param  array<string, mixed>  $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): ?Money
    {
        if ($value === null) {
            return null;
        }

        $currency = $attributes["{$key}_currency"] ?? 'ARS';

        return new Money((int) $value, $currency);
    }

    /**
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if ($value === null) {
            return [
                $key => null,
                "{$key}_currency" => null,
            ];
        }

        if (! $value instanceof Money) {
            throw new \InvalidArgumentException('El valor debe ser una instancia de Money');
        }

        return [
            $key => $value->cents,
            "{$key}_currency" => $value->currency,
        ];
    }
}
```

### Uso en Eloquent Model

```php
<?php

declare(strict_types=1);

namespace App\Models;

use App\Casts\MoneyCast;
use App\ValueObjects\Money;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

final class Product extends Model
{
    use HasFactory;

    protected $fillable = [
        'name',
        'description',
        'price',
        'price_currency',
    ];

    public function casts(): array
    {
        return [
            'price' => MoneyCast::class,
        ];
    }

    // Type hint correcto para PHPStan
    public function getFinalPrice(): Money
    {
        // Lógica de descuentos o promociones
        return $this->price;
    }
}
```

### Migration

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->text('description')->nullable();
            $table->integer('price')->unsigned(); // centavos
            $table->string('price_currency', 3)->default('ARS');
            $table->timestamps();

            $table->index('price');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

### Tests con Pest

```php
<?php

use App\ValueObjects\Money;

describe('Money Value Object', function () {
    it('crea un Money válido', function () {
        $money = new Money(10000, 'ARS');

        expect($money->cents)->toBe(10000)
            ->and($money->currency)->toBe('ARS');
    });

    it('rechaza montos negativos', function () {
        new Money(-100, 'ARS');
    })->throws(InvalidArgumentException::class, 'El monto no puede ser negativo');

    it('rechaza monedas no soportadas', function () {
        new Money(100, 'BRL');
    })->throws(InvalidArgumentException::class, 'Moneda no soportada');

    it('formatea correctamente en ARS', function () {
        $money = new Money(125050, 'ARS');

        expect($money->format())->toBe('$ 1.250,50');
    });

    it('crea desde monto decimal', function () {
        $money = Money::fromAmount(123.45, 'ARS');

        expect($money->cents)->toBe(12345);
    });

    it('suma dos montos de la misma moneda', function () {
        $money1 = new Money(10000, 'ARS');
        $money2 = new Money(5000, 'ARS');

        $result = $money1->add($money2);

        expect($result->cents)->toBe(15000);
    });

    it('rechaza sumar montos de diferentes monedas', function () {
        $money1 = new Money(10000, 'ARS');
        $money2 = new Money(5000, 'USD');

        $money1->add($money2);
    })->throws(InvalidArgumentException::class, 'diferentes monedas');

    it('multiplica correctamente', function () {
        $money = new Money(10000, 'ARS');
        $result = $money->multiply(3);

        expect($result->cents)->toBe(30000);
    });

    it('compara correctamente', function () {
        $money1 = new Money(10000, 'ARS');
        $money2 = new Money(5000, 'ARS');

        expect($money1->isGreaterThan($money2))->toBeTrue()
            ->and($money2->isGreaterThan($money1))->toBeFalse();
    });
});

describe('Money en Eloquent', function () {
    it('persiste y recupera correctamente', function () {
        $product = Product::factory()->create([
            'price' => new Money(50000, 'ARS'),
        ]);

        $product->refresh();

        expect($product->price)->toBeInstanceOf(Money::class)
            ->and($product->price->cents)->toBe(50000)
            ->and($product->price->currency)->toBe('ARS');
    });
});
```

---

## Patrón: PhoneNumber

**Complejidad**: 🟡 Intermedio | **Casos de uso**: Normalización de formato, validación internacional, integración
WhatsApp

### Value Object

```php
<?php

declare(strict_types=1);

namespace App\ValueObjects;

use InvalidArgumentException;
use Livewire\Wireable;

final readonly class PhoneNumber implements Wireable
{
    public function __construct(
        public string $number,
        public string $countryCode = '+54'
    ) {
        $normalized = $this->normalizeNumber($number);

        if (! $this->isValid($normalized)) {
            throw new InvalidArgumentException("Número de teléfono inválido: {$number}");
        }
    }

    private function normalizeNumber(string $number): string
    {
        // Remover espacios, guiones, paréntesis
        return preg_replace('/[^0-9+]/', '', $number);
    }

    private function isValid(string $number): bool
    {
        // Argentina: 10-13 dígitos (con/sin código de país)
        $length = strlen($number);

        return $length >= 10 && $length <= 13;
    }

    public function format(): string
    {
        // Formato: +54 9 11 1234-5678
        $clean = str_replace($this->countryCode, '', $this->number);
        $clean = ltrim($clean, '0');

        if (strlen($clean) === 10) {
            return sprintf(
                '%s 9 %s %s-%s',
                $this->countryCode,
                substr($clean, 0, 2),
                substr($clean, 2, 4),
                substr($clean, 6, 4)
            );
        }

        return $this->countryCode . ' ' . $this->number;
    }

    public function getWhatsAppUrl(string $message = ''): string
    {
        $number = str_replace(['+', ' ', '-'], '', $this->format());

        return sprintf(
            'https://wa.me/%s?text=%s',
            $number,
            urlencode($message)
        );
    }

    public function toLivewire(): array
    {
        return [
            'number' => $this->number,
            'countryCode' => $this->countryCode,
        ];
    }

    public static function fromLivewire($value): self
    {
        return new self($value['number'], $value['countryCode']);
    }
}
```

---

## Patrón: Stock

**Complejidad**: 🟡 Intermedio | **Casos de uso**: Gestión de inventario, reservas, transiciones de estado con validación

### Value Object

```php
<?php

declare(strict_types=1);

namespace App\ValueObjects\Product;

use InvalidArgumentException;
use Livewire\Wireable;

final readonly class Stock implements Wireable
{
    public function __construct(
        public int $available,
        public int $reserved = 0
    ) {
        if ($this->available < 0) {
            throw new InvalidArgumentException('Stock disponible no puede ser negativo');
        }

        if ($this->reserved < 0) {
            throw new InvalidArgumentException('Stock reservado no puede ser negativo');
        }
    }

    public function total(): int
    {
        return $this->available + $this->reserved;
    }

    public function canReserve(int $quantity): bool
    {
        return $this->available >= $quantity;
    }

    public function reserve(int $quantity): self
    {
        if (! $this->canReserve($quantity)) {
            throw new InvalidArgumentException("Stock insuficiente. Disponible: {$this->available}, solicitado: {$quantity}");
        }

        return new self(
            $this->available - $quantity,
            $this->reserved + $quantity
        );
    }

    public function confirmReservation(int $quantity): self
    {
        if ($this->reserved < $quantity) {
            throw new InvalidArgumentException("No hay suficiente stock reservado para confirmar");
        }

        return new self(
            $this->available,
            $this->reserved - $quantity
        );
    }

    public function cancelReservation(int $quantity): self
    {
        if ($this->reserved < $quantity) {
            throw new InvalidArgumentException("No hay suficiente stock reservado para cancelar");
        }

        return new self(
            $this->available + $quantity,
            $this->reserved - $quantity
        );
    }

    public function toLivewire(): array
    {
        return [
            'available' => $this->available,
            'reserved' => $this->reserved,
        ];
    }

    public static function fromLivewire($value): self
    {
        return new self($value['available'], $value['reserved']);
    }
}
```

---

## Anti-Patrones: Cuándo NO Usar Value Objects

### ❌ Over-Engineering: VOs para datos simples sin lógica

```php
// ❌ MAL: No aporta valor
final readonly class ProductName
{
    public function __construct(public string $value) {}
}

// ✅ BIEN: String directo con validación en FormRequest
class CreateProductRequest extends FormRequest
{
    public function rules(): array
    {
        return ['name' => 'required|string|max:255'];
    }
}
```

### ❌ Dependencias externas dentro del VO

```php
// ❌ MAL: VO no debe depender de servicios externos
final readonly class ProductPrice
{
    public function __construct(
        private CurrencyService $currencyService, // ❌ Inyección de dependencia
        public int $cents
    ) {}
}

// ✅ BIEN: VO solo con datos y lógica pura
final readonly class Money
{
    public function __construct(
        public int $cents,
        public string $currency = 'ARS'
    ) {}
    
    // Conversión se hace fuera del VO
}
```

### ❌ Mutabilidad: VOs que cambian su estado

```php
// ❌ MAL: VO mutable
class Money
{
    public function __construct(public int $cents) {}
    
    public function add(int $amount): void // ❌ Modifica estado
    {
        $this->cents += $amount;
    }
}

// ✅ BIEN: VO inmutable retorna nueva instancia
final readonly class Money
{
    public function __construct(public int $cents) {}
    
    public function add(self $other): self // ✅ Nueva instancia
    {
        return new self($this->cents + $other->cents);
    }
}
```

### ❌ Validación débil o inconsistente

```php
// ❌ MAL: Permite crear VOs inválidos
final readonly class Email
{
    public function __construct(public string $value) {} // ❌ Sin validación
}

// ✅ BIEN: Garantiza estado válido siempre
final readonly class Email
{
    public function __construct(public string $value)
    {
        if (! filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException("Email inválido: {$value}");
        }
    }
}
```

---

## Checklist de Implementación

Al crear un nuevo Value Object, asegurate de:

- [ ] Clase `final readonly`
- [ ] Constructor valida invariantes (lanza excepciones)
- [ ] Implementa `Wireable` si se usa en Livewire
- [ ] Métodos de dominio en lugar de getters simples
- [ ] Eloquent Cast si se persiste en base de datos
- [ ] Unit tests para validaciones y comportamiento
- [ ] Feature tests si interactúa con Eloquent
- [ ] Documentar en tabla comparativa de `requirements.md` si es ejemplo relevante

---

## Notas de Seguridad

- Validar en constructor: nunca permitir estado inválido
- Sanitizar inputs en normalización (ej: `PhoneNumber`)
- No exponer información sensible en métodos `format()` o `toLivewire()`
- En Cast de Eloquent: validar tipos antes de castear

---

## Referencias

- [Spatie Laravel Data](https://spatie.be/docs/laravel-data)
- [Eloquent Casts](https://laravel.com/docs/eloquent-mutators#custom-casts)
- [Livewire Wireable](https://livewire.laravel.com/docs/properties#binding-directly-to-model-properties)

