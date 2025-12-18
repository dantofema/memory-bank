---
name: "Catalog Module - Class Diagram"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "Detailed class diagram for Catalog module with Value Objects and Services"
module: "Catalog"
related_docs:
    - "project_definition.md"
    - "module_diagram.md"
conventions: "laravel/conventions.md"
---

# Diagrama de Clases - Módulo Catalog

## Descripción General

El módulo **Catalog** es responsable de la gestión completa de productos, categorías, variantes y promociones.
Implementa Value Objects para propiedades complejas siguiendo las convenciones de Laravel definidas en el proyecto.

**Principios aplicados:**

- Clases `final` sin métodos `protected`
- Value Objects con `Wireable` para Livewire
- Casts de Eloquent para mapear Value Objects
- Services con un solo método público
- Tipado fuerte en toda la implementación

---

## Diagrama de Clases

```mermaid
classDiagram
%% ========================================
%% MODELS (Eloquent)
%% ========================================
    class Product {
        <<final>>
        +int id
        +string name
        +string description
        +PriceValueObject price
        +StockValueObject stock
        +bool is_active
        +int category_id
        +timestamps
        +SoftDeletes
        --
        +category() BelongsTo~Category~
        +variants() HasMany~ProductVariant~
        +promotions() BelongsToMany~Promotion~
        +orderItems() HasMany~OrderItem~
        --
        +getCasts() array
    }

    class ProductVariant {
        <<final>>
        +int id
        +int product_id
        +string name
        +VariantAttributesValueObject attributes
        +PriceValueObject price
        +StockValueObject stock
        +bool is_active
        +timestamps
        --
        +product() BelongsTo~Product~
        +orderItems() HasMany~OrderItem~
        --
        +getCasts() array
    }

    class Category {
        <<final>>
        +int id
        +string name
        +string slug
        +string description
        +int display_order
        +bool is_active
        +timestamps
        --
        +products() HasMany~Product~
    }

    class Promotion {
        <<final>>
        +int id
        +string name
        +PromotionTypeEnum type
        +DiscountValueObject discount
        +PromotionPeriodValueObject period
        +bool is_active
        +timestamps
        --
        +products() BelongsToMany~Product~
        +isValid() bool
        +apply(Money price) Money
        --
        +getCasts() array
    }

%% ========================================
%% VALUE OBJECTS (Product)
%% ========================================

    class PriceValueObject {
        <<final>>
        <<readonly>>
        <<Wireable>>
        +int cents
        +string currency
        --
        +__construct(int cents, string currency)
        +toMoney() Money
        +format() string
        +isZero() bool
        +isPositive() bool
        +toLivewire() array
        +fromLivewire(value) self
    }

    class StockValueObject {
        <<final>>
        <<readonly>>
        <<Wireable>>
        +int available
        +int reserved
        --
        +__construct(int available, int reserved)
        +total() int
        +canReserve(int quantity) bool
        +reserve(int quantity) self
        +release(int quantity) self
        +isInStock() bool
        +toLivewire() array
        +fromLivewire(value) self
    }

    class VariantAttributesValueObject {
        <<finalreadonly>>
        <<Wireable>>
        +array~string, string~ attributes
        --
        +__construct(array attributes)
        +get(string key) ?string
        +has(string key) bool
        +toArray() array
        +toString() string
        +toLivewire() array
        +fromLivewire(value) self
    }

%% ========================================
%% VALUE OBJECTS (Promotion)
%% ========================================

    class DiscountValueObject {
        <<finalreadonly>>
        <<Wireable>>
        +PromotionTypeEnum type
        +int value
        --
        +__construct(PromotionTypeEnum type, int value)
        +isPercentage() bool
        +isFixedPrice() bool
        +calculate(Money originalPrice) Money
        +toLivewire() array
        +fromLivewire(value) self
    }

    class PromotionPeriodValueObject {
        <<finalreadonly>>
        <<Wireable>>
        +Carbon start
        +Carbon end
        --
        +__construct(Carbon start, Carbon end)
        +isActive() bool
        +isActiveAt(Carbon date) bool
        +hasStarted() bool
        +hasEnded() bool
        +durationInDays() int
        +toLivewire() array
        +fromLivewire(value) self
    }

%% ========================================
%% SHARED VALUE OBJECTS
%% ========================================

    class Money {
        <<finalreadonly>>
        <<Wireable>>
        +int cents
        +string currency
        --
        +__construct(int cents, string currency)
        +add(Money other) Money
        +subtract(Money other) Money
        +multiply(float factor) Money
        +applyPercentage(int percentage) Money
        +format() string
        +isZero() bool
        +isPositive() bool
        +equals(Money other) bool
        +toLivewire() array
        +fromLivewire(value) self
    }

    class Quantity {
        <<finalreadonly>>
        <<Wireable>>
        +int value
        --
        +__construct(int value)
        +add(Quantity other) Quantity
        +subtract(Quantity other) Quantity
        +isZero() bool
        +isPositive() bool
        +greaterThan(Quantity other) bool
        +lessThan(Quantity other) bool
        +equals(Quantity other) bool
        +toLivewire() array
        +fromLivewire(value) self
    }

%% ========================================
%% ENUMS
%% ========================================

    class PromotionTypeEnum {
        <<enum>>
        PERCENTAGE
        FIXED_PRICE
        --
        +label() string
        +description() string
    }

%% ========================================
%% CASTS (Eloquent)
%% ========================================

    class PriceValueObjectCast {
        <<final>>
        +get(Model model, string key, mixed value, array attributes) ?PriceValueObject
        +set(Model model, string key, mixed value, array attributes) array
    }

    class StockValueObjectCast {
        <<final>>
        +get(Model model, string key, mixed value, array attributes) ?StockValueObject
        +set(Model model, string key, mixed value, array attributes) array
    }

    class VariantAttributesValueObjectCast {
        <<final>>
        +get(Model model, string key, mixed value, array attributes) ?VariantAttributesValueObject
        +set(Model model, string key, mixed value, array attributes) array
    }

    class DiscountValueObjectCast {
        <<final>>
        +get(Model model, string key, mixed value, array attributes) ?DiscountValueObject
        +set(Model model, string key, mixed value, array attributes) array
    }

    class PromotionPeriodValueObjectCast {
        <<final>>
        +get(Model model, string key, mixed value, array attributes) ?PromotionPeriodValueObject
        +set(Model model, string key, mixed value, array attributes) array
    }

%% ========================================
%% SERVICES
%% ========================================

    class CheckProductAvailabilityService {
        <<final>>
        +__construct(ProductRepository repo)
        +execute(CheckProductAvailabilityData data) ProductAvailabilityResult
    }

    class ApplyPromotionService {
        <<final>>
        +__construct(PromotionRepository repo)
        +execute(ApplyPromotionData data) AppliedPromotionResult
    }

    class ReserveStockService {
        <<final>>
        +__construct(ProductRepository repo, ProductVariantRepository variantRepo)
        +execute(ReserveStockData data) StockReservationResult
    }

%% ========================================
%% DATA OBJECTS (Spatie Laravel Data)
%% ========================================

    class CheckProductAvailabilityData {
        <<finalreadonly>>
        +int productId
        +?int variantId
        +int quantity
        --
        +fromRequest(Request request) self
    }

    class ProductAvailabilityResult {
        <<finalreadonly>>
        +bool isAvailable
        +int availableStock
        +?string message
    }

    class ApplyPromotionData {
        <<finalreadonly>>
        +int productId
        +Money originalPrice
        +Carbon checkDate
    }

    class AppliedPromotionResult {
        <<finalreadonly>>
        +Money finalPrice
        +?Promotion appliedPromotion
        +bool hasDiscount
        +Money discountAmount
    }

    class ReserveStockData {
        <<finalreadonly>>
        +int productId
        +?int variantId
        +int quantity
    }

    class StockReservationResult {
        <<finalreadonly>>
        +bool success
        +int reservedQuantity
        +int remainingStock
        +?string errorMessage
    }

%% ========================================
%% INTERFACES (Contracts)
%% ========================================

    class CheckProductAvailabilityInterface {
        <<interface>>
        +check(CheckProductAvailabilityData data) ProductAvailabilityResult
    }

    class ApplyPromotionInterface {
        <<interface>>
        +apply(ApplyPromotionData data) AppliedPromotionResult
    }

    class ReserveStockInterface {
        <<interface>>
        +reserve(ReserveStockData data) StockReservationResult
    }

%% ========================================
%% REPOSITORIES
%% ========================================

    class ProductRepository {
        <<final>>
        +findById(int id) ?Product
        +findByIdWithVariants(int id) ?Product
        +findActiveProducts() Collection~Product~
        +updateStock(int productId, StockValueObject stock) bool
        +reserveStock(int productId, int quantity) bool
    }

    class ProductVariantRepository {
        <<final>>
        +findById(int id) ?ProductVariant
        +findByProductId(int productId) Collection~ProductVariant~
        +updateStock(int variantId, StockValueObject stock) bool
        +reserveStock(int variantId, int quantity) bool
    }

    class PromotionRepository {
        <<final>>
        +findActiveForProduct(int productId, Carbon date) ?Promotion
        +findActivePromotions() Collection~Promotion~
    }

    class CategoryRepository {
        <<final>>
        +findById(int id) ?Category
        +findActiveCategories() Collection~Category~
        +findBySlug(string slug) ?Category
    }

%% ========================================
%% FACTORIES
%% ========================================

    class ProductFactory {
        <<Factory>>
        +definition() array
        +withCategory(Category category) self
        +active() self
        +inactive() self
        +withStock(int stock) self
    }

    class ProductVariantFactory {
        <<Factory>>
        +definition() array
        +forProduct(Product product) self
        +withStock(int stock) self
    }

    class CategoryFactory {
        <<Factory>>
        +definition() array
        +active() self
    }

    class PromotionFactory {
        <<Factory>>
        +definition() array
        +percentage(int value) self
        +fixedPrice(int cents) self
        +active() self
        +forPeriod(Carbon start, Carbon end) self
    }

%% ========================================
%% RELATIONSHIPS
%% ========================================
    Product "1" --> "0..1" Category: belongs to
    Product "1" --> "0..*" ProductVariant: has many
    Product "0..*" --> "0..*" Promotion: belongs to many
    ProductVariant "1" --> "1" Product: belongs to
%% Value Objects usage
    Product ..> PriceValueObject: uses
    Product ..> StockValueObject: uses
    ProductVariant ..> PriceValueObject: uses
    ProductVariant ..> StockValueObject: uses
    ProductVariant ..> VariantAttributesValueObject: uses
    Promotion ..> DiscountValueObject: uses
    Promotion ..> PromotionPeriodValueObject: uses
    Promotion ..> PromotionTypeEnum: uses
%% Shared Value Objects
    PriceValueObject ..> Money: converts to
    DiscountValueObject ..> Money: calculates with
    StockValueObject ..> Quantity: uses concept
%% Casts relationship
    Product ..> PriceValueObjectCast: casts with
    Product ..> StockValueObjectCast: casts with
    ProductVariant ..> PriceValueObjectCast: casts with
    ProductVariant ..> StockValueObjectCast: casts with
    ProductVariant ..> VariantAttributesValueObjectCast: casts with
    Promotion ..> DiscountValueObjectCast: casts with
    Promotion ..> PromotionPeriodValueObjectCast: casts with
%% Services relationships
    CheckProductAvailabilityService ..|> CheckProductAvailabilityInterface: implements
    ApplyPromotionService ..|> ApplyPromotionInterface: implements
    ReserveStockService ..|> ReserveStockInterface: implements
    CheckProductAvailabilityService --> ProductRepository: uses
    ApplyPromotionService --> PromotionRepository: uses
    ReserveStockService --> ProductRepository: uses
    ReserveStockService --> ProductVariantRepository: uses
    CheckProductAvailabilityService ..> CheckProductAvailabilityData: receives
    CheckProductAvailabilityService ..> ProductAvailabilityResult: returns
    ApplyPromotionService ..> ApplyPromotionData: receives
    ApplyPromotionService ..> AppliedPromotionResult: returns
    ReserveStockService ..> ReserveStockData: receives
    ReserveStockService ..> StockReservationResult: returns
%% Factories
    ProductFactory ..> Product: creates
    ProductVariantFactory ..> ProductVariant: creates
    CategoryFactory ..> Category: creates
    PromotionFactory ..> Promotion: creates
%% ========================================
%% STYLES
%% ========================================
classDef modelClass fill: #3498db, stroke:#2980b9, stroke-width:2px, color:#fff
classDef valueObjectClass fill: #e74c3c, stroke:#c0392b, stroke-width:2px, color:#fff
classDef sharedVOClass fill: #e67e22, stroke:#d35400, stroke-width:2px, color:#fff
classDef enumClass fill: #9b59b6, stroke:#8e44ad, stroke-width:2px, color:#fff
classDef castClass fill: #1abc9c, stroke:#16a085, stroke-width:2px, color:#fff
classDef serviceClass fill: #f39c12, stroke:#d68910, stroke-width:2px, color:#fff
classDef dataClass fill: #95a5a6, stroke:#7f8c8d, stroke-width:2px, color:#fff
classDef interfaceClass fill: #34495e, stroke:#2c3e50, stroke-width:2px, color:#fff
classDef repoClass fill: #16a085, stroke:#117a65, stroke-width:2px, color:#fff
classDef factoryClass fill: #7d3c98, stroke:#633974, stroke-width:2px, color:#fff

class Product, ProductVariant, Category, Promotion modelClass
class PriceValueObject, StockValueObject, VariantAttributesValueObject, DiscountValueObject, PromotionPeriodValueObject valueObjectClass
class Money, Quantity sharedVOClass
class PromotionTypeEnum enumClass
class PriceValueObjectCast, StockValueObjectCast, VariantAttributesValueObjectCast, DiscountValueObjectCast, PromotionPeriodValueObjectCast castClass
class CheckProductAvailabilityService, ApplyPromotionService, ReserveStockService serviceClass
class CheckProductAvailabilityData, ProductAvailabilityResult, ApplyPromotionData, AppliedPromotionResult, ReserveStockData, StockReservationResult dataClass
class CheckProductAvailabilityInterface, ApplyPromotionInterface, ReserveStockInterface interfaceClass
class ProductRepository, ProductVariantRepository, PromotionRepository, CategoryRepository repoClass
class ProductFactory, ProductVariantFactory, CategoryFactory, PromotionFactory factoryClass
```

---

## Estructura de Archivos

```
Modules/Catalog/
├── app/
│   ├── Models/
│   │   ├── Product.php
│   │   ├── ProductVariant.php
│   │   ├── Category.php
│   │   └── Promotion.php
│   │
│   ├── ValueObjects/
│   │   ├── Product/
│   │   │   ├── PriceValueObject.php
│   │   │   └── StockValueObject.php
│   │   │
│   │   ├── ProductVariant/
│   │   │   └── VariantAttributesValueObject.php
│   │   │
│   │   └── Promotion/
│   │       ├── DiscountValueObject.php
│   │       └── PromotionPeriodValueObject.php
│   │
│   ├── Casts/
│   │   ├── PriceValueObjectCast.php
│   │   ├── StockValueObjectCast.php
│   │   ├── VariantAttributesValueObjectCast.php
│   │   ├── DiscountValueObjectCast.php
│   │   └── PromotionPeriodValueObjectCast.php
│   │
│   ├── Enums/
│   │   └── PromotionTypeEnum.php
│   │
│   ├── Services/
│   │   ├── CheckProductAvailabilityService.php
│   │   ├── ApplyPromotionService.php
│   │   └── ReserveStockService.php
│   │
│   ├── Contracts/
│   │   ├── CheckProductAvailabilityInterface.php
│   │   ├── ApplyPromotionInterface.php
│   │   └── ReserveStockInterface.php
│   │
│   ├── Data/
│   │   ├── CheckProductAvailabilityData.php
│   │   ├── ProductAvailabilityResult.php
│   │   ├── ApplyPromotionData.php
│   │   ├── AppliedPromotionResult.php
│   │   ├── ReserveStockData.php
│   │   └── StockReservationResult.php
│   │
│   └── Repositories/
│       ├── ProductRepository.php
│       ├── ProductVariantRepository.php
│       ├── PromotionRepository.php
│       └── CategoryRepository.php
│
├── database/
│   ├── factories/
│   │   ├── ProductFactory.php
│   │   ├── ProductVariantFactory.php
│   │   ├── CategoryFactory.php
│   │   └── PromotionFactory.php
│   │
│   └── migrations/
│       ├── 2025_01_01_000001_create_categories_table.php
│       ├── 2025_01_01_000002_create_products_table.php
│       ├── 2025_01_01_000003_create_product_variants_table.php
│       ├── 2025_01_01_000004_create_promotions_table.php
│       └── 2025_01_01_000005_create_product_promotion_table.php
│
└── tests/
    ├── Unit/
    │   ├── ValueObjects/
    │   │   ├── PriceValueObjectTest.php
    │   │   ├── StockValueObjectTest.php
    │   │   ├── VariantAttributesValueObjectTest.php
    │   │   ├── DiscountValueObjectTest.php
    │   │   └── PromotionPeriodValueObjectTest.php
    │   │
    │   └── Enums/
    │       └── PromotionTypeEnumTest.php
    │
    └── Feature/
        ├── Services/
        │   ├── CheckProductAvailabilityServiceTest.php
        │   ├── ApplyPromotionServiceTest.php
        │   └── ReserveStockServiceTest.php
        │
        └── Models/
            ├── ProductTest.php
            ├── ProductVariantTest.php
            ├── CategoryTest.php
            └── PromotionTest.php
```

---

## Value Objects Compartidos

Los siguientes Value Objects son compartidos y se encuentran en `app/ValueObjects/` (raíz del proyecto):

- **Money**: maneja montos monetarios con operaciones aritméticas
- **Quantity**: representa cantidades con validaciones

Estos Value Objects son reutilizables por otros módulos (Orders, Payments, etc.).

---

## Reglas de Negocio del Módulo

### Product

1. Un producto pertenece a una única categoría (relación 1:1)
2. El stock debe ser siempre >= 0
3. El precio debe ser positivo
4. Un producto puede tener múltiples variantes
5. Un producto puede tener múltiples promociones aplicables
6. Solo los productos activos (`is_active = true`) son visibles en el catálogo público

### ProductVariant

1. Una variante pertenece a un único producto
2. Tiene precio y stock independientes del producto padre
3. Los atributos (talle, color, etc.) se almacenan en `VariantAttributesValueObject`
4. Solo las variantes activas son seleccionables en el carrito

### Promotion

1. Las promociones tienen vigencia definida por `PromotionPeriodValueObject`
2. Pueden ser de tipo porcentual o precio fijo (`PromotionTypeEnum`)
3. Las promociones no se acumulan (se aplica la más beneficiosa)
4. Solo las promociones activas y dentro de vigencia aplican
5. Una promoción puede aplicarse a múltiples productos (relación N:N)

### Stock (Transaccional)

1. El descuento de stock debe ser transaccional (usar locks pessimistas)
2. El stock se divide en `available` (disponible) y `reserved` (reservado)
3. Al crear un pedido, se descuenta de `available` y se suma a `reserved`
4. Al cancelar un pedido, se revierte la operación

---

## Interfaces Expuestas a Otros Módulos

El módulo **Catalog** expone las siguientes interfaces para comunicación síncrona con otros módulos:

### CheckProductAvailabilityInterface

**Propósito**: Verificar si un producto tiene stock disponible.

**Uso típico**: El módulo **Cart** consulta disponibilidad antes de agregar items al carrito.

```php
$result = $checkAvailability->check(new CheckProductAvailabilityData(
    productId: 1,
    variantId: null,
    quantity: 2
));

if (!$result->isAvailable) {
    throw new InsufficientStockException($result->message);
}
```

### ApplyPromotionInterface

**Propósito**: Aplicar la mejor promoción disponible a un producto.

**Uso típico**: El módulo **Orders** aplica promociones al calcular el total del pedido.

```php
$result = $applyPromotion->apply(new ApplyPromotionData(
    productId: 1,
    originalPrice: new Money(5000, 'ARS'),
    checkDate: now()
));

$finalPrice = $result->finalPrice;
$discount = $result->discountAmount;
```

### ReserveStockInterface

**Propósito**: Reservar stock de forma transaccional al confirmar un pedido.

**Uso típico**: El módulo **Orders** reserva stock al crear un nuevo pedido.

```php
$result = $reserveStock->reserve(new ReserveStockData(
    productId: 1,
    variantId: null,
    quantity: 2
));

if (!$result->success) {
    throw new StockReservationFailedException($result->errorMessage);
}
```

---

## Eventos Emitidos

El módulo **Catalog** NO emite eventos de dominio en el MVP. Los cambios de stock y promociones son manejados de forma
síncrona mediante interfaces.

**Nota**: En futuras iteraciones se podrían agregar eventos como:

- `ProductStockLowEvent` (cuando el stock baja de un umbral)
- `PromotionActivatedEvent` (cuando una promoción entra en vigencia)
- `PromotionExpiredEvent` (cuando una promoción termina)

---

## Testing

### Unit Tests (Obligatorios)

- **Value Objects**: validar constructor, invariantes, métodos de dominio, Wireable
- **Enums**: validar casos y métodos auxiliares
- **Casts**: validar mapeo bidireccional (get/set)

### Feature Tests (Obligatorios)

- **Services**: validar lógica de negocio completa con fixtures
- **Models**: validar relaciones, scopes, comportamiento con Factories
- **Repositories**: validar consultas y actualización de datos

### Ejemplo de Unit Test para Value Object

```php
it('valida que PriceValueObject no permite valores negativos', function () {
    expect(fn() => new PriceValueObject(-100, 'ARS'))
        ->toThrow(InvalidArgumentException::class);
});

it('formatea correctamente el precio', function () {
    $price = new PriceValueObject(5000, 'ARS');
    expect($price->format())->toBe('$50.00');
});

it('convierte correctamente a Money', function () {
    $price = new PriceValueObject(5000, 'ARS');
    $money = $price->toMoney();
    
    expect($money)->toBeInstanceOf(Money::class)
        ->and($money->cents)->toBe(5000)
        ->and($money->currency)->toBe('ARS');
});
```

---

## Notas de Implementación

### Performance

- **Eager Loading**: usar `with()` en consultas que incluyan relaciones para evitar N+1
- **Índices**: agregar índices en columnas `is_active`, `category_id`, fechas de promociones
- **Cache**: considerar cache de promociones activas (TTL corto, ej: 5 minutos)

### Seguridad

- **Validación**: todos los inputs deben validarse (precio > 0, stock >= 0)
- **SQL Injection**: usar Eloquent ORM exclusivamente
- **Mass Assignment**: proteger con `$fillable` o `$guarded`
- **Soft Deletes**: usar en `Product`, `ProductVariant`, `Category` para mantener historial

### Transaccionalidad

- La reserva de stock DEBE ejecutarse dentro de una transacción DB con lock pesimista:
  ```php
  DB::transaction(function () use ($productId, $quantity) {
      $product = Product::lockForUpdate()->findOrFail($productId);
      $product->stock = $product->stock->reserve($quantity);
      $product->save();
  });
  ```

---

## Referencias

- **Convenciones Laravel**: `/memory-bank/laravel/conventions.md`
- **Definición del Proyecto**: `docs/project_definition.md`
- **Diagrama de Módulos**: `docs/module_diagram.md`
- **Value Objects (detalle)**: `/memory-bank/laravel/value-objects.md`

---

## Leyenda del Diagrama

### Colores

- **Azul**: Modelos Eloquent
- **Rojo**: Value Objects específicos del modelo
- **Naranja**: Value Objects compartidos
- **Morado**: Enums
- **Turquesa**: Casts de Eloquent
- **Amarillo**: Services
- **Gris**: Data Objects (Spatie Laravel Data)
- **Gris Oscuro**: Interfaces (Contracts)
- **Verde**: Repositories
- **Morado Oscuro**: Factories

### Relaciones

- **Línea continua con flecha**: Relación Eloquent (HasMany, BelongsTo, BelongsToMany)
- **Línea punteada**: Uso/dependencia
- **Línea punteada con triángulo**: Implementación de interfaz

---

*Diagrama generado siguiendo las convenciones técnicas del proyecto Laravel 12 con Filament v4, Livewire v3 y
arquitectura modular.*

