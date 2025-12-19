---
title: "Cart Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Cart module following the 5-agent architecture"
module: "Cart"
phase: "2 - MVP Funcional"
dependencies:
  - "Catalog"
  - "Security"
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
---

# Cart Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Cart module** for a single-tenant e-commerce platform. This module manages the shopping cart experience for anonymous users, including:

- Session-based cart management (add, remove, update quantities)
- Product and variant validation
- Stock availability checking
- Promotion application
- Checkout form processing
- Order creation orchestration

This module is **critical** for the MVP and represents the core user journey from browsing to purchase.

---

## Architecture Overview

This system follows a **5-agent architecture** where each agent has specific responsibilities:

1. **Agent A**: Contracts, Data Objects, Value Objects, Enums
2. **Agent B**: Actions (Commands, Queries, Internal)
3. **Agent C**: Repositories, Models, Persistence, Migrations
4. **Agent D**: HTTP, Livewire/Volt, Filament, Tests Feature
5. **Agent E**: Events, Listeners, Jobs

You must implement the module following this structure, respecting Laravel 12 conventions and the project's strict quality standards.

---

## Project Context

**Project:** E-Commerce WhatsApp + Mercado Libre

**Architecture:**
- **Type:** Monolith with Laravel Modules
- **Frontend public:** Livewire v3 + Volt (anonymous users)
- **Backoffice:** Filament v4 (merchants only)
- **Backend:** Laravel 12 + PHP 8.5

**Quality Gates (NON-NEGOTIABLE):**
- ✅ PHPStan level 6+ (strict types everywhere)
- ✅ Pint (PSR-12 strict)
- ✅ Pest 4 (unit + feature tests, 100% coverage for critical paths)
- ✅ All classes `final`
- ✅ All Value Objects `final readonly` + `Wireable`
- ✅ No `protected` methods (composition over inheritance)
- ✅ No public arrays (use Value Objects or Data Objects)

**Key Constraints:**
- Single-tenant (one merchant per instance)
- No authentication for public frontend
- Session-based cart (no persistence for logged-in users)
- Cart expires after 7 days
- Max 50 items per cart, max 9999 quantity per item

---

## Domain Model Reference

### Key Entities

**Cart:**
- Session-based shopping cart
- Contains CartItemCollection
- Totals calculated dynamically
- Expires after 7 days of inactivity

**CartItem:**
- Individual item with product/variant reference
- Price snapshot at add time (immutable)
- Quantity validation against stock
- Promotion application (one per item)

**CheckoutData:**
- CustomerData (name, phone, WhatsApp consent)
- AddressData (delivery address)
- PaymentMethodType (Mercado Pago, Cash, Transfer)
- Observations

### Key Value Objects

**CartId** (UUID), **CartItemId** (UUID), **Money** (cents-based), **Quantity** (validated), **PhoneNumber** (normalized), **CustomerData**, **AddressData**, **PaymentMethodType** (enum)

### Key Services

**CartService:**
- Cart lifecycle management
- Item operations (add, remove, update)
- Stock validation coordination
- Promotion application

**CheckoutService:**
- Checkout validation
- Stock reservation (pessimistic locking)
- Order creation coordination
- WhatsApp notification

**StockValidator:**
- Stock availability checking
- Pessimistic locking during checkout
- Stock release on failure

**PromotionApplicator:**
- Find applicable promotions
- Calculate discounts
- Apply best promotion per item

---

## Business Rules (CRITICAL)

### Cart Management
1. One cart per session (Laravel session ID)
2. Cart expires after 7 days of inactivity
3. Max 50 items per cart (configurable)
4. Max 9999 quantity per item (configurable)
5. Totals recalculated automatically on every change
6. Cart cleared after successful order creation

### Item Management
7. Product or variant must exist and be active
8. Stock validated before adding/updating quantity
9. Price snapshot taken at add time (not updated later)
10. If variant specified, use variant price/stock; otherwise use product price/stock
11. Only one promotion per item (best discount applied)
12. Cannot add same product/variant twice (update quantity instead)

### Stock Validation
13. Stock checked at: add to cart, update quantity, checkout start, order creation
14. Pessimistic locking during checkout (`SELECT FOR UPDATE`)
15. Stock lock timeout: 30 seconds
16. Stock released on checkout failure
17. Stock decremented atomically with order creation

### Promotion Application
18. Only one promotion per item
19. Promotion must be active and within validity dates
20. Promotion types: percentage discount or fixed price
21. Best discount wins if multiple promotions available
22. Promotions applied automatically on cart total calculation

### Checkout Process
23. Customer data required: name, phone, WhatsApp consent
24. Address required: street, number, city, state
25. Payment method required: Mercado Pago, Cash, or Transfer
26. Rate limiting: 5 attempts/hour per IP, 3 orders/hour per phone
27. Max 2 active orders per phone (configurable 2-5)
28. Captcha validation required
29. Stock revalidated before order creation
30. Checkout is atomic: stock + order creation in transaction

### Data Integrity
31. All money values stored in cents (integer)
32. Phone numbers normalized before storage
33. Value Objects used for type safety
34. Database transactions for all write operations
35. Optimistic locking for cart updates
36. Pessimistic locking for stock operations

---

## Module Structure

```
Modules/Cart/
├── Contracts/                 # Agent A
│   ├── Commands/
│   ├── Queries/
│   └── Data/
├── ValueObjects/              # Agent A
│   ├── CartId.php
│   ├── CartItemId.php
│   ├── CustomerData.php
│   ├── AddressData.php
│   └── ValidationResult.php
├── Enums/                     # Agent A
│   └── PaymentMethodType.php
├── Casts/                     # Agent A
│   ├── CartIdCast.php
│   ├── CartItemIdCast.php
│   ├── MoneyCast.php
│   └── QuantityCast.php
├── Actions/                   # Agent B
│   ├── Commands/
│   │   ├── AddItemToCartAction.php
│   │   ├── RemoveItemFromCartAction.php
│   │   ├── UpdateCartItemQuantityAction.php
│   │   ├── ClearCartAction.php
│   │   └── ProcessCheckoutAction.php
│   ├── Queries/
│   │   ├── GetCartAction.php
│   │   ├── CalculateCartTotalsAction.php
│   │   └── ValidateCartStockAction.php
│   └── Internal/
│       ├── ApplyPromotionsAction.php
│       ├── ReserveStockAction.php
│       └── ReleaseStockAction.php
├── Exceptions/                # Agent B
│   ├── CartNotFoundException.php
│   ├── CartItemNotFoundException.php
│   ├── InsufficientStockException.php
│   ├── MaxCartItemsExceededException.php
│   ├── InvalidQuantityException.php
│   ├── InvalidCheckoutDataException.php
│   ├── StockReservationFailedException.php
│   └── CartExpiredException.php
├── Models/                    # Agent C
│   ├── Cart.php
│   └── CartItem.php
├── Repositories/              # Agent C
│   ├── CartRepository.php
│   └── CartItemRepository.php
├── Database/                  # Agent C
│   ├── Factories/
│   │   ├── CartFactory.php
│   │   └── CartItemFactory.php
│   └── Migrations/
│       ├── xxxx_create_carts_table.php
│       └── xxxx_create_cart_items_table.php
├── Livewire/                  # Agent D
│   ├── CartComponent.php
│   ├── CheckoutComponent.php
│   └── AddToCartButton.php
├── routes/                    # Agent D
│   └── web.php
├── Events/                    # Agent E
│   ├── CartCreated.php
│   ├── ItemAddedToCart.php
│   ├── ItemRemovedFromCart.php
│   ├── ItemQuantityUpdated.php
│   ├── CheckoutStarted.php
│   ├── CheckoutCompleted.php
│   └── CheckoutFailed.php
├── Listeners/                 # Agent E
│   ├── ClearCartAfterCheckout.php
│   └── ReleaseStockOnFailure.php
└── Tests/
    ├── Unit/                  # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   └── Actions/
    └── Feature/               # Agent D
        ├── CartManagementTest.php
        ├── CheckoutFlowTest.php
        └── StockValidationTest.php
```

---

## Dependencies

### With Catalog Module
- **Read:** Product data (name, price, stock, active status)
- **Read:** ProductVariant data (name, price, stock, active status)
- **Read:** Promotion data (type, amount, validity, applicable products)
- **Interfaces consumed:**
  - `CheckProductAvailabilityInterface`
  - `ApplyPromotionInterface`
  - `GetProductDetailsInterface`

### With Orders Module
- **Write:** Create Order from CheckoutData
- **Interfaces consumed:**
  - `CreateOrderInterface`

### With Security Module
- **Validation:** Rate limiting on checkout
- **Validation:** Captcha on checkout
- **Validation:** Max active orders per phone
- **Interfaces consumed:**
  - `ValidatePhoneNumberInterface`
  - `CheckActiveOrdersLimitInterface`

### With WhatsApp Module
- **Trigger:** Send notification on order creation (event-driven)

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**
1. **Value Objects:**
   - `CartId` (UUID-based, Wireable)
   - `CartItemId` (UUID-based, Wireable)
   - `CustomerData` (name, phone, email?, whatsapp_consent, Wireable)
   - `AddressData` (street, number, apartment?, city, state, postal_code?, reference?, Wireable)
   - `ValidationResult` (is_valid, errors, warnings, Wireable)

2. **Enums:**
   - `PaymentMethodType` (MERCADO_PAGO, CASH, TRANSFER, backed string enum)

3. **Casts:**
   - `CartIdCast` (UUID ↔ CartId)
   - `CartItemIdCast` (UUID ↔ CartItemId)
   - `MoneyCast` (cents + currency ↔ Money)
   - `QuantityCast` (int ↔ Quantity)
   - `PhoneNumberCast` (string + country_code ↔ PhoneNumber)

4. **Data Objects (Spatie Laravel Data):**
   - `CartData` (id, session_id, items, subtotal, discount, total)
   - `CartItemData` (id, product_id, variant_id?, name, price, quantity, subtotal, discount, total)
   - `CheckoutData` (cart_id, customer, address, payment_method, observations, totals)
   - `CartTotalsData` (subtotal, discount, total, item_count)

5. **Contracts (Commands):**
   - `AddItemToCartInterface`
   - `RemoveItemFromCartInterface`
   - `UpdateCartItemQuantityInterface`
   - `ClearCartInterface`
   - `ProcessCheckoutInterface`

6. **Contracts (Queries):**
   - `GetCartInterface`
   - `CalculateCartTotalsInterface`
   - `ValidateCartStockInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] All VOs validate in constructor (throw exceptions)
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] Unit tests for all VOs (validation, behavior)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**
1. **Actions Commands:**
   - `AddItemToCartAction` (validate stock, price snapshot, add/update item)
   - `RemoveItemFromCartAction` (remove item, recalculate totals)
   - `UpdateCartItemQuantityAction` (validate stock, update quantity, recalculate)
   - `ClearCartAction` (remove all items)
   - `ProcessCheckoutAction` (validate, reserve stock, create order, notify, clear cart)

2. **Actions Queries:**
   - `GetCartAction` (get or create cart by session)
   - `CalculateCartTotalsAction` (apply promotions, calculate subtotal/discount/total)
   - `ValidateCartStockAction` (check all items against current stock)

3. **Actions Internal:**
   - `ApplyPromotionsAction` (find best promotion per item, calculate discounts)
   - `ReserveStockAction` (pessimistic lock, reserve stock)
   - `ReleaseStockAction` (release reserved stock on failure)

4. **Exceptions:**
   - `CartNotFoundException` (404)
   - `CartItemNotFoundException` (404)
   - `InsufficientStockException` (422, with product_id, requested, available)
   - `MaxCartItemsExceededException` (422)
   - `InvalidQuantityException` (422)
   - `InvalidCheckoutDataException` (422, with ValidationResult)
   - `StockReservationFailedException` (409)
   - `CartExpiredException` (410)

**Unit Tests:**
- Mock repositories, external dependencies
- Test business logic in isolation
- Edge cases: stock limits, concurrent updates, expired carts
- 100% coverage of critical paths

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method (`handle()` or `execute()`)
- [ ] All Actions use dependency injection
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**
1. **Models:**
   - `Cart` (id: uuid, session_id, subtotal_cents, discount_cents, total_cents, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `hasMany(CartItem::class)`
   - `CartItem` (id: uuid, cart_id, product_id, variant_id?, name, price_cents, quantity, subtotal_cents, applied_promotion_id?, discount_cents, total_cents, is_available, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Cart::class)`, `belongsTo(Product::class)`, `belongsTo(ProductVariant::class)`, `belongsTo(Promotion::class)`

2. **Repositories:**
   - `CartRepository` (findBySessionId, findById, save, delete, deleteExpired)
   - `CartItemRepository` (findByCartId, save, delete, deleteByCartId)

3. **Migrations:**
   - `create_carts_table` (uuid primary key, session_id unique index, cents columns, timestamps, indexes)
   - `create_cart_items_table` (uuid primary key, foreign keys with cascade delete, cents columns, indexes)

4. **Factories:**
   - `CartFactory` (with items, with promotions applied)
   - `CartItemFactory` (with product, with variant, with discount)

**Integration Tests:**
- Database transactions
- Cascade deletes
- Pessimistic locking
- Index performance

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs
- [ ] All models are `final`
- [ ] All repositories implement contracts
- [ ] Factories for all models
- [ ] Migrations with proper indexes and foreign keys
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Livewire, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**
1. **Livewire Components:**
   - `CartComponent` (main cart display, add/remove/update, totals)
     - Public properties: `Cart $cart`
     - Actions: `addToCart()`, `removeFromCart()`, `updateQuantity()`, `clearCart()`
     - Events emitted: `cart-updated`, `item-added`, `item-removed`
   - `CheckoutComponent` (checkout form, validation, submission)
     - Public properties: `CheckoutData $checkout`
     - Actions: `submitCheckout()`
     - Real-time validation
     - Rate limiting
   - `AddToCartButton` (quick add from product listing)

2. **Volt Pages:**
   - `cart.blade.php` (cart view page)
   - `checkout.blade.php` (checkout page)

3. **Routes (web.php):**
   - `GET /cart` → CartComponent
   - `GET /checkout` → CheckoutComponent

4. **Filament Resources:**
   - **None** (Cart module has no backoffice management)

**Feature Tests:**
- Complete user flows (add to cart → checkout → order creation)
- Stock validation scenarios
- Promotion application
- Rate limiting enforcement
- Error handling (insufficient stock, expired cart, validation errors)
- Smoke tests for all pages

**Acceptance Criteria:**
- [ ] All Livewire components use typed properties
- [ ] Real-time validation on forms
- [ ] Rate limiting middleware applied
- [ ] Captcha validation on checkout
- [ ] Feature tests cover all user journeys
- [ ] Smoke tests for all pages
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**
1. **Events:**
   - `CartCreated` (Cart)
   - `ItemAddedToCart` (Cart, CartItem)
   - `ItemRemovedFromCart` (Cart, CartItemId)
   - `ItemQuantityUpdated` (Cart, CartItem, old_quantity, new_quantity)
   - `CheckoutStarted` (Cart, CheckoutData)
   - `CheckoutCompleted` (Order, Cart)
   - `CheckoutFailed` (Cart, CheckoutData, ValidationResult)

2. **Listeners:**
   - `ClearCartAfterCheckout` (listens to `CheckoutCompleted`)
   - `ReleaseStockOnFailure` (listens to `CheckoutFailed`)

3. **Jobs:**
   - None (WhatsApp notification handled by WhatsApp module)

**Event Tests:**
- Event dispatching
- Listener execution
- Idempotency

**Acceptance Criteria:**
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Listeners implement `ShouldQueue` where appropriate
- [ ] Event tests verify dispatching and handling
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

## Quality Checklist (Before Completion)

### Code Quality
- [ ] PHPStan level 6+ passes (strict types everywhere)
- [ ] Pint executed (PSR-12 strict)
- [ ] All classes are `final`
- [ ] No `protected` methods
- [ ] All Value Objects are `final readonly` + `Wireable`
- [ ] No public arrays (use Value Objects or Data Objects)

### Testing
- [ ] Unit tests for all Value Objects (validation, behavior)
- [ ] Unit tests for all Actions (mocked dependencies)
- [ ] Feature tests for all user journeys
- [ ] Integration tests for database operations
- [ ] Smoke tests for all pages
- [ ] 100% coverage for critical paths

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex business rules have inline comments
- [ ] README.md in module root (overview, setup, usage)

### Database
- [ ] Migrations have proper indexes
- [ ] Foreign keys with cascade deletes
- [ ] Factories for all models

### Performance
- [ ] N+1 queries avoided (eager loading)
- [ ] Indexes on foreign keys and frequently queried columns
- [ ] Pessimistic locking only during checkout

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Use public arrays in method signatures
- Mix different responsibilities in a single Action
- Forget to validate stock at checkout (concurrent purchase issue)
- Skip transaction wrapping for write operations
- Use optimistic locking for stock operations (use pessimistic)
- Allow cart modifications after checkout starts (lock the cart)
- Forget to release stock on checkout failure
- Skip rate limiting or captcha validation
- Expose sensitive data in exceptions or logs
- Use float for money calculations (use integer cents)
- Skip index creation (performance killer)
- Forget to clear cart after successful order creation

✅ **DO:**
- Use Value Objects for all domain concepts
- Wrap all write operations in transactions
- Use pessimistic locking for stock operations
- Validate stock at every critical step
- Apply rate limiting and captcha
- Release stock on any failure
- Clear cart after successful checkout
- Use cents for money (integer precision)
- Create proper indexes
- Test all edge cases (concurrent updates, stock limits)

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Anonymous user can add products/variants to cart
   - User can update quantities and remove items
   - Cart persists across page refreshes (session-based)
   - Promotions applied automatically
   - Stock validated at every step
   - Checkout form validates all required fields
   - Order created successfully on checkout
   - WhatsApp notification sent to merchant
   - Cart cleared after successful checkout

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - No N+1 queries
   - Proper indexes on all foreign keys
   - Pessimistic locking only during checkout
   - Cart operations complete in < 200ms

4. **Security:**
   - Rate limiting enforced
   - Captcha validation on checkout
   - Max active orders per phone enforced
   - Phone numbers normalized
   - No SQL injection vectors (Eloquent ORM only)

---

## Validation Commands

```bash
# Static analysis
./vendor/bin/sail composer run phpstan

# Code style
./vendor/bin/sail bin pint --dirty

# Refactoring
./vendor/bin/sail composer run rector

# Unit tests
./vendor/bin/sail test --filter=Unit

# Feature tests
./vendor/bin/sail test --filter=Feature

# All tests
./vendor/bin/sail test

# Specific module tests
./vendor/bin/sail test Modules/Cart
```

---

## Final Notes

- This module is **phase-critical** for the MVP.
- Follow the 5-agent architecture strictly.
- Refer to the full domain model (`@e-commerce-wa-ml/cart/domain_model.md`) for detailed specifications.
- Consult project definition (`@e-commerce-wa-ml/project_definition.md`) for context.
- Use Value Objects guide (`@laravel/conventions/value-objects.md`) for implementation patterns.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 12-16 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
