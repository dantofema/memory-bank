---
name: "Agente B — Actions y Tests Unitarios"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Implementar casos de uso del módulo mediante Actions"
role: "Define el comportamiento del negocio, no la infraestructura ni la entrada/salida"
dependencies:
  - conventions.md
  - agente-a.md
  - value-objects.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
---

# Agente B — Actions y Tests Unitarios

## Resumen

Agente responsable de **implementar los casos de uso del módulo**.
Trabaja exclusivamente sobre los **contratos, Data y Value Objects definidos por el Agente A**.

**Principio fundamental**: Este agente define el *comportamiento del negocio*, no la infraestructura ni la entrada/salida.

---

## Input del Agente

Este agente es **agnóstico del proyecto** y recibe contexto de:

- **Task específica**: define QUÉ Actions implementar y sus reglas de negocio
- **Project definition**: define el contexto del negocio y requisitos funcionales
- **Domain definition**: define reglas de negocio específicas del dominio
- **Output del Agente A**: contratos, Data Objects y Value Objects a implementar

El agente proporciona la **metodología** (el CÓMO implementar Actions).  
La task proporciona el **contexto** (el QUÉ implementar específicamente y las reglas de negocio).

### Ejemplos en este documento

Los ejemplos usan "Catalog/Product" como **placeholder genérico** para ilustrar la metodología.  
En tu task, **reemplázalos con las entidades de tu dominio específico** (ej: Order, Payment, User).

**Ver**: `laravel/AGENTS_ARCHITECTURE.md` para entender el sistema completo.

---

## Alcance Estricto

### ✅ Archivos Permitidos

#### Dentro del módulo

```
Modules/{ModuleName}/
├── Actions/
│   ├── Commands/          # Implementaciones de Commands (modifican estado)
│   ├── Queries/           # Implementaciones de Queries (solo lectura)
│   └── Internal/          # Actions auxiliares (privadas al módulo)
└── Exceptions/            # Excepciones de dominio del módulo
```

#### Tests

```
Modules/{ModuleName}/tests/Unit/Actions/
├── Commands/              # Tests de Actions Commands
├── Queries/               # Tests de Actions Queries
└── Internal/              # Tests de Actions auxiliares
```

---

### ❌ Archivos Prohibidos

**No crear bajo ningún concepto**:

- ❌ Models (Eloquent)
- ❌ Repositories concretos
- ❌ Controllers
- ❌ Filament Resources/Pages
- ❌ Events
- ❌ Migrations
- ❌ Factories
- ❌ Queries SQL directas
- ❌ Acceso directo a `DB` o `Schema`
- ❌ Código HTTP (Request, Response)
- ❌ Value Objects nuevos (solo usar los existentes del Agente A)
- ❌ Contratos nuevos (solo implementar los existentes del Agente A)
- ❌ Casts

---

## Input Disponible para el Agente B

El Agente B recibe como **input obligatorio** del Agente A:

- ✅ Contratos (`Contracts/Commands/*`, `Contracts/Queries/*`)
- ✅ Data Objects (`Data/*`)
- ✅ Enums (`Enums/*`)
- ✅ Value Objects (`ValueObjects/*`)
- ✅ Tests existentes de Value Objects y Enums

**Restricciones estrictas**:
- ❌ **No puede modificar nada del Agente A**
- ❌ **No puede redefinir contratos**
- ❌ **No puede crear nuevos Value Objects**

---

## Responsabilidades del Agente B

### 1. Implementar Actions para Commands

**Actions que modifican estado del sistema**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Actions\Commands;

use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Enums\ProductStatus;
use Modules\Catalog\Exceptions\ProductAlreadyExistsException;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class CreateProductAction implements CreateProductInterface
{
    public function __construct(
        private ProductRepositoryInterface $productRepository,
    ) {}

    /**
     * @throws ProductAlreadyExistsException
     */
    public function execute(ProductData $data): ProductId
    {
        // Validación de negocio: el SKU no debe existir
        if ($this->productRepository->existsBySku($data->sku)) {
            throw new ProductAlreadyExistsException(
                sprintf('El producto con SKU "%s" ya existe', $data->sku)
            );
        }

        // Validación de negocio: el stock inicial debe ser no negativo
        if ($data->stock->value() < 0) {
            throw new InvalidArgumentException('El stock inicial no puede ser negativo');
        }

        // Crear el producto a través del repositorio
        $productId = $this->productRepository->create($data);

        return $productId;
    }
}
```

**Características obligatorias**:
- ✅ `final readonly class`
- ✅ Implementa el contrato correspondiente del Agente A
- ✅ Un solo método público: `execute()`
- ✅ Inyección de dependencias mediante constructor (interfaces)
- ✅ Lanza excepciones de dominio documentadas
- ✅ No accede directamente a Eloquent ni DB

---

### 2. Implementar Actions para Queries

**Actions de solo lectura**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Actions\Queries;

use Modules\Catalog\Contracts\Queries\GetProductInterface;
use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Exceptions\ProductNotFoundException;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class GetProductAction implements GetProductInterface
{
    public function __construct(
        private ProductRepositoryInterface $productRepository,
    ) {}

    /**
     * @throws ProductNotFoundException
     */
    public function execute(ProductId $id): ProductData
    {
        $product = $this->productRepository->findById($id);

        if ($product === null) {
            throw new ProductNotFoundException(
                sprintf('El producto con ID "%s" no fue encontrado', $id->value())
            );
        }

        return $product;
    }
}
```

**Características obligatorias**:
- ✅ `final readonly class`
- ✅ Implementa el contrato Query del Agente A
- ✅ Un solo método público: `execute()`
- ✅ No modifica estado
- ✅ Devuelve Data Objects o Value Objects

---

### 3. Crear Actions Auxiliares (Internal)

**Actions privadas del módulo** que no están expuestas como contratos públicos.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Actions\Internal;

use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class CalculateDiscountedPriceAction
{
    public function __construct(
        private ProductRepositoryInterface $productRepository,
    ) {}

    /**
     * Calcula el precio con descuento aplicado.
     */
    public function execute(ProductId $productId, int $discountPercentage): Money
    {
        $product = $this->productRepository->findById($productId);

        if ($product === null) {
            throw new InvalidArgumentException('Producto no encontrado');
        }

        $discountAmount = (int) ($product->price->cents * $discountPercentage / 100);
        $discountedPrice = $product->price->cents - $discountAmount;

        return new Money(cents: $discountedPrice, currency: $product->price->currency);
    }
}
```

**Características obligatorias**:
- ✅ `final readonly class`
- ✅ **No implementa contrato público** (es interna al módulo)
- ✅ Puede ser llamada por otras Actions del mismo módulo
- ✅ Un solo método público
- ✅ Encapsula lógica de negocio auxiliar

---

### 4. Crear Excepciones de Dominio

**Excepciones específicas del módulo**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Exceptions;

use Exception;

final class ProductAlreadyExistsException extends Exception
{
    public static function forSku(string $sku): self
    {
        return new self(sprintf('El producto con SKU "%s" ya existe', $sku));
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Exceptions;

use Exception;
use Modules\Catalog\ValueObjects\ProductId;

final class ProductNotFoundException extends Exception
{
    public static function forId(ProductId $id): self
    {
        return new self(sprintf('El producto con ID "%s" no fue encontrado', $id->value()));
    }
}
```

**Características obligatorias**:
- ✅ `final class extends Exception`
- ✅ Named constructors semánticos (`forSku()`, `forId()`)
- ✅ Mensajes descriptivos en español
- ✅ Tipado fuerte en parámetros

---

## Testing Obligatorio

### Unit Tests para Actions Commands

**Mock de repositorios y validación de comportamiento**.

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Actions\Commands\CreateProductAction;
use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Exceptions\ProductAlreadyExistsException;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

beforeEach(function () {
    $this->productRepository = Mockery::mock(ProductRepositoryInterface::class);
    $this->action = new CreateProductAction($this->productRepository);
});

it('crea un producto correctamente', function () {
    $data = new ProductData(
        id: null,
        sku: 'PROD-001',
        name: 'Producto Test',
        description: 'Descripción del producto',
        price: new Money(cents: 1500, currency: 'ARS'),
        stock: new Stock(value: 10),
        isActive: true,
    );

    $productId = ProductId::generate();

    $this->productRepository
        ->shouldReceive('existsBySku')
        ->once()
        ->with('PROD-001')
        ->andReturn(false);

    $this->productRepository
        ->shouldReceive('create')
        ->once()
        ->with($data)
        ->andReturn($productId);

    $result = $this->action->execute($data);

    expect($result)->toBeInstanceOf(ProductId::class)
        ->and($result->value())->toBe($productId->value());
});

it('lanza excepción si el SKU ya existe', function () {
    $data = new ProductData(
        id: null,
        sku: 'PROD-001',
        name: 'Producto Test',
        description: 'Descripción',
        price: new Money(cents: 1500, currency: 'ARS'),
        stock: new Stock(value: 10),
        isActive: true,
    );

    $this->productRepository
        ->shouldReceive('existsBySku')
        ->once()
        ->with('PROD-001')
        ->andReturn(true);

    $this->action->execute($data);
})->throws(ProductAlreadyExistsException::class, 'El producto con SKU "PROD-001" ya existe');

it('lanza excepción si el stock inicial es negativo', function () {
    $data = new ProductData(
        id: null,
        sku: 'PROD-002',
        name: 'Producto Test',
        description: 'Descripción',
        price: new Money(cents: 1500, currency: 'ARS'),
        stock: new Stock(value: -5),
        isActive: true,
    );

    $this->productRepository
        ->shouldReceive('existsBySku')
        ->once()
        ->with('PROD-002')
        ->andReturn(false);

    $this->action->execute($data);
})->throws(InvalidArgumentException::class);
```

---

### Unit Tests para Actions Queries

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Actions\Queries\GetProductAction;
use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Exceptions\ProductNotFoundException;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

beforeEach(function () {
    $this->productRepository = Mockery::mock(ProductRepositoryInterface::class);
    $this->action = new GetProductAction($this->productRepository);
});

it('obtiene un producto por ID', function () {
    $productId = ProductId::generate();

    $productData = new ProductData(
        id: $productId,
        sku: 'PROD-001',
        name: 'Producto Test',
        description: 'Descripción',
        price: new Money(cents: 1500, currency: 'ARS'),
        stock: new Stock(value: 10),
        isActive: true,
    );

    $this->productRepository
        ->shouldReceive('findById')
        ->once()
        ->with($productId)
        ->andReturn($productData);

    $result = $this->action->execute($productId);

    expect($result)->toBeInstanceOf(ProductData::class)
        ->and($result->name)->toBe('Producto Test')
        ->and($result->sku)->toBe('PROD-001');
});

it('lanza excepción si el producto no existe', function () {
    $productId = ProductId::generate();

    $this->productRepository
        ->shouldReceive('findById')
        ->once()
        ->with($productId)
        ->andReturn(null);

    $this->action->execute($productId);
})->throws(ProductNotFoundException::class);
```

---

### Unit Tests para Actions Auxiliares (Internal)

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Actions\Internal\CalculateDiscountedPriceAction;
use Modules\Catalog\Contracts\Repositories\ProductRepositoryInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\ProductId;
use Modules\Catalog\ValueObjects\Stock;

beforeEach(function () {
    $this->productRepository = Mockery::mock(ProductRepositoryInterface::class);
    $this->action = new CalculateDiscountedPriceAction($this->productRepository);
});

it('calcula el precio con descuento correctamente', function () {
    $productId = ProductId::generate();

    $productData = new ProductData(
        id: $productId,
        sku: 'PROD-001',
        name: 'Producto Test',
        description: 'Descripción',
        price: new Money(cents: 1000, currency: 'ARS'),
        stock: new Stock(value: 10),
        isActive: true,
    );

    $this->productRepository
        ->shouldReceive('findById')
        ->once()
        ->with($productId)
        ->andReturn($productData);

    $result = $this->action->execute($productId, discountPercentage: 20);

    expect($result)->toBeInstanceOf(Money::class)
        ->and($result->cents)->toBe(800)
        ->and($result->currency)->toBe('ARS');
});

it('lanza excepción si el producto no existe', function () {
    $productId = ProductId::generate();

    $this->productRepository
        ->shouldReceive('findById')
        ->once()
        ->with($productId)
        ->andReturn(null);

    $this->action->execute($productId, discountPercentage: 10);
})->throws(InvalidArgumentException::class, 'Producto no encontrado');
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
   ./vendor/bin/sail test --filter=Actions
   ```

5. ✅ **Tipado fuerte**: sin `mixed`, `array` o tipos genéricos en signatures públicas

6. ✅ **Inyección de dependencias**: todas las dependencias mediante constructor

7. ✅ **Un solo método público** por Action: `execute()`

8. ✅ **Mock de repositorios**: nunca acceder directamente a DB en tests unitarios

---

## Entregables del Agente B

Al finalizar, el Agente B debe haber creado:

### Archivos de Código

- [ ] Actions Commands en `Actions/Commands/`
- [ ] Actions Queries en `Actions/Queries/`
- [ ] Actions auxiliares en `Actions/Internal/` (si aplica)
- [ ] Excepciones de dominio en `Exceptions/`

### Tests

- [ ] Unit tests para todos los Actions Commands (cobertura 100%)
- [ ] Unit tests para todos los Actions Queries (cobertura 100%)
- [ ] Unit tests para Actions auxiliares (cobertura 100%)
- [ ] Tests de excepciones de dominio

### Validaciones

- [ ] PHPStan level 6 sin errores
- [ ] Pint ejecutado y sin cambios pendientes
- [ ] Tests verdes con cobertura del 100%
- [ ] Mockery utilizado correctamente para simular repositorios

---

## Comunicación con Otros Agentes

### Entrada del Agente B

**Recibe del Agente A**:
- ✅ Contratos (Interfaces)
- ✅ Data Objects
- ✅ Value Objects
- ✅ Enums

**Restricción**: NO puede modificar nada del Agente A.

### Salida del Agente B

**Entrega a Agentes posteriores (C, D...)**:
- ✅ Actions implementadas (Commands, Queries, Internal)
- ✅ Excepciones de dominio documentadas
- ✅ Tests unitarios con mocks

**Los Agentes posteriores pueden**:
- ✅ Implementar los repositorios mockados
- ✅ Crear modelos Eloquent
- ✅ Crear infraestructura (Controllers, Filament, Listeners)
- ✅ Usar las Actions desde puntos de entrada

---

## Principios Fundamentales

### 1. Actions Solo Reciben y Devuelven Objetos

❌ **Nunca**:
```php
public function execute(array $data): array // ❌
```

✅ **Siempre**:
```php
public function execute(ProductData $data): ProductId // ✅
```

---

### 2. Actions No Acceden Directamente a Infraestructura

❌ **Nunca**:
```php
// ❌ Acceso directo a Eloquent
$product = Product::where('sku', $sku)->first();

// ❌ Acceso directo a DB
DB::table('products')->insert($data);
```

✅ **Siempre**:
```php
// ✅ Mediante contrato de repositorio
$product = $this->productRepository->findBySku($sku);
```

---

### 3. Actions Lanzan Excepciones de Dominio

❌ **Nunca**:
```php
if (!$product) {
    return null; // ❌
}
```

✅ **Siempre**:
```php
if (!$product) {
    throw ProductNotFoundException::forId($id); // ✅
}
```

---

### 4. Actions Tienen Un Solo Método Público

❌ **Nunca**:
```php
final class CreateProductAction
{
    public function execute(ProductData $data): ProductId { }
    public function validate(ProductData $data): bool { } // ❌
}
```

✅ **Siempre**:
```php
final class CreateProductAction
{
    public function execute(ProductData $data): ProductId { } // ✅
}
```

---

### 5. Tests Unitarios Usan Mocks, No DB

❌ **Nunca**:
```php
it('crea un producto', function () {
    $product = Product::factory()->create(); // ❌ No es unit test
});
```

✅ **Siempre**:
```php
it('crea un producto', function () {
    $this->productRepository
        ->shouldReceive('create')
        ->once()
        ->andReturn($productId); // ✅ Mock
    
    $result = $this->action->execute($data);
});
```

---

## Notas de Seguridad

- ❌ **No exponer información sensible** en mensajes de excepción
- ✅ **Validar invariantes de negocio** antes de delegar al repositorio
- ✅ **Documentar excepciones** con `@throws` en docblocks
- ✅ **Sanitizar inputs** que provengan de usuarios (aunque debe hacerse en capas superiores)

---

## Supuestos

- Asumo que Laravel Modules está instalado y configurado
- Asumo que Mockery está disponible para tests
- Asumo que los contratos del Agente A están completos y finalizados
- Asumo que los repositorios serán implementados por un Agente posterior
- Asumo que el proyecto sigue las convenciones de `conventions.md`

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Agente A (Contratos)**: [`agente-a.md`](agente-a.md)
- **Value Objects completos**: [`value-objects.md`](../conventions/value-objects.md)
- **Arquitectura de módulos**: [`modules.md`](../conventions/modules.md)
- **Mockery**: https://github.com/mockery/mockery
- **Pest**: https://pestphp.com/

---

**Versión**: 1.0  
**Última actualización**: 2025-12-17  
**Autor**: Alejandro Leone

