---
name: "Agente C — Repositorios, Modelos Eloquent e Infraestructura de Persistencia"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Implementar la capa de persistencia y acceso a datos del módulo"
role: "Define el *cómo* se persiste y recupera la información, nunca el comportamiento del negocio"
dependencies:
  - conventions.md
  - agente-a.md
  - agente-b.md
  - value-objects.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
  - sail
---

# Agente C — Repositorios, Modelos Eloquent e Infraestructura de Persistencia

## Resumen

Agente responsable de **implementar la capa de persistencia del módulo**.
Trabaja sobre los **contratos de repositorio definidos por el Agente A** y utiliza los **Value Objects y Data Objects**
creados por el Agente A.

**Principio fundamental**: Este agente define el *cómo* se persiste y recupera la información, nunca el comportamiento
del negocio.

---

## Alcance Estricto

### ✅ Archivos Permitidos

#### Dentro del módulo

```
Modules/{ModuleName}/
├── Models/                # Modelos Eloquent
├── Repositories/          # Implementaciones de repositorios
├── Casts/                 # Eloquent Casts para Value Objects
├── Factories/             # Factories de modelos
└── Database/
    └── Migrations/        # Migraciones de base de datos
```

#### Tests

```
Modules/{ModuleName}/tests/
├── Unit/
│   ├── Models/           # Tests unitarios de modelos (relaciones, scopes)
│   └── Casts/            # Tests unitarios de Casts
└── Feature/
    └── Repositories/     # Tests de integración de repositorios (con DB)
```

---

### ❌ Archivos Prohibidos

**No crear bajo ningún concepto**:

- ❌ Actions (ya creadas por Agente B)
- ❌ Controllers
- ❌ Filament Resources/Pages
- ❌ Events
- ❌ Listeners
- ❌ Jobs
- ❌ Value Objects nuevos (solo usar los existentes del Agente A)
- ❌ Contratos nuevos (solo implementar los existentes del Agente A)
- ❌ Modificar Actions existentes
- ❌ Lógica de negocio (va en Actions)

---

## Input Disponible para el Agente C

El Agente C recibe como **input obligatorio**:

**Del Agente A**:

- ✅ Contratos de repositorio (`Contracts/Repositories/*`)
- ✅ Data Objects (`Data/*`)
- ✅ Value Objects (`ValueObjects/*`)
- ✅ Enums (`Enums/*`)

**Del Agente B**:

- ✅ Actions implementadas (para entender flujos de negocio)
- ✅ Excepciones de dominio

**Restricciones estrictas**:

- ❌ **No puede modificar contratos del Agente A**
- ❌ **No puede modificar Actions del Agente B**
- ❌ **No puede crear lógica de negocio** (solo persistencia)

---

## Responsabilidades del Agente C

### 1. Crear Modelos Eloquent

**Modelos con tipado fuerte y Value Objects**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Modules\Catalog\Casts\MoneyCast;
use Modules\Catalog\Casts\ProductIdCast;
use Modules\Catalog\Casts\StockCast;
use Modules\Catalog\Database\Factories\ProductFactory;
use Modules\Catalog\Enums\ProductStatus;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

/**
 * @property ProductId $id
 * @property string $sku
 * @property string $name
 * @property string $description
 * @property Money $price
 * @property Stock $stock
 * @property ProductStatus $status
 * @property bool $is_active
 * @property \Carbon\Carbon $created_at
 * @property \Carbon\Carbon $updated_at
 */
final class Product extends Model
{
    use HasFactory;

    protected $table = 'catalog_products';

    protected $keyType = 'string';

    public $incrementing = false;

    protected $fillable = [
        'sku',
        'name',
        'description',
        'price',
        'stock',
        'status',
        'is_active',
    ];

    public function casts(): array
    {
        return [
            'id' => ProductIdCast::class,
            'price' => MoneyCast::class,
            'stock' => StockCast::class,
            'status' => ProductStatus::class,
            'is_active' => 'boolean',
            'created_at' => 'datetime',
            'updated_at' => 'datetime',
        ];
    }

    protected static function newFactory(): ProductFactory
    {
        return ProductFactory::new();
    }

    /**
     * Boot del modelo para generar ID automáticamente.
     */
    protected static function booted(): void
    {
        static::creating(function (self $product): void {
            if ($product->id === null) {
                $product->id = ProductId::generate();
            }
        });
    }
}
```

**Características obligatorias**:

- ✅ `final class extends Model`
- ✅ Docblock con `@property` para todas las propiedades (ayuda a PHPStan)
- ✅ Usar Casts para mapear Value Objects
- ✅ Enum en lugar de strings para status
- ✅ `$keyType`, `$incrementing` configurados si ID no es autoincremental
- ✅ Factory asociada
- ✅ `booted()` para lógica de lifecycle si aplica

---

### 2. Crear Eloquent Casts para Value Objects

**Casts que transforman entre DB y Value Objects**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use InvalidArgumentException;
use Modules\Catalog\ValueObjects\Money;

/**
 * @implements CastsAttributes<Money, Money>
 */
final class MoneyCast implements CastsAttributes
{
    /**
     * @param  array<string, mixed>  $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): Money
    {
        if ($value === null) {
            throw new InvalidArgumentException('El valor de Money no puede ser null');
        }

        $currency = $attributes["{$key}_currency"] ?? 'ARS';

        return new Money(
            cents: (int) $value,
            currency: $currency,
        );
    }

    /**
     * @param  array<string, mixed>  $attributes
     * @return array<string, mixed>
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): array
    {
        if (!$value instanceof Money) {
            throw new InvalidArgumentException('El valor debe ser una instancia de Money');
        }

        return [
            $key => $value->cents,
            "{$key}_currency" => $value->currency,
        ];
    }
}
```

**Características obligatorias**:

- ✅ `final class implements CastsAttributes`
- ✅ Tipado genérico en docblock: `@implements CastsAttributes<Money, Money>`
- ✅ Validación de tipos en `get()` y `set()`
- ✅ Lanzar excepciones si valor no es válido
- ✅ Retornar array en `set()` con todos los campos necesarios

---

### 3. Implementar Repositorios

**Implementaciones de contratos de repositorio**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Repositories;

use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Models\Product;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class ProductRepository implements ProductRepositoryInterface
{
    public function findById(ProductId $id): ?ProductData
    {
        $product = Product::find($id->value());

        if ($product === null) {
            return null;
        }

        return ProductData::from([
            'id' => $product->id,
            'sku' => $product->sku,
            'name' => $product->name,
            'description' => $product->description,
            'price' => $product->price,
            'stock' => $product->stock,
            'isActive' => $product->is_active,
        ]);
    }

    public function findBySku(string $sku): ?ProductData
    {
        $product = Product::where('sku', $sku)->first();

        if ($product === null) {
            return null;
        }

        return $this->toData($product);
    }

    public function existsBySku(string $sku): bool
    {
        return Product::where('sku', $sku)->exists();
    }

    public function create(ProductData $data): ProductId
    {
        $product = Product::create([
            'sku' => $data->sku,
            'name' => $data->name,
            'description' => $data->description,
            'price' => $data->price,
            'stock' => $data->stock,
            'is_active' => $data->isActive,
        ]);

        return $product->id;
    }

    public function update(ProductId $id, ProductData $data): bool
    {
        $product = Product::find($id->value());

        if ($product === null) {
            return false;
        }

        return $product->update([
            'sku' => $data->sku,
            'name' => $data->name,
            'description' => $data->description,
            'price' => $data->price,
            'stock' => $data->stock,
            'is_active' => $data->isActive,
        ]);
    }

    public function delete(ProductId $id): bool
    {
        $product = Product::find($id->value());

        if ($product === null) {
            return false;
        }

        return (bool) $product->delete();
    }

    /**
     * Convierte un modelo Product a ProductData.
     */
    private function toData(Product $product): ProductData
    {
        return ProductData::from([
            'id' => $product->id,
            'sku' => $product->sku,
            'name' => $product->name,
            'description' => $product->description,
            'price' => $product->price,
            'stock' => $product->stock,
            'isActive' => $product->is_active,
        ]);
    }
}
```

**Características obligatorias**:

- ✅ `final readonly class`
- ✅ Implementa el contrato del Agente A
- ✅ **Devuelve Data Objects, nunca modelos Eloquent**
- ✅ Métodos privados auxiliares (`toData()`) para transformaciones
- ✅ No contiene lógica de negocio (solo persistencia)

---

### 4. Crear Migraciones

**Migraciones con índices y tipos correctos**.

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('catalog_products', function (Blueprint $table): void {
            // ID como UUID
            $table->uuid('id')->primary();

            // Campos obligatorios
            $table->string('sku', 100)->unique();
            $table->string('name', 255);
            $table->text('description');

            // Precio (Money Value Object)
            $table->integer('price');
            $table->char('price_currency', 3)->default('ARS');

            // Stock (Stock Value Object)
            $table->integer('stock')->default(0);

            // Status (Enum)
            $table->string('status', 50)->default('active');

            // Estado
            $table->boolean('is_active')->default(true);

            // Timestamps
            $table->timestamps();

            // Índices para optimizar consultas
            $table->index('sku');
            $table->index('status');
            $table->index('is_active');
            $table->index('created_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('catalog_products');
    }
};
```

**Características obligatorias**:

- ✅ Declarar `declare(strict_types=1)`
- ✅ Tipar `up()` y `down()` con `: void`
- ✅ **Incluir índices** para columnas que se usan en WHERE, ORDER BY
- ✅ Valores por defecto cuando aplique
- ✅ Nombres de tablas prefijados con módulo (`catalog_products`)
- ✅ Documentar si afecta producción: `@migration-affects-production`

---

### 5. Crear Factories

**Factories para testing**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Modules\Catalog\Enums\ProductStatus;
use Modules\Catalog\Models\Product;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

/**
 * @extends Factory<Product>
 */
final class ProductFactory extends Factory
{
    protected $model = Product::class;

    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'id' => ProductId::generate(),
            'sku' => strtoupper($this->faker->unique()->bothify('PROD-###??')),
            'name' => $this->faker->words(3, true),
            'description' => $this->faker->paragraph(),
            'price' => new Money(
                cents: $this->faker->numberBetween(100, 100000),
                currency: 'ARS',
            ),
            'stock' => new Stock(value: $this->faker->numberBetween(0, 1000)),
            'status' => ProductStatus::ACTIVE,
            'is_active' => true,
        ];
    }

    /**
     * Estado: producto sin stock.
     */
    public function outOfStock(): self
    {
        return $this->state(fn (array $attributes): array => [
            'stock' => new Stock(value: 0),
        ]);
    }

    /**
     * Estado: producto inactivo.
     */
    public function inactive(): self
    {
        return $this->state(fn (array $attributes): array => [
            'is_active' => false,
            'status' => ProductStatus::INACTIVE,
        ]);
    }
}
```

**Características obligatorias**:

- ✅ `final class extends Factory`
- ✅ Docblock: `@extends Factory<Product>`
- ✅ Usar Value Objects en `definition()`
- ✅ Crear estados (`outOfStock()`, `inactive()`) para casos comunes
- ✅ Tipado: `definition(): array` y `state(fn (array $attributes): array => [])`

---

## Testing Obligatorio

### Unit Tests para Modelos

**Tests de relaciones, scopes, mutators**.

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Enums\ProductStatus;
use Modules\Catalog\Models\Product;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

it('genera un ID automáticamente al crear el producto', function () {
    $product = Product::factory()->make(['id' => null]);

    $product->save();

    expect($product->id)->toBeInstanceOf(ProductId::class);
});

it('castea correctamente el precio a Money', function () {
    $product = Product::factory()->create([
        'price' => new Money(cents: 1500, currency: 'ARS'),
    ]);

    $product->refresh();

    expect($product->price)->toBeInstanceOf(Money::class)
        ->and($product->price->cents)->toBe(1500)
        ->and($product->price->currency)->toBe('ARS');
});

it('castea correctamente el stock a Stock', function () {
    $product = Product::factory()->create([
        'stock' => new Stock(value: 50),
    ]);

    $product->refresh();

    expect($product->stock)->toBeInstanceOf(Stock::class)
        ->and($product->stock->value())->toBe(50);
});

it('castea correctamente el status a enum', function () {
    $product = Product::factory()->create([
        'status' => ProductStatus::ACTIVE,
    ]);

    $product->refresh();

    expect($product->status)->toBeInstanceOf(ProductStatus::class)
        ->and($product->status)->toBe(ProductStatus::ACTIVE);
});
```

---

### Unit Tests para Casts

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Casts\MoneyCast;
use Modules\Catalog\Models\Product;
use Modules\Catalog\ValueObjects\Money;

it('castea correctamente de DB a Money', function () {
    $cast = new MoneyCast();
    $model = new Product();

    $result = $cast->get($model, 'price', 1500, [
        'price' => 1500,
        'price_currency' => 'USD',
    ]);

    expect($result)->toBeInstanceOf(Money::class)
        ->and($result->cents)->toBe(1500)
        ->and($result->currency)->toBe('USD');
});

it('castea correctamente de Money a DB', function () {
    $cast = new MoneyCast();
    $model = new Product();
    $money = new Money(cents: 2000, currency: 'EUR');

    $result = $cast->set($model, 'price', $money, []);

    expect($result)->toBe([
        'price' => 2000,
        'price_currency' => 'EUR',
    ]);
});

it('lanza excepción si el valor en get es null', function () {
    $cast = new MoneyCast();
    $model = new Product();

    $cast->get($model, 'price', null, []);
})->throws(InvalidArgumentException::class, 'El valor de Money no puede ser null');

it('lanza excepción si el valor en set no es Money', function () {
    $cast = new MoneyCast();
    $model = new Product();

    $cast->set($model, 'price', 'invalid', []);
})->throws(InvalidArgumentException::class, 'El valor debe ser una instancia de Money');
```

---

### Feature Tests para Repositorios

**Tests de integración con DB real**.

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Models\Product;
use Modules\Catalog\Repositories\ProductRepository;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

beforeEach(function () {
    $this->repository = new ProductRepository();
});

it('encuentra un producto por ID', function () {
    $product = Product::factory()->create();

    $result = $this->repository->findById($product->id);

    expect($result)->toBeInstanceOf(ProductData::class)
        ->and($result->id->value())->toBe($product->id->value())
        ->and($result->name)->toBe($product->name);
});

it('retorna null si el producto no existe', function () {
    $result = $this->repository->findById(ProductId::generate());

    expect($result)->toBeNull();
});

it('encuentra un producto por SKU', function () {
    $product = Product::factory()->create(['sku' => 'TEST-SKU']);

    $result = $this->repository->findBySku('TEST-SKU');

    expect($result)->toBeInstanceOf(ProductData::class)
        ->and($result->sku)->toBe('TEST-SKU');
});

it('verifica si existe un producto por SKU', function () {
    Product::factory()->create(['sku' => 'EXIST-SKU']);

    expect($this->repository->existsBySku('EXIST-SKU'))->toBeTrue()
        ->and($this->repository->existsBySku('NO-EXIST'))->toBeFalse();
});

it('crea un producto correctamente', function () {
    $data = new ProductData(
        id: null,
        sku: 'NEW-SKU',
        name: 'Nuevo Producto',
        description: 'Descripción',
        price: new Money(cents: 1000, currency: 'ARS'),
        stock: new Stock(value: 10),
        isActive: true,
    );

    $productId = $this->repository->create($data);

    expect($productId)->toBeInstanceOf(ProductId::class);

    $product = Product::find($productId->value());
    expect($product)->not->toBeNull()
        ->and($product->sku)->toBe('NEW-SKU')
        ->and($product->name)->toBe('Nuevo Producto');
});

it('actualiza un producto correctamente', function () {
    $product = Product::factory()->create(['name' => 'Nombre Original']);

    $data = new ProductData(
        id: $product->id,
        sku: $product->sku,
        name: 'Nombre Actualizado',
        description: $product->description,
        price: $product->price,
        stock: $product->stock,
        isActive: $product->is_active,
    );

    $result = $this->repository->update($product->id, $data);

    expect($result)->toBeTrue();

    $product->refresh();
    expect($product->name)->toBe('Nombre Actualizado');
});

it('elimina un producto correctamente', function () {
    $product = Product::factory()->create();

    $result = $this->repository->delete($product->id);

    expect($result)->toBeTrue()
        ->and(Product::find($product->id->value()))->toBeNull();
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

4. ✅ **Migraciones ejecutadas sin errores**
   ```bash
   ./vendor/bin/sail artisan migrate:fresh --seed
   ```

5. ✅ **Tests con cobertura del 100%**
   ```bash
   ./vendor/bin/sail test --filter=Repositories
   ./vendor/bin/sail test --filter=Models
   ./vendor/bin/sail test --filter=Casts
   ```

6. ✅ **Factories funcionan correctamente**
   ```bash
   ./vendor/bin/sail tinker
   >>> Modules\Catalog\Models\Product::factory()->count(10)->create()
   ```

---

## Entregables del Agente C

Al finalizar, el Agente C debe haber creado:

### Archivos de Código

- [ ] Modelos Eloquent en `Models/`
- [ ] Casts en `Casts/`
- [ ] Repositorios en `Repositories/`
- [ ] Migraciones en `Database/Migrations/`
- [ ] Factories en `Database/Factories/`

### Tests

- [ ] Unit tests para modelos (lifecycle, casts, relaciones)
- [ ] Unit tests para Casts (get/set)
- [ ] Feature tests para repositorios (integración con DB)

### Validaciones

- [ ] PHPStan level 6 sin errores
- [ ] Pint ejecutado y sin cambios pendientes
- [ ] Migraciones ejecutadas exitosamente
- [ ] Tests verdes con cobertura del 100%
- [ ] Factories generan datos válidos

---

## Comunicación con Otros Agentes

### Entrada del Agente C

**Recibe del Agente A**:

- ✅ Contratos de repositorio
- ✅ Data Objects
- ✅ Value Objects
- ✅ Enums

**Recibe del Agente B**:

- ✅ Actions (para entender flujo de negocio)
- ✅ Excepciones de dominio

**Restricción**: NO puede modificar nada de los Agentes A y B.

### Salida del Agente C

**Entrega a Agentes posteriores (D, E...)**:

- ✅ Modelos Eloquent funcionando
- ✅ Repositorios implementados
- ✅ Base de datos migrada
- ✅ Factories disponibles para tests

**Los Agentes posteriores pueden**:

- ✅ Crear Controllers que usen los repositorios
- ✅ Crear Filament Resources que usen los modelos
- ✅ Crear Listeners que persistan mediante repositorios
- ✅ Usar Factories en tests de integración

---

## Principios Fundamentales

### 1. Repositorios Devuelven Data Objects, Nunca Eloquent

❌ **Nunca**:

```php
public function findById(ProductId $id): ?Product // ❌ Eloquent
```

✅ **Siempre**:

```php
public function findById(ProductId $id): ?ProductData // ✅ Data Object
```

---

### 2. Modelos Usan Casts para Value Objects

❌ **Nunca**:

```php
// ❌ Acceder directamente a price_cents
$product->price_cents;
```

✅ **Siempre**:

```php
// ✅ Usar Value Object mediante Cast
$product->price->cents; // Money VO
```

---

### 3. Migraciones Incluyen Índices

❌ **Nunca**:

```php
$table->string('sku');
// ❌ Falta índice
```

✅ **Siempre**:

```php
$table->string('sku')->unique();
$table->index('sku'); // ✅ Índice explícito
```

---

### 4. No Hay Lógica de Negocio en Repositorios

❌ **Nunca**:

```php
public function create(ProductData $data): ProductId
{
    // ❌ Validación de negocio en repositorio
    if ($data->price->cents < 0) {
        throw new InvalidPriceException();
    }
    
    return Product::create($data)->id;
}
```

✅ **Siempre**:

```php
public function create(ProductData $data): ProductId
{
    // ✅ Solo persistencia, sin validaciones de negocio
    return Product::create([
        'name' => $data->name,
        'price' => $data->price,
        // ...
    ])->id;
}
```

---

## Notas de Seguridad

- ❌ **No exponer modelos Eloquent fuera del módulo** (usar Data Objects)
- ✅ **Validar tipos en Casts** (lanzar excepciones si valor inválido)
- ✅ **Índices en columnas sensibles** (mejorar performance y evitar queries lentas)
- ✅ **Soft deletes** cuando aplique (no eliminar datos sensibles)

---

## Supuestos

- Asumo que Laravel Modules está instalado y configurado
- Asumo que Laravel Sail está disponible para ejecutar migraciones
- Asumo que los contratos del Agente A están completos
- Asumo que las Actions del Agente B están finalizadas
- Asumo que el proyecto sigue las convenciones de `conventions.md`

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Agente A (Contratos)**: [`agente-a.md`](agente-a.md)
- **Agente B (Actions)**: [`agente-b.md`](agente-b.md)
- **Value Objects completos**: [`value-objects.md`](../conventions/value-objects.md)
- **Arquitectura de módulos**: [`modules.md`](../conventions/modules.md)
- **Eloquent**: https://laravel.com/docs/eloquent
- **Migrations**: https://laravel.com/docs/migrations

---

**Versión**: 1.0  
**Última actualización**: 2025-12-17  
**Autor**: Alejandro Leone

