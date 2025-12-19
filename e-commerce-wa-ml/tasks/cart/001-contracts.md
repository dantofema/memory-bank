---
task_id: "cart-001-contracts"
module: "Cart"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Cart - Contracts, Value Objects, Enums y Data Objects"
priority: "HIGH"
estimated_time: "8 hours"
dependencies: []
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 001: Cart - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Cart: Value Objects, Enums, Eloquent Casts y Data Transfer Objects que definen la frontera pública del módulo y garantizan la validez de datos.

## 📋 Contexto

El módulo Cart gestiona carritos de compra session-based para usuarios anónimos. Esta tarea establece los fundamentos con tipos de datos inmutables y validados.

### Referencias del Domain Model
- **Value Objects:** CartId, CartItemId, CustomerData, AddressData (líneas 165-228)
- **Enums:** PaymentMethodType (líneas 229-234)
- **Business Rules:** Cart/Item Management (líneas 132-172)

## 📦 Artefactos a Crear

### 1. Value Object: CartId (UUID)

**Ubicación:** `Modules/Cart/ValueObjects/CartId.php`

**Especificaciones:**

```php
final readonly class CartId implements Wireable
{
    public string $value;
    
    private function __construct(string $value)
    {
        if (!Str::isUuid($value)) {
            throw new \InvalidArgumentException('CartId must be a valid UUID');
        }
        $this->value = $value;
    }
    
    public static function generate(): self;
    public static function fromString(string $value): self;
    public function equals(CartId $other): bool;
    public function toString(): string;
    
    // Wireable
    public function toLivewire(): string;
    public static function fromLivewire($value): self;
}
```

---

### 2. Value Object: CartItemId (UUID)

**Ubicación:** `Modules/Cart/ValueObjects/CartItemId.php`

Similar structure to CartId.

---

### 3. Value Object: CustomerData

**Ubicación:** `Modules/Cart/ValueObjects/CustomerData.php`

**Especificaciones:**

```php
final readonly class CustomerData implements Wireable
{
    public string $name;
    public PhoneNumber $phone;
    public ?string $email;
    public bool $whatsapp_consent;
    
    public function __construct(
        string $name,
        PhoneNumber $phone,
        ?string $email,
        bool $whatsapp_consent
    ) {
        // Validate name (min 2 chars, max 255)
        // Validate email format if provided
        // Store normalized values
    }
    
    public static function from(array $data): self;
    public function toArray(): array;
}
```

**Reglas de Negocio:**
- Name required: min 2, max 255 caracteres
- Phone required y normalizado
- Email opcional pero validado si presente
- WhatsApp consent required (boolean)

---

### 4. Value Object: AddressData

**Ubicación:** `Modules/Cart/ValueObjects/AddressData.php`

**Especificaciones:**

```php
final readonly class AddressData implements Wireable
{
    public string $street;
    public string $number;
    public ?string $apartment;
    public string $city;
    public string $state;
    public ?string $postal_code;
    public ?string $reference;
    
    public function __construct(...) {
        // Validate required fields
        // Normalize and trim all strings
    }
    
    public static function from(array $data): self;
    public function toString(): string; // Format for display
    public function toArray(): array;
}
```

**Reglas de Negocio:**
- Street, number, city, state required
- Apartment, postal_code, reference optional
- All strings trimmed and normalized

---

### 5. Value Object: ValidationResult

**Ubicación:** `Modules/Cart/ValueObjects/ValidationResult.php`

**Especificaciones:**

```php
final readonly class ValidationResult implements Wireable
{
    public bool $is_valid;
    public array $errors;
    public array $warnings;
    
    public static function success(): self;
    public static function failure(array $errors, array $warnings = []): self;
    public function isValid(): bool;
    public function hasErrors(): bool;
    public function hasWarnings(): bool;
    public function getErrors(): array;
    public function getWarnings(): array;
    public function addError(string $field, string $message): self;
}
```

---

### 6. Enum: PaymentMethodType

**Ubicación:** `Modules/Cart/Enums/PaymentMethodType.php`

**Especificaciones:**

```php
enum PaymentMethodType: string
{
    case MERCADO_PAGO = 'mercado_pago';
    case CASH = 'cash';
    case TRANSFER = 'transfer';
    
    public function label(): string;
    public function icon(): string;
    public function description(): string;
}
```

---

### 7-11. Eloquent Casts

**Ubicación:** `Modules/Cart/Casts/`

- `CartIdCast.php` - UUID ↔ CartId
- `CartItemIdCast.php` - UUID ↔ CartItemId
- `MoneyCast.php` - cents ↔ Money VO (shared)
- `QuantityCast.php` - int ↔ Quantity VO (shared)
- `PhoneNumberCast.php` - string ↔ PhoneNumber VO (shared)

---

### 12-15. Data Objects (Spatie Laravel Data)

**Ubicación:** `Modules/Cart/Data/`

```php
// CartData.php
final class CartData extends Data
{
    public function __construct(
        public CartId $id,
        public string $session_id,
        public array $items, // CartItemData[]
        public Money $subtotal,
        public Money $discount,
        public Money $total,
    ) {}
}

// CartItemData.php
final class CartItemData extends Data
{
    public function __construct(
        public CartItemId $id,
        public int $product_id,
        public ?int $variant_id,
        public string $name,
        public Money $price,
        public Quantity $quantity,
        public Money $subtotal,
        public Money $discount,
        public Money $total,
    ) {}
}

// CheckoutData.php
final class CheckoutData extends Data
{
    public function __construct(
        public CartId $cart_id,
        public CustomerData $customer,
        public AddressData $address,
        public PaymentMethodType $payment_method,
        public ?string $observations,
        public Money $subtotal,
        public Money $discount,
        public Money $total,
    ) {}
}

// CartTotalsData.php
final class CartTotalsData extends Data
{
    public function __construct(
        public Money $subtotal,
        public Money $discount,
        public Money $total,
        public int $item_count,
    ) {}
}
```

---

## ✅ Criterios de Aceptación

### Funcionales
- [ ] CartId y CartItemId son UUIDs válidos
- [ ] CustomerData valida name, phone, email
- [ ] AddressData valida campos requeridos
- [ ] ValidationResult diferencia errores y warnings
- [ ] PaymentMethodType tiene 3 casos
- [ ] Todos los VOs implementan Wireable
- [ ] Todos los Casts son bidireccionales
- [ ] Todos los Data Objects usan Spatie Laravel Data

### Técnicos
- [ ] Todas las clases son `final`
- [ ] VOs son `readonly`
- [ ] Tipado fuerte completo
- [ ] Constructor valida y lanza excepciones
- [ ] Tests con Pest 4
- [ ] Cobertura 100% VOs
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado

## 🧪 Tests Requeridos

**Ubicación:** `tests/Unit/Cart/ValueObjects/`, `tests/Unit/Cart/Enums/`, `tests/Unit/Cart/Casts/`

```php
describe('CartId VO', function () {
    it('generates valid UUID');
    it('validates UUID format');
    it('throws for invalid UUID');
    it('equals comparison works');
    it('implements Wireable');
});

describe('CustomerData VO', function () {
    it('creates from valid data');
    it('validates name length');
    it('validates email format');
    it('requires whatsapp consent');
    it('normalizes phone number');
});

// Similar tests for other VOs, Enums, Casts
```

---

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Bloqueante para:** Task 002, 003
