---
title: "Orders Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Orders module following the 5-agent architecture"
module: "Orders"
phase: "2 - MVP Funcional"
module_type: "CORE"
dependencies:
  - "Catalog"
  - "Payments"
  - "Security"
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
---

# Orders Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Orders module**, which is the **CORE module** in Phase 2 (MVP Funcional) of the e-commerce platform. This is the **most critical module** of the entire system as it orchestrates:

- **Order creation** from shopping cart with stock validation
- **Transactional stock reservation** with pessimistic locking
- **Order lifecycle management** with state machines
- **Payment status tracking** independent from order status
- **Limited order editing** with business rules enforcement
- **Audit trail** for all status changes
- **Anti-abuse controls** (active orders limit, rate limiting)
- **Stock release** on cancellations/rejections

This module coordinates with Catalog (products/stock), Payments (payment methods), WhatsApp (notifications), and Security (anti-abuse).

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
- Anonymous users (no authentication required)
- Stock must be transactional with pessimistic locking
- Orders have two independent states: OrderStatus and PaymentStatus
- Limited editing based on order status
- Full audit trail for status changes
- Max N active orders per phone (configurable 2-5)

---

## Domain Model Overview

### Key Entities

**Order:**
- Core aggregate root with order number, customer data, status, payment info
- Has OrderItems (1:many), Address (1:1), OrderStatusLogs (1:many)
- Two independent states: OrderStatus and PaymentStatus
- Limited editing based on status (cannot edit DELIVERED or REFUNDED)
- Historical prices preserved forever

**OrderItem:**
- Line item with product/variant reference, quantity, prices, discount
- Historical prices (never change after creation)
- Quantity can be modified if stock available and order editable
- Cannot add/remove products, only modify quantities

**Address:**
- Delivery address (1:1 with Order)
- Can be edited until order is DELIVERED
- Optional GPS coordinates for optimization

**OrderStatusLog:**
- Immutable audit trail for status changes
- Tracks OrderStatus and PaymentStatus changes
- Includes user_id, timestamps, old/new values, reason

### Key Value Objects

**PhoneNumber** (normalized, validated, WhatsApp format), **Money** (cents-based, shared), **Quantity** (validated int, shared), **Coordinates** (GPS lat/lng), **OrderNumber** (unique format: ORD-YYYYMMDD-XXXXX)

### Key Enums

**OrderStatus** (NEW, CONFIRMED, IN_DELIVERY, DELIVERED, REJECTED, CANCELLED)
**PaymentStatus** (PENDING, PAID, REFUNDED)
**PaymentMethod** (MERCADO_PAGO, CASH, TRANSFER)
**StatusField** (ORDER_STATUS, PAYMENT_STATUS)

### Key Actions

**CreateOrderAction:**
- Validate active orders limit
- Validate stock availability
- Apply promotions
- Calculate totals
- Reserve stock transactionally (pessimistic lock)
- Create order + items + address
- Emit OrderCreated event

**UpdateOrderAction:**
- Validate order is editable
- Validate stock for quantity changes
- Update allowed fields
- Recalculate totals
- Adjust stock if quantities changed

**ChangeOrderStatusAction:**
- Validate state transition
- Update status and timestamps
- Create audit log
- Release stock if CANCELLED/REJECTED
- Emit OrderStatusChanged event

**ChangePaymentStatusAction:**
- Validate state transition
- Update payment status
- Create audit log
- Emit PaymentStatusChanged event

**ValidateActiveOrdersLimit:**
- Count active orders by phone
- Compare with configurable limit
- Throw exception if exceeded

**ReleaseStockOnCancellation:**
- Release stock for all order items
- Triggered by status change to CANCELLED/REJECTED

---

## Business Rules (CRITICAL)

### Order Creation (Rules 1-6)
1. Stock must be available for all items
2. Phone must not exceed active orders limit (configurable 2-5, default: 2)
3. Rate limiting: 5 attempts/hour per IP, 3 orders/hour per phone
4. Stock is reserved transactionally with pessimistic locking (`SELECT FOR UPDATE`)
5. Historical prices are saved and never change
6. Promotions are applied at creation time based on validity dates

### Order Editing (Rules 7-12)
7. Orders in DELIVERED or REFUNDED status **cannot be edited**
8. Customer name cannot be changed after status != NEW
9. Products cannot be added or removed, only quantities modified
10. Quantity changes require stock validation
11. Payment method can be changed at any time by merchant
12. Address can be edited until status = DELIVERED

### Status Transitions (Rules 13-17)
13. Only valid transitions are allowed (see state machine diagrams below)
14. All transitions must be audited with user_id, timestamp, reason
15. Terminal states (DELIVERED, REJECTED, CANCELLED, REFUNDED) cannot transition
16. Cancellations and rejections trigger automatic stock release
17. Confirmed orders must have confirmed_at timestamp

### OrderStatus State Machine
```
NEW → CONFIRMED | REJECTED | CANCELLED
CONFIRMED → IN_DELIVERY | CANCELLED
IN_DELIVERY → DELIVERED | CANCELLED
DELIVERED → (terminal, no transitions)
REJECTED → (terminal, no transitions)
CANCELLED → (terminal, no transitions)
```

### PaymentStatus State Machine
```
PENDING → PAID | REFUNDED
PAID → REFUNDED
REFUNDED → (terminal, no transitions)
```

### Stock Management (Rules 18-21)
18. Stock is decremented transactionally on order creation
19. Stock is released on cancellation or rejection
20. Stock validation is required for quantity updates
21. Pessimistic locking (`SELECT FOR UPDATE`) prevents race conditions

### Payment Flow (Rules 22-25)
22. Payment status is **independent** of order status
23. Mercado Pago payments update automatically via webhook
24. Cash/Transfer payments require manual confirmation by merchant
25. Payment method can be changed by merchant at any time

### Anti-Abuse (Rules 26-29)
26. Maximum N active orders per phone (configurable 2-5)
27. Rate limiting: 5 attempts/hour per IP, 3 orders/hour per phone
28. Phone number validation and normalization required
29. Captcha validation on checkout

### Data Integrity (Rules 30-35)
30. `order.total = order.subtotal - order.discount`
31. `order.subtotal = SUM(order_items.total)`
32. `order_item.total = order_item.subtotal - order_item.discount`
33. `order_item.subtotal = order_item.unit_price * order_item.quantity`
34. Audit logs are **immutable** (no updates/deletes allowed)
35. Historical prices **never change** after order creation

---

## Module Structure

```
Modules/Orders/
├── Contracts/                         # Agent A
│   ├── Commands/
│   │   ├── CreateOrderInterface.php
│   │   ├── UpdateOrderInterface.php
│   │   ├── ChangeOrderStatusInterface.php
│   │   └── ChangePaymentStatusInterface.php
│   ├── Queries/
│   │   ├── GetOrderInterface.php
│   │   ├── GetOrdersByStatusInterface.php
│   │   ├── GetOrdersByPhoneInterface.php
│   │   └── GetOrderAuditTrailInterface.php
│   └── Data/
│       ├── OrderData.php
│       ├── OrderItemData.php
│       ├── AddressData.php
│       ├── CreateOrderData.php
│       ├── UpdateOrderData.php
│       └── OrderAuditData.php
├── ValueObjects/                      # Agent A
│   ├── Order/
│   │   ├── OrderNumber.php
│   │   └── OrderId.php
│   ├── OrderItem/
│   │   └── OrderItemId.php
│   ├── Address/
│   │   ├── AddressId.php
│   │   └── Coordinates.php
│   └── Audit/
│       └── AuditLogId.php
├── Enums/                             # Agent A
│   ├── OrderStatus.php
│   ├── PaymentStatus.php
│   ├── PaymentMethod.php
│   └── StatusField.php
├── Casts/                             # Agent A
│   ├── OrderNumberCast.php
│   ├── CoordinatesCast.php
│   ├── OrderStatusCast.php
│   ├── PaymentStatusCast.php
│   └── PaymentMethodCast.php
├── Actions/                           # Agent B
│   ├── Commands/
│   │   ├── CreateOrderAction.php
│   │   ├── UpdateOrderAction.php
│   │   ├── ChangeOrderStatusAction.php
│   │   ├── ChangePaymentStatusAction.php
│   │   └── CancelOrderAction.php
│   ├── Queries/
│   │   ├── GetOrderAction.php
│   │   ├── GetOrdersByStatusAction.php
│   │   ├── GetOrdersByPhoneAction.php
│   │   ├── GetActiveOrdersCountAction.php
│   │   └── GetOrderAuditTrailAction.php
│   └── Internal/
│       ├── ValidateActiveOrdersLimitAction.php
│       ├── ReserveStockAction.php
│       ├── ReleaseStockAction.php
│       ├── CalculateOrderTotalsAction.php
│       ├── ApplyPromotionsToOrderAction.php
│       ├── ValidateOrderEditableAction.php
│       └── LogStatusChangeAction.php
├── Exceptions/                        # Agent B
│   ├── OrderNotFoundException.php
│   ├── OrderNotEditableException.php
│   ├── InvalidStatusTransitionException.php
│   ├── InvalidPaymentStatusTransitionException.php
│   ├── InsufficientStockException.php
│   ├── ActiveOrdersLimitExceededException.php
│   ├── RateLimitExceededException.php
│   ├── StockReservationFailedException.php
│   └── OrderValidationException.php
├── Models/                            # Agent C
│   ├── Order.php
│   ├── OrderItem.php
│   ├── Address.php
│   └── OrderStatusLog.php
├── Repositories/                      # Agent C
│   ├── OrderRepository.php
│   ├── OrderItemRepository.php
│   ├── AddressRepository.php
│   └── OrderStatusLogRepository.php
├── Database/                          # Agent C
│   ├── Factories/
│   │   ├── OrderFactory.php
│   │   ├── OrderItemFactory.php
│   │   ├── AddressFactory.php
│   │   └── OrderStatusLogFactory.php
│   ├── Migrations/
│   │   ├── xxxx_create_orders_table.php
│   │   ├── xxxx_create_order_items_table.php
│   │   ├── xxxx_create_addresses_table.php
│   │   └── xxxx_create_order_status_logs_table.php
│   └── Seeders/
│       └── OrderSeeder.php
├── Livewire/                          # Agent D
│   ├── OrderConfirmation.php
│   └── OrderTracking.php
├── Filament/                          # Agent D
│   └── Resources/
│       └── OrderResource.php
│           ├── Pages/
│           │   ├── ListOrders.php
│           │   ├── ViewOrder.php
│           │   └── EditOrder.php
│           └── Widgets/
│               ├── OrderStatsWidget.php
│               └── RecentOrdersWidget.php
├── routes/                            # Agent D
│   └── web.php
├── Events/                            # Agent E
│   ├── OrderCreated.php
│   ├── OrderStatusChanged.php
│   └── PaymentStatusChanged.php
├── Listeners/                         # Agent E
│   ├── SendOrderCreatedNotification.php
│   ├── SendOrderStatusChangeNotification.php
│   ├── SendPaymentConfirmationNotification.php
│   └── ReleaseStockOnCancellation.php
└── Tests/
    ├── Unit/                          # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   └── Actions/
    └── Feature/                       # Agent D
        ├── OrderCreationTest.php
        ├── OrderEditingTest.php
        ├── OrderStatusTransitionsTest.php
        ├── PaymentStatusTransitionsTest.php
        ├── StockReservationTest.php
        ├── ActiveOrdersLimitTest.php
        └── OrderAuditTrailTest.php
```

---

## Interfaces Exposed (Communication with Other Modules)

### CreateOrderInterface

```php
interface CreateOrderInterface
{
    /**
     * Create an order from checkout data.
     * 
     * @throws ActiveOrdersLimitExceededException
     * @throws InsufficientStockException
     * @throws RateLimitExceededException
     * @throws OrderValidationException
     */
    public function execute(CreateOrderData $data): OrderData;
}
```

**Consumed by:** Cart

---

### UpdateOrderInterface

```php
interface UpdateOrderInterface
{
    /**
     * Update editable fields of an order.
     * 
     * @throws OrderNotFoundException
     * @throws OrderNotEditableException
     * @throws InsufficientStockException
     */
    public function execute(int $orderId, UpdateOrderData $data): OrderData;
}
```

**Consumed by:** Backoffice (Filament)

---

### ChangeOrderStatusInterface

```php
interface ChangeOrderStatusInterface
{
    /**
     * Change order status with validation and audit.
     * 
     * @throws OrderNotFoundException
     * @throws InvalidStatusTransitionException
     */
    public function execute(int $orderId, OrderStatus $newStatus, int $userId, ?string $reason = null): void;
}
```

**Consumed by:** Backoffice (Filament)

---

### ChangePaymentStatusInterface

```php
interface ChangePaymentStatusInterface
{
    /**
     * Change payment status (manual or webhook).
     * 
     * @throws OrderNotFoundException
     * @throws InvalidPaymentStatusTransitionException
     */
    public function execute(int $orderId, PaymentStatus $newStatus, ?int $userId = null, ?string $reason = null): void;
}
```

**Consumed by:** Payments module (webhooks), Backoffice (manual)

---

## Interfaces Consumed (Dependencies)

### From Catalog Module

```php
CheckProductAvailabilityInterface::check(ProductId, ?VariantId, Quantity): AvailabilityResult
ApplyPromotionInterface::apply(ProductId, Money): PriceResult
GetProductDetailsInterface::get(ProductId): ProductData
```

### From Security Module

```php
ValidatePhoneNumberInterface::validate(string): PhoneNumber
CheckActiveOrdersLimitInterface::check(PhoneNumber): bool
```

### From Payments Module

```php
RequestPaymentInterface::requestMercadoPago(OrderId, Money): PaymentLinkData
```

---

## Events Emitted (Asynchronous Communication)

### OrderCreated

```php
final readonly class OrderCreated
{
    public function __construct(
        public int $orderId,
        public string $orderNumber,
        public PhoneNumber $customerPhone,
        public Money $total,
        public Carbon $createdAt
    ) {}
}
```

**Consumed by:** WhatsApp module (notifications)

---

### OrderStatusChanged

```php
final readonly class OrderStatusChanged
{
    public function __construct(
        public int $orderId,
        public OrderStatus $oldStatus,
        public OrderStatus $newStatus,
        public int $changedBy,
        public Carbon $changedAt
    ) {}
}
```

**Consumed by:** WhatsApp module (notifications), Stock release listener

---

### PaymentStatusChanged

```php
final readonly class PaymentStatusChanged
{
    public function __construct(
        public int $orderId,
        public PaymentStatus $oldStatus,
        public PaymentStatus $newStatus,
        public ?int $changedBy,
        public Carbon $changedAt
    ) {}
}
```

**Consumed by:** WhatsApp module (notifications)

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `OrderNumber` (format: ORD-YYYYMMDD-XXXXX, unique, Wireable)
   - `OrderId` (int-based, Wireable, validation > 0)
   - `OrderItemId` (int-based, Wireable, validation > 0)
   - `AddressId` (int-based, Wireable, validation > 0)
   - `AuditLogId` (int-based, Wireable, validation > 0)
   - `Coordinates` (lat/lng, validation ranges, Wireable)

2. **Enums:**
   - `OrderStatus` (NEW, CONFIRMED, IN_DELIVERY, DELIVERED, REJECTED, CANCELLED)
     - Methods: `canTransitionTo()`, `isTerminal()`, `label()`, `color()`
   - `PaymentStatus` (PENDING, PAID, REFUNDED)
     - Methods: `canTransitionTo()`, `isTerminal()`, `label()`, `color()`
   - `PaymentMethod` (MERCADO_PAGO, CASH, TRANSFER)
     - Methods: `requiresExternalLink()`, `label()`
   - `StatusField` (ORDER_STATUS, PAYMENT_STATUS)

3. **Casts:**
   - `OrderNumberCast` (string ↔ OrderNumber)
   - `OrderIdCast` (int ↔ OrderId)
   - `OrderItemIdCast` (int ↔ OrderItemId)
   - `AddressIdCast` (int ↔ AddressId)
   - `CoordinatesCast` (lat/lng ↔ Coordinates)
   - `OrderStatusCast` (string ↔ OrderStatus enum)
   - `PaymentStatusCast` (string ↔ PaymentStatus enum)
   - `PaymentMethodCast` (string ↔ PaymentMethod enum)

4. **Data Objects (Spatie Laravel Data):**
   - `OrderData` (id, order_number, customer_name, customer_phone, status, payment_status, payment_method, subtotal, discount, total, items, address, timestamps)
   - `OrderItemData` (id, product_id, variant_id, product_name, variant_name, quantity, unit_price, discount, subtotal, total, promotion_id)
   - `AddressData` (street, number, floor, apartment, city, state, postal_code, reference, coordinates)
   - `CreateOrderData` (customer_name, customer_phone, notes, payment_method, address, items[])
   - `UpdateOrderData` (customer_phone?, notes?, payment_method?, address?, items[]?)
   - `OrderAuditData` (field, old_value, new_value, changed_by, reason, timestamp)

5. **Contracts (Commands):**
   - `CreateOrderInterface`
   - `UpdateOrderInterface`
   - `ChangeOrderStatusInterface`
   - `ChangePaymentStatusInterface`

6. **Contracts (Queries):**
   - `GetOrderInterface`
   - `GetOrdersByStatusInterface`
   - `GetOrdersByPhoneInterface`
   - `GetActiveOrdersCountInterface`
   - `GetOrderAuditTrailInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] All VOs validate in constructor (throw exceptions)
- [ ] OrderNumber format validation (ORD-YYYYMMDD-XXXXX)
- [ ] Coordinates validate lat (-90 to 90) and lng (-180 to 180)
- [ ] All Enums implement `canTransitionTo()` with state machine logic
- [ ] OrderStatus terminal states: DELIVERED, REJECTED, CANCELLED
- [ ] PaymentStatus terminal state: REFUNDED
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] Unit tests for all VOs (validation, behavior)
- [ ] Unit tests for Enum transitions (all valid/invalid combinations)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**

1. **Actions Commands:**
   - `CreateOrderAction`
     - Validate active orders limit (call ValidateActiveOrdersLimitAction)
     - Validate stock availability (call Catalog::CheckProductAvailability)
     - Apply promotions (call Catalog::ApplyPromotion)
     - Calculate totals (call CalculateOrderTotalsAction)
     - Reserve stock transactionally (call ReserveStockAction with pessimistic lock)
     - Create order + items + address
     - Emit OrderCreated event
   - `UpdateOrderAction`
     - Validate order editable (call ValidateOrderEditableAction)
     - Validate stock for quantity changes
     - Update allowed fields (phone, notes, payment_method, address, item quantities)
     - Recalculate totals if quantities changed
     - Adjust stock (release old, reserve new)
   - `ChangeOrderStatusAction`
     - Validate transition (enum.canTransitionTo())
     - Update status and timestamps (confirmed_at, delivered_at, cancelled_at)
     - Create audit log (call LogStatusChangeAction)
     - Release stock if CANCELLED/REJECTED (call ReleaseStockAction)
     - Emit OrderStatusChanged event
   - `ChangePaymentStatusAction`
     - Validate transition (enum.canTransitionTo())
     - Update payment status
     - Create audit log (call LogStatusChangeAction)
     - Emit PaymentStatusChanged event
   - `CancelOrderAction`
     - Wrapper for ChangeOrderStatusAction with status = CANCELLED
     - Additional validation and reason requirement

2. **Actions Queries:**
   - `GetOrderAction` (get order by ID with items, address, audit trail)
   - `GetOrdersByStatusAction` (paginated, filter by OrderStatus or PaymentStatus)
   - `GetOrdersByPhoneAction` (paginated, normalized phone lookup)
   - `GetActiveOrdersCountAction` (count orders with status NEW/CONFIRMED/IN_DELIVERY)
   - `GetOrderAuditTrailAction` (get all status changes for an order, chronological)

3. **Actions Internal:**
   - `ValidateActiveOrdersLimitAction`
     - Count active orders by normalized phone
     - Compare with configured limit (default: 2)
     - Throw exception if exceeded
   - `ReserveStockAction`
     - Use pessimistic locking (`SELECT FOR UPDATE`)
     - Decrement stock for each item (product or variant)
     - Atomic transaction with timeout (30s)
   - `ReleaseStockAction`
     - Increment stock for each item
     - Called on cancellation/rejection
   - `CalculateOrderTotalsAction`
     - Calculate item subtotals and totals
     - Sum order subtotal, discount, total
     - Validate invariants (total = subtotal - discount)
   - `ApplyPromotionsToOrderAction`
     - Apply best promotion per item
     - Calculate discounts
     - Return updated items with discount info
   - `ValidateOrderEditableAction`
     - Check status != DELIVERED and != REFUNDED
     - Throw OrderNotEditableException if not editable
   - `LogStatusChangeAction`
     - Create OrderStatusLog entry
     - Immutable record with user_id, timestamps, old/new values

4. **Exceptions:**
   - `OrderNotFoundException` (404)
   - `OrderNotEditableException` (422, "Order cannot be edited in status: {status}")
   - `InvalidStatusTransitionException` (422, "Cannot transition from {old} to {new}")
   - `InvalidPaymentStatusTransitionException` (422, "Cannot transition payment status from {old} to {new}")
   - `InsufficientStockException` (422, "Insufficient stock for product {id}: requested {requested}, available {available}")
   - `ActiveOrdersLimitExceededException` (422, "Phone {phone} has {count} active orders, maximum allowed: {limit}")
   - `RateLimitExceededException` (429)
   - `StockReservationFailedException` (409, "Failed to reserve stock due to concurrent operation")
   - `OrderValidationException` (422)

**Unit Tests:**
- Mock all repositories and external dependencies
- Test CreateOrderAction flow (happy path + all exceptions)
- Test UpdateOrderAction with all editable fields
- Test status transitions (all valid and invalid combinations)
- Test active orders limit validation
- Test stock reservation with pessimistic locking (mock)
- Test stock release logic
- Test totals calculation (invariants)
- Test promotion application
- 100% coverage of critical paths

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method (`execute()`)
- [ ] All Actions use dependency injection
- [ ] CreateOrderAction validates stock before reserving
- [ ] ReserveStockAction uses pessimistic locking
- [ ] Status transitions validate with enum.canTransitionTo()
- [ ] All status changes create audit logs
- [ ] Stock released automatically on CANCELLED/REJECTED
- [ ] Totals calculation respects invariants
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**

1. **Models:**
   - `Order` (id, order_number unique, customer_name, customer_phone, notes, status, payment_status, payment_method, subtotal_cents, discount_cents, total_cents, confirmed_at, delivered_at, cancelled_at, confirmed_by_user_id, timestamps, soft_deletes)
     - Uses Casts for Value Objects and Enums
     - Relationships: `hasMany(OrderItem::class)`, `hasOne(Address::class)`, `hasMany(OrderStatusLog::class)`
     - Scopes: `active()`, `byStatus()`, `byPaymentStatus()`, `byPhone()`
     - Methods: `canBeEdited()`, `isActive()`, `isPaid()`, `isCompleted()`
   - `OrderItem` (id, order_id, product_id, variant_id nullable, product_name, variant_name nullable, quantity, unit_price_cents, discount_cents, subtotal_cents, total_cents, promotion_id nullable, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Order::class)`, `belongsTo(Product::class)`, `belongsTo(ProductVariant::class)`, `belongsTo(Promotion::class)`
   - `Address` (id, order_id, street, number, floor, apartment, city, state, postal_code, reference, latitude, longitude, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Order::class)`
   - `OrderStatusLog` (id, order_id, user_id, field, old_value, new_value, reason, created_at)
     - **No updates or deletes** (immutable)
     - Relationships: `belongsTo(Order::class)`, `belongsTo(User::class)`
     - Scope: `chronological()`

2. **Repositories:**
   - `OrderRepository` (findById, findByOrderNumber, findByPhone, findByStatus, save, delete, countActiveByPhone)
   - `OrderItemRepository` (findByOrderId, save, delete, deleteByOrderId)
   - `AddressRepository` (findByOrderId, save, delete)
   - `OrderStatusLogRepository` (findByOrderId, create) - **No update/delete methods**

3. **Migrations:**
   - `create_orders_table`
     - Columns: id, order_number (unique), customer_name, customer_phone, notes (text), status (enum string), payment_status (enum string), payment_method (enum string), subtotal_cents, discount_cents, total_cents, confirmed_at, delivered_at, cancelled_at, confirmed_by_user_id (foreign), timestamps, soft_deletes
     - Indexes: order_number (unique), customer_phone, status, payment_status, created_at, confirmed_by_user_id
   - `create_order_items_table`
     - Columns: id, order_id (foreign cascade), product_id (foreign), variant_id (foreign nullable), product_name, variant_name (nullable), quantity, unit_price_cents, discount_cents, subtotal_cents, total_cents, promotion_id (foreign nullable), timestamps
     - Indexes: order_id, product_id, variant_id
     - Cascade delete with order
   - `create_addresses_table`
     - Columns: id, order_id (foreign cascade unique), street, number, floor, apartment, city, state, postal_code, reference, latitude (decimal), longitude (decimal), timestamps
     - Indexes: order_id (unique)
     - Cascade delete with order
   - `create_order_status_logs_table`
     - Columns: id, order_id (foreign), user_id (foreign), field (enum string), old_value, new_value, reason (text nullable), created_at
     - Indexes: order_id, created_at
     - **No soft deletes** (immutable audit)

4. **Factories:**
   - `OrderFactory` (states: new, confirmed, in_delivery, delivered, rejected, cancelled, withItems, withAddress, paid, pending, refunded)
   - `OrderItemFactory` (withProduct, withVariant, withDiscount, withPromotion)
   - `AddressFactory` (withCoordinates, complete)
   - `OrderStatusLogFactory` (orderStatusChange, paymentStatusChange)

5. **Seeders:**
   - `OrderSeeder` (10-20 orders with various states, items, addresses)

**Integration Tests:**
- Database transactions
- Cascade deletes (order → items, order → address)
- Soft deletes for orders
- Immutability of audit logs (no updates/deletes)
- Unique constraint (order_number)
- Foreign key constraints
- Pessimistic locking simulation
- Factory states

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs and Enums
- [ ] All models are `final`
- [ ] Order model has scopes for filtering
- [ ] OrderStatusLog is immutable (only create, no update/delete)
- [ ] All repositories implement contracts
- [ ] Factories for all models with relevant states
- [ ] Migrations with proper indexes and foreign keys
- [ ] Unique constraint on order_number
- [ ] Cascade deletes configured (order → items, order → address)
- [ ] Soft deletes on orders only (not on logs)
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Livewire, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **Livewire Components (Public Frontend):**
   - `OrderConfirmation` (order success page after checkout)
     - Public properties: `OrderId $orderId`
     - Display: order number, items, total, payment link (if Mercado Pago)
     - Actions: copy order number, open WhatsApp
   - `OrderTracking` (simple tracking by order number + phone)
     - Public properties: `string $orderNumber`, `string $phone`
     - Display: current status, estimated delivery
     - No authentication required

2. **Volt Pages:**
   - `orders/confirmation.blade.php` (order confirmation page)
   - `orders/tracking.blade.php` (order tracking page)

3. **Routes (web.php):**
   - `GET /orders/{orderNumber}/confirmation` → OrderConfirmation
   - `GET /orders/tracking` → OrderTracking

4. **Filament Resources (Backoffice):**
   - `OrderResource` (full CRUD with restrictions)
     - **List Page:**
       - Table: order_number, customer_name, customer_phone, status badge, payment_status badge, total, created_at
       - Filters: status, payment_status, date range, payment_method
       - Bulk actions: none (editing is sensitive)
       - Sorting: created_at desc by default
     - **View Page:**
       - Order details (read-only)
       - Items table
       - Address display
       - Audit trail (status changes with user, timestamp, reason)
       - Actions: Change Status, Change Payment Status, Edit (if editable)
     - **Edit Page:**
       - Form: customer_phone, notes, payment_method, address fields, item quantities
       - Disabled fields: customer_name (after confirmation), product selection
       - Validation: stock availability for quantity changes
       - Show warning if order not editable
     - **Actions:**
       - `ChangeStatusAction` (modal with status dropdown, reason textarea)
       - `ChangePaymentStatusAction` (modal with status dropdown, reason textarea)
       - `CancelOrderAction` (modal with reason textarea)
     - **Relation Managers:**
       - OrderItemsRelationManager (display only, quantity editable with validation)
       - AuditTrailRelationManager (display only, chronological)

5. **Filament Widgets:**
   - `OrderStatsWidget` (total orders, orders by status, orders by payment status)
   - `RecentOrdersWidget` (last 10 orders, quick status view)

**Feature Tests:**
- Order creation flow from checkout data
- Stock validation during order creation
- Active orders limit enforcement
- Order editing (allowed and restricted fields)
- Status transitions (valid and invalid)
- Payment status changes
- Stock release on cancellation
- Audit trail creation and retrieval
- Smoke tests for all pages (confirmation, tracking, Filament resource)

**Acceptance Criteria:**
- [ ] All Livewire components use typed properties
- [ ] Order confirmation page displays payment link for Mercado Pago
- [ ] Order tracking validates phone + order number
- [ ] Filament resource shows audit trail
- [ ] Status change actions create audit logs
- [ ] Edit page disables customer_name after confirmation
- [ ] Edit page validates stock for quantity changes
- [ ] Feature tests cover all critical paths
- [ ] Smoke tests for all public and backoffice pages
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - `OrderCreated` (orderId, orderNumber, customerPhone, total, createdAt)
   - `OrderStatusChanged` (orderId, oldStatus, newStatus, changedBy, changedAt)
   - `PaymentStatusChanged` (orderId, oldStatus, newStatus, changedBy, changedAt)

2. **Listeners:**
   - `SendOrderCreatedNotification` (listens to `OrderCreated`)
     - Dispatch job to send WhatsApp message to merchant
     - Message template: "New order {orderNumber} from {phone} - Total: {total}"
   - `SendOrderStatusChangeNotification` (listens to `OrderStatusChanged`)
     - Dispatch job to send WhatsApp message to merchant
     - Message template: "Order {orderNumber} status changed: {oldStatus} → {newStatus}"
   - `SendPaymentConfirmationNotification` (listens to `PaymentStatusChanged`)
     - Only if newStatus = PAID
     - Dispatch job to send WhatsApp message to merchant
     - Message template: "Payment confirmed for order {orderNumber}"
   - `ReleaseStockOnCancellation` (listens to `OrderStatusChanged`)
     - Only if newStatus = CANCELLED or REJECTED
     - Call ReleaseStockAction to increment stock

3. **Jobs:**
   - None directly in Orders module (WhatsApp jobs handled by WhatsApp module)

**Event Tests:**
- Event dispatching on order creation
- Event dispatching on status changes
- Listener execution (mock jobs)
- Stock release triggered correctly
- Idempotency of listeners

**Acceptance Criteria:**
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Listeners implement `ShouldQueue` for async processing
- [ ] ReleaseStockOnCancellation only triggers for CANCELLED/REJECTED
- [ ] Notification listeners only dispatch jobs (don't block)
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
- [ ] Unit tests for all Enums (state transitions)
- [ ] Unit tests for all Actions (mocked dependencies)
- [ ] Feature tests for all user journeys
- [ ] Integration tests for database operations
- [ ] Concurrency tests for stock reservation
- [ ] Smoke tests for all pages
- [ ] 100% coverage for critical paths (order creation, stock reservation, status transitions)

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex business rules have inline comments
- [ ] README.md in module root (overview, setup, usage)

### Database
- [ ] Migrations have proper indexes
- [ ] Foreign keys with cascade deletes where appropriate
- [ ] Unique constraint on order_number
- [ ] Soft deletes on orders
- [ ] No updates/deletes on audit logs
- [ ] Factories for all models with states

### Performance
- [ ] N+1 queries avoided (eager loading)
- [ ] Pessimistic locking only during stock reservation
- [ ] Indexes on customer_phone for active orders lookup
- [ ] Indexes on status fields for filtering

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Allow editing of DELIVERED or REFUNDED orders
- Allow adding/removing products after order creation
- Modify historical prices of order items
- Allow invalid state transitions
- Skip audit logging for status changes
- Use optimistic locking for stock operations (must be pessimistic)
- Forget to release stock on cancellation/rejection
- Skip active orders limit validation
- Allow customer_name changes after confirmation
- Update or delete audit logs (immutable)
- Use float for money calculations (use integer cents)
- Skip rate limiting validation

✅ **DO:**
- Use pessimistic locking (`SELECT FOR UPDATE`) for stock operations
- Create immutable audit logs for all status changes
- Validate state transitions with enum methods
- Release stock automatically on CANCELLED/REJECTED
- Preserve historical prices forever
- Validate active orders limit before creation
- Use integer cents for money values
- Enforce rate limiting per IP and phone
- Allow only quantity changes for existing products
- Emit events for all significant domain changes

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Anonymous users can create orders from cart
   - Stock is reserved transactionally with pessimistic locking
   - Merchants can view/edit orders (with restrictions)
   - Merchants can change order status (with validation)
   - Merchants can change payment status (manual or webhook)
   - Orders are limited per phone (configurable)
   - Status changes are fully audited
   - Stock is released on cancellation/rejection
   - WhatsApp notifications sent for all events
   - Payment links generated for Mercado Pago

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - No N+1 queries
   - Proper indexes on all foreign keys and filter fields
   - Pessimistic locking with 30s timeout
   - Order creation completes in < 500ms
   - Stock reservation prevents race conditions

4. **Security:**
   - Rate limiting enforced
   - Active orders limit enforced
   - Phone numbers normalized
   - No SQL injection vectors (Eloquent ORM only)
   - Audit logs immutable

---

## Configuration

### Environment Variables

```env
# Orders Module
MAX_ACTIVE_ORDERS_PER_PHONE=2
ORDER_RATE_LIMIT_IP=5
ORDER_RATE_LIMIT_PHONE=3
STOCK_LOCK_TIMEOUT=30
ORDER_NUMBER_PREFIX=ORD
```

### Config File: `config/orders.php`

```php
return [
    'max_active_orders_per_phone' => env('MAX_ACTIVE_ORDERS_PER_PHONE', 2),
    'rate_limit_ip' => env('ORDER_RATE_LIMIT_IP', 5),
    'rate_limit_phone' => env('ORDER_RATE_LIMIT_PHONE', 3),
    'stock_lock_timeout' => env('STOCK_LOCK_TIMEOUT', 30),
    'order_number_prefix' => env('ORDER_NUMBER_PREFIX', 'ORD'),
];
```

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
./vendor/bin/sail test Modules/Orders

# Test with coverage
./vendor/bin/sail test --coverage
```

---

## Dependencies and Integration

### Modules that Depend on Orders

- **Reports**: Reads Order data for analytics
- **WhatsApp**: Consumes events for notifications

### External Dependencies

- **Catalog**: Product/stock validation, promotion application
- **Payments**: Payment method handling, webhook processing
- **Security**: Rate limiting, phone validation, active orders limit

---

## Final Notes

- This module is **phase-critical** as it's the core transactional module of the entire system.
- Must be completed in **Phase 2** after Cart module.
- **Stock operations are CRITICAL**: use pessimistic locking, test concurrency scenarios.
- **State machines are STRICT**: use enum methods for transition validation.
- **Audit trail is IMMUTABLE**: no updates or deletes allowed on logs.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 20-24 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
