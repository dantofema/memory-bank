---
title: "Payments Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Payments module following the 5-agent architecture"
module: "Payments"
phase: "3 - Integraciones"
module_type: "CORE"
dependencies:
  - "Orders"
  - "Auth"
  - "Security"
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
---

# Payments Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Payments module**, a **CORE module** in Phase 3 (Integraciones) of the e-commerce platform. This module manages:

- **Payment method handling** (Mercado Pago, Cash, Transfer)
- **External payment link generation** (Mercado Pago integration)
- **Webhook processing** from external providers (idempotent, verified)
- **Manual payment status updates** by merchants
- **Payment status lifecycle** (pending → paid → refunded)
- **Audit trail** for all payment status changes
- **Payment link expiration** and tracking
- **Transaction history** for financial reporting

This module is **independent from order status** but coordinates closely with the Orders module for payment confirmation.

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
- Payment status is **independent** from order status
- Webhooks must be **idempotent** (handle duplicates)
- Webhook signatures must be **verified** (security)
- Only merchants can manually update payment status
- Payment links expire after 24 hours (configurable)
- Each order has exactly **one active payment record**
- No automated refunds (manual only for MVP)

---

## Domain Model Overview

### Key Entities

**Payment (Aggregate Root):**
- Main entity with payment method, status, amount, external metadata
- Has PaymentStatusLog (audit), PaymentTransaction (history), PaymentLink (for Mercado Pago)
- Status lifecycle: pending → paid → refunded (or cancelled)
- Historical amounts are immutable

**PaymentStatusLog:**
- Immutable audit trail for status changes
- Tracks manual merchant updates vs webhook updates
- Records user_id, timestamps, old/new status, reason

**MercadoPagoWebhook:**
- Incoming webhook notifications from Mercado Pago
- Stores raw payload for debugging
- Idempotent processing (duplicate detection)

**PaymentLink:**
- Generated external payment links for Mercado Pago
- Tracks expiration (24h default) and usage status

**PaymentTransaction:**
- Individual transaction records (payment, refund, cancellation)
- Detailed financial tracking for reporting

### Key Value Objects

**PaymentAmount** (currency + amount, decimal 2 precision), **PaymentMethod** (enum: mercado_pago, cash, transfer), **PaymentStatus** (enum: pending, paid, refunded, cancelled), **MercadoPagoMetadata** (transaction_id, payment_link_url, preference_id, expires_at), **PaymentReference** (internal UUID, external transaction ID, order reference)

### Key Enums

**PaymentStatusEnum** (PENDING, PAID, REFUNDED, CANCELLED)
**PaymentMethodEnum** (MERCADO_PAGO, CASH, TRANSFER)
**WebhookTopicEnum** (PAYMENT, MERCHANT_ORDER)

### Key Services

**PaymentService:**
- Payment lifecycle orchestration
- Status transition coordination
- Webhook processing coordination
- Manual confirmation/refund

**MercadoPagoService:**
- Payment link generation (external API)
- Webhook signature verification
- Transaction status polling

**PaymentValidator:**
- Status transition validation
- Payment method validation
- Amount validation
- Webhook payload validation

---

## Business Rules (CRITICAL)

### Payment Creation (Rules 1-4)
1. Payment status is **independent** from order status
2. Payment amount must match order total at creation time
3. Each order has exactly **one active payment record**
4. Historical payment amounts are **immutable** once created

### Payment Methods (Rules 5-7)
5. Three methods supported: Mercado Pago, Cash, Transfer
6. Mercado Pago requires external link generation
7. Cash/Transfer require manual merchant confirmation

### Payment Status Lifecycle (Rules 8-11)
```
pending → paid → refunded
   ↓       ↓
cancelled  cancelled
```
8. Only valid transitions allowed (see state machine)
9. Only merchants can manually update payment status
10. Webhook updates **override** manual updates
11. Payment method can be changed by merchant at any time

### Webhook Processing (Rules 12-16)
12. Webhooks must be **idempotent** (handle duplicates via external_id)
13. Webhook signatures must be **verified** before processing
14. Failed webhook processing does **not** fail the order
15. Webhook payload stored for debugging
16. Rate limiting on webhook endpoints (100/min configurable)

### Payment Links (Rules 17-19)
17. Payment links expire after configured time (default: 24h)
18. Expired links cannot be used (validation)
19. Payment links marked as "used" after successful payment

### Refunds (Rules 20-22)
20. Only `paid` payments can transition to `refunded`
21. Refunds must be processed **manually** (no automation in MVP)
22. Refund creates audit log and transaction record

### Data Integrity (Rules 23-25)
23. All status changes are **audited** with timestamp and user
24. Audit logs are **immutable** (no updates/deletes)
25. Payment amounts stored with 2 decimal precision

---

## Module Structure

```
Modules/Payments/
├── Contracts/                             # Agent A
│   ├── Commands/
│   │   ├── CreatePaymentInterface.php
│   │   ├── UpdatePaymentStatusInterface.php
│   │   ├── ConfirmPaymentInterface.php
│   │   ├── RefundPaymentInterface.php
│   │   └── ProcessWebhookInterface.php
│   ├── Queries/
│   │   ├── GetPaymentInterface.php
│   │   ├── GetPaymentByOrderInterface.php
│   │   ├── GetPaymentHistoryInterface.php
│   │   └── GetPaymentLinkInterface.php
│   └── Data/
│       ├── PaymentData.php
│       ├── CreatePaymentData.php
│       ├── WebhookData.php
│       ├── PaymentLinkData.php
│       └── PaymentHistoryData.php
├── ValueObjects/                          # Agent A
│   ├── Payment/
│   │   ├── PaymentId.php
│   │   ├── PaymentAmount.php
│   │   ├── PaymentReference.php
│   │   └── MercadoPagoMetadata.php
│   └── StatusLog/
│       └── StatusLogId.php
├── Enums/                                 # Agent A
│   ├── PaymentStatusEnum.php
│   ├── PaymentMethodEnum.php
│   └── WebhookTopicEnum.php
├── Casts/                                 # Agent A
│   ├── PaymentAmountCast.php
│   ├── PaymentReferenceCast.php
│   ├── MercadoPagoMetadataCast.php
│   ├── PaymentStatusCast.php
│   └── PaymentMethodCast.php
├── Actions/                               # Agent B
│   ├── Commands/
│   │   ├── CreatePaymentAction.php
│   │   ├── UpdatePaymentStatusAction.php
│   │   ├── ConfirmPaymentManuallyAction.php
│   │   ├── RefundPaymentAction.php
│   │   ├── CancelPaymentAction.php
│   │   └── ProcessWebhookAction.php
│   ├── Queries/
│   │   ├── GetPaymentAction.php
│   │   ├── GetPaymentByOrderAction.php
│   │   ├── GetPaymentHistoryAction.php
│   │   └── GetPaymentLinkAction.php
│   └── Internal/
│       ├── GeneratePaymentLinkAction.php
│       ├── VerifyWebhookSignatureAction.php
│       ├── CheckWebhookDuplicateAction.php
│       ├── LogPaymentStatusChangeAction.php
│       ├── RecordPaymentTransactionAction.php
│       └── ValidatePaymentStatusTransitionAction.php
├── Exceptions/                            # Agent B
│   ├── PaymentNotFoundException.php
│   ├── InvalidPaymentStatusTransitionException.php
│   ├── InvalidPaymentMethodException.php
│   ├── InvalidPaymentAmountException.php
│   ├── PaymentAlreadyProcessedException.php
│   ├── WebhookVerificationFailedException.php
│   ├── PaymentLinkExpiredException.php
│   └── DuplicateWebhookException.php
├── Models/                                # Agent C
│   ├── Payment.php
│   ├── PaymentStatusLog.php
│   ├── MercadoPagoWebhook.php
│   ├── PaymentLink.php
│   └── PaymentTransaction.php
├── Repositories/                          # Agent C
│   ├── PaymentRepository.php
│   ├── PaymentStatusLogRepository.php
│   ├── MercadoPagoWebhookRepository.php
│   ├── PaymentLinkRepository.php
│   └── PaymentTransactionRepository.php
├── Services/                              # Agent C
│   ├── PaymentService.php
│   ├── MercadoPagoService.php
│   └── PaymentValidator.php
├── Database/                              # Agent C
│   ├── Factories/
│   │   ├── PaymentFactory.php
│   │   ├── PaymentStatusLogFactory.php
│   │   ├── MercadoPagoWebhookFactory.php
│   │   ├── PaymentLinkFactory.php
│   │   └── PaymentTransactionFactory.php
│   ├── Migrations/
│   │   ├── xxxx_create_payments_table.php
│   │   ├── xxxx_create_payment_status_logs_table.php
│   │   ├── xxxx_create_mercado_pago_webhooks_table.php
│   │   ├── xxxx_create_payment_links_table.php
│   │   └── xxxx_create_payment_transactions_table.php
│   └── Seeders/
│       └── PaymentSeeder.php
├── Http/                                  # Agent D
│   └── Controllers/
│       └── WebhookController.php
├── Filament/                              # Agent D
│   └── Resources/
│       └── PaymentResource.php
│           ├── Pages/
│           │   ├── ListPayments.php
│           │   ├── ViewPayment.php
│           │   └── EditPayment.php
│           └── Widgets/
│               └── PaymentStatsWidget.php
├── routes/                                # Agent D
│   ├── api.php
│   └── web.php
├── Events/                                # Agent E
│   ├── PaymentCreated.php
│   ├── PaymentStatusChanged.php
│   ├── PaymentConfirmed.php
│   └── PaymentRefunded.php
├── Listeners/                             # Agent E
│   ├── UpdateOrderPaymentStatus.php
│   ├── SendPaymentConfirmationNotification.php
│   └── SendPaymentRefundedNotification.php
└── Tests/
    ├── Unit/                              # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   └── Actions/
    └── Feature/                           # Agent D
        ├── PaymentCreationTest.php
        ├── PaymentStatusUpdateTest.php
        ├── WebhookProcessingTest.php
        ├── WebhookDuplicateHandlingTest.php
        ├── ManualConfirmationTest.php
        ├── RefundProcessingTest.php
        └── PaymentLinkGenerationTest.php
```

---

## Interfaces Exposed (Communication with Other Modules)

### CreatePaymentInterface

```php
interface CreatePaymentInterface
{
    /**
     * Create payment record for an order.
     * 
     * @throws InvalidPaymentAmountException
     * @throws InvalidPaymentMethodException
     */
    public function execute(CreatePaymentData $data): PaymentData;
}
```

**Consumed by:** Orders module

---

### UpdatePaymentStatusInterface

```php
interface UpdatePaymentStatusInterface
{
    /**
     * Update payment status (manual or webhook).
     * 
     * @throws PaymentNotFoundException
     * @throws InvalidPaymentStatusTransitionException
     */
    public function execute(int $paymentId, PaymentStatusEnum $newStatus, ?int $userId = null, ?string $reason = null): void;
}
```

**Consumed by:** Orders module, Webhook processing

---

### RequestPaymentInterface

```php
interface RequestPaymentInterface
{
    /**
     * Generate Mercado Pago payment link.
     * 
     * @throws PaymentNotFoundException
     * @throws PaymentLinkExpiredException
     */
    public function requestMercadoPago(int $orderId, Money $amount): PaymentLinkData;
}
```

**Consumed by:** Orders module (at checkout)

---

### ProcessWebhookInterface

```php
interface ProcessWebhookInterface
{
    /**
     * Process webhook from external provider.
     * 
     * @throws WebhookVerificationFailedException
     * @throws DuplicateWebhookException
     */
    public function process(WebhookData $data): void;
}
```

**Consumed by:** Webhook controller (public endpoint)

---

## Interfaces Consumed (Dependencies)

### From Orders Module

```php
ChangePaymentStatusInterface::execute(int $orderId, PaymentStatus, ?int $userId, ?string $reason): void
```

---

## Events Emitted (Asynchronous Communication)

### PaymentCreated

```php
final readonly class PaymentCreated
{
    public function __construct(
        public int $paymentId,
        public int $orderId,
        public PaymentMethodEnum $method,
        public PaymentAmount $amount,
        public Carbon $createdAt
    ) {}
}
```

**Consumed by:** Reports module

---

### PaymentStatusChanged

```php
final readonly class PaymentStatusChanged
{
    public function __construct(
        public int $paymentId,
        public int $orderId,
        public PaymentStatusEnum $oldStatus,
        public PaymentStatusEnum $newStatus,
        public ?int $changedBy,
        public bool $isWebhookUpdate,
        public Carbon $changedAt
    ) {}
}
```

**Consumed by:** Orders module (update order payment status)

---

### PaymentConfirmed

```php
final readonly class PaymentConfirmed
{
    public function __construct(
        public int $paymentId,
        public int $orderId,
        public PaymentMethodEnum $method,
        public PaymentAmount $amount,
        public Carbon $confirmedAt
    ) {}
}
```

**Consumed by:** WhatsApp module (notifications), Orders module

---

### PaymentRefunded

```php
final readonly class PaymentRefunded
{
    public function __construct(
        public int $paymentId,
        public int $orderId,
        public PaymentAmount $amount,
        public ?int $refundedBy,
        public Carbon $refundedAt
    ) {}
}
```

**Consumed by:** WhatsApp module (notifications), Orders module

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `PaymentId` (int-based, Wireable, validation > 0)
   - `PaymentAmount` (currency + amount, Wireable)
     - Properties: `string $currency`, `float $amount` (2 decimal precision)
     - Methods: `format()`, `equals()`, `add()`, `subtract()`
     - Validation: amount >= 0, currency must be valid ISO (MVP: ARS only)
   - `PaymentReference` (internal UUID, external ID, order ref, Wireable)
     - Properties: `string $internalReference` (UUID), `?string $externalReference`, `string $orderReference`
     - Methods: `getInternal()`, `getExternal()`, `getOrder()`
   - `MercadoPagoMetadata` (transaction_id, link, preference_id, expires_at, Wireable)
     - Properties: `string $transactionId`, `string $paymentLinkUrl`, `Carbon $linkExpiresAt`, `string $preferenceId`, `string $externalReference`
     - Methods: `toArray()`, `isExpired()`
   - `StatusLogId` (int-based, Wireable, validation > 0)

2. **Enums:**
   - `PaymentStatusEnum` (PENDING, PAID, REFUNDED, CANCELLED)
     - Methods: `canTransitionTo()`, `isPending()`, `isPaid()`, `isRefunded()`, `isCancelled()`, `label()`, `color()`
     - Transitions: PENDING → PAID/CANCELLED, PAID → REFUNDED/CANCELLED
   - `PaymentMethodEnum` (MERCADO_PAGO, CASH, TRANSFER)
     - Methods: `requiresExternalLink()`, `isMercadoPago()`, `isCash()`, `isTransfer()`, `label()`
   - `WebhookTopicEnum` (PAYMENT, MERCHANT_ORDER)

3. **Casts:**
   - `PaymentIdCast` (int ↔ PaymentId)
   - `PaymentAmountCast` (amount + currency ↔ PaymentAmount)
   - `PaymentReferenceCast` (internal + external + order ↔ PaymentReference)
   - `MercadoPagoMetadataCast` (JSON ↔ MercadoPagoMetadata)
   - `PaymentStatusCast` (string ↔ PaymentStatusEnum)
   - `PaymentMethodCast` (string ↔ PaymentMethodEnum)

4. **Data Objects (Spatie Laravel Data):**
   - `PaymentData` (id, order_id, reference, amount, method, status, mercado_pago_metadata, created_at, updated_at, paid_at, refunded_at, merchant_id, is_manual_confirmation)
   - `CreatePaymentData` (order_id, amount, method, reference)
   - `WebhookData` (external_id, topic, action, status, payload)
   - `PaymentLinkData` (url, preference_id, expires_at)
   - `PaymentHistoryData` (status_logs[], transactions[])

5. **Contracts (Commands):**
   - `CreatePaymentInterface`
   - `UpdatePaymentStatusInterface`
   - `ConfirmPaymentInterface`
   - `RefundPaymentInterface`
   - `ProcessWebhookInterface`

6. **Contracts (Queries):**
   - `GetPaymentInterface`
   - `GetPaymentByOrderInterface`
   - `GetPaymentHistoryInterface`
   - `GetPaymentLinkInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] All VOs validate in constructor (throw exceptions)
- [ ] PaymentAmount validates >= 0 and 2 decimal precision
- [ ] MercadoPagoMetadata validates expiration date
- [ ] PaymentStatusEnum implements `canTransitionTo()` with state machine logic
- [ ] PaymentStatusEnum terminal states: REFUNDED, CANCELLED
- [ ] PaymentMethodEnum.requiresExternalLink() returns true only for MERCADO_PAGO
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
   - `CreatePaymentAction`
     - Validate payment amount matches order total
     - Generate PaymentReference (UUID)
     - Create Payment with status PENDING
     - Generate Mercado Pago link if method = MERCADO_PAGO
     - Record transaction
     - Emit PaymentCreated event
   - `UpdatePaymentStatusAction`
     - Validate status transition (enum.canTransitionTo())
     - Update status and timestamps (paid_at, refunded_at)
     - Create audit log (call LogPaymentStatusChangeAction)
     - Emit PaymentStatusChanged event
   - `ConfirmPaymentManuallyAction`
     - Validate merchant authorization
     - Wrapper for UpdatePaymentStatusAction with status = PAID
     - Mark as manual confirmation
     - Emit PaymentConfirmed event
   - `RefundPaymentAction`
     - Validate payment is PAID
     - Update status to REFUNDED
     - Create audit log
     - Record refund transaction
     - Emit PaymentRefunded event
   - `CancelPaymentAction`
     - Update status to CANCELLED
     - Create audit log
   - `ProcessWebhookAction`
     - Verify webhook signature (call VerifyWebhookSignatureAction)
     - Check for duplicates (call CheckWebhookDuplicateAction)
     - Store webhook payload in MercadoPagoWebhook
     - Extract payment status from payload
     - Update payment status (call UpdatePaymentStatusAction)
     - Mark webhook as processed
     - Handle errors gracefully (log but don't fail)

2. **Actions Queries:**
   - `GetPaymentAction` (get payment by ID with metadata)
   - `GetPaymentByOrderAction` (get payment by order_id)
   - `GetPaymentHistoryAction` (get status logs and transactions, chronological)
   - `GetPaymentLinkAction` (get active payment link for order)

3. **Actions Internal:**
   - `GeneratePaymentLinkAction`
     - Call MercadoPagoService to generate link
     - Store PaymentLink record with expiration
     - Return PaymentLinkData
   - `VerifyWebhookSignatureAction`
     - Extract signature from headers
     - Verify against webhook secret
     - Throw WebhookVerificationFailedException if invalid
   - `CheckWebhookDuplicateAction`
     - Check MercadoPagoWebhook by external_id
     - Return true if already processed
     - Throw DuplicateWebhookException if duplicate
   - `LogPaymentStatusChangeAction`
     - Create PaymentStatusLog entry
     - Include user_id, old/new status, reason, is_webhook_update
     - Immutable record
   - `RecordPaymentTransactionAction`
     - Create PaymentTransaction entry
     - Types: payment, refund, cancellation
   - `ValidatePaymentStatusTransitionAction`
     - Check enum.canTransitionTo()
     - Throw InvalidPaymentStatusTransitionException if invalid

4. **Exceptions:**
   - `PaymentNotFoundException` (404)
   - `InvalidPaymentStatusTransitionException` (422, "Cannot transition from {old} to {new}")
   - `InvalidPaymentMethodException` (422)
   - `InvalidPaymentAmountException` (422)
   - `PaymentAlreadyProcessedException` (409)
   - `WebhookVerificationFailedException` (401, "Webhook signature verification failed")
   - `PaymentLinkExpiredException` (410, "Payment link has expired")
   - `DuplicateWebhookException` (409, "Webhook already processed")

**Unit Tests:**
- Mock all repositories, external services
- Test CreatePaymentAction (happy path + exceptions)
- Test UpdatePaymentStatusAction with all status transitions
- Test ProcessWebhookAction (signature verification, duplicates, happy path)
- Test webhook duplicate detection
- Test payment link generation
- Test status transition validation (all valid/invalid combinations)
- 100% coverage of critical paths

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method (`execute()`)
- [ ] All Actions use dependency injection
- [ ] CreatePaymentAction generates UUID reference
- [ ] UpdatePaymentStatusAction validates transitions
- [ ] ProcessWebhookAction verifies signature before processing
- [ ] ProcessWebhookAction checks duplicates via external_id
- [ ] ProcessWebhookAction stores raw payload for debugging
- [ ] All status changes create audit logs
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database, no external APIs)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**

1. **Models:**
   - `Payment` (id, order_id, internal_reference UUID, external_reference, order_reference, amount_cents, currency, method, status, mercado_pago_metadata JSON, paid_at, refunded_at, merchant_id, is_manual_confirmation, timestamps)
     - Uses Casts for Value Objects and Enums
     - Relationships: `belongsTo(Order::class)`, `hasMany(PaymentStatusLog::class)`, `hasMany(PaymentTransaction::class)`, `hasOne(PaymentLink::class)`
     - Scopes: `byStatus()`, `byMethod()`, `byOrder()`
     - Methods: `isPending()`, `isPaid()`, `isRefunded()`, `canTransitionTo()`
   - `PaymentStatusLog` (id, payment_id, merchant_id nullable, old_status, new_status, change_reason, is_webhook_update, created_at)
     - **No updates or deletes** (immutable)
     - Relationships: `belongsTo(Payment::class)`, `belongsTo(User::class)`
     - Scope: `chronological()`
   - `MercadoPagoWebhook` (id, external_id unique, topic, action, payload JSON, status, processed, received_at, processed_at, error_message)
     - Relationships: none
     - Scopes: `unprocessed()`, `failed()`
     - Methods: `markAsProcessed()`, `markAsFailed()`
   - `PaymentLink` (id, payment_id, url, preference_id, expires_at, created_at, used)
     - Relationships: `belongsTo(Payment::class)`
     - Methods: `isExpired()`, `markAsUsed()`
   - `PaymentTransaction` (id, payment_id, transaction_type, amount_cents, currency, metadata JSON, created_at)
     - Relationships: `belongsTo(Payment::class)`

2. **Repositories:**
   - `PaymentRepository` (findById, findByOrderId, findByReference, save, delete)
   - `PaymentStatusLogRepository` (findByPaymentId, create) - **No update/delete methods**
   - `MercadoPagoWebhookRepository` (findByExternalId, findUnprocessed, save)
   - `PaymentLinkRepository` (findByPaymentId, findActiveByOrderId, save)
   - `PaymentTransactionRepository` (findByPaymentId, create)

3. **Services:**
   - `PaymentService` (orchestrates payment operations)
   - `MercadoPagoService` (external API integration)
     - Methods: `generatePaymentLink()`, `verifyWebhookSignature()`, `getTransactionStatus()`
     - Use HTTP client for API calls
     - Store API keys in config (never hardcode)
   - `PaymentValidator` (validates business rules)

4. **Migrations:**
   - `create_payments_table`
     - Columns: id, order_id (foreign unique), internal_reference (UUID unique), external_reference (nullable), order_reference, amount_cents, currency, method (enum string), status (enum string), mercado_pago_metadata (JSON nullable), paid_at, refunded_at, merchant_id (foreign nullable), is_manual_confirmation, timestamps
     - Indexes: order_id (unique), internal_reference (unique), status, method, created_at
   - `create_payment_status_logs_table`
     - Columns: id, payment_id (foreign), merchant_id (foreign nullable), old_status, new_status, change_reason (text nullable), is_webhook_update, created_at
     - Indexes: payment_id, created_at
     - **No soft deletes** (immutable)
   - `create_mercado_pago_webhooks_table`
     - Columns: id, external_id (unique), topic, action, payload (JSON), status, processed, received_at, processed_at, error_message (text nullable)
     - Indexes: external_id (unique), processed, received_at
   - `create_payment_links_table`
     - Columns: id, payment_id (foreign unique), url, preference_id, expires_at, created_at, used
     - Indexes: payment_id (unique), expires_at
   - `create_payment_transactions_table`
     - Columns: id, payment_id (foreign), transaction_type, amount_cents, currency, metadata (JSON), created_at
     - Indexes: payment_id, transaction_type, created_at

5. **Factories:**
   - `PaymentFactory` (states: pending, paid, refunded, cancelled, mercadoPago, cash, transfer, withLink)
   - `PaymentStatusLogFactory` (manualUpdate, webhookUpdate)
   - `MercadoPagoWebhookFactory` (processed, unprocessed, failed)
   - `PaymentLinkFactory` (expired, active, used)
   - `PaymentTransactionFactory` (payment, refund, cancellation)

6. **Seeders:**
   - `PaymentSeeder` (5-10 payments with various states, methods, links)

**Integration Tests:**
- Database transactions
- Unique constraints (order_id, internal_reference, external_id)
- Immutability of status logs
- Foreign key constraints
- Factory states

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs and Enums
- [ ] All models are `final`
- [ ] PaymentStatusLog is immutable (only create)
- [ ] Payment has unique constraint on order_id (one payment per order)
- [ ] MercadoPagoWebhook has unique constraint on external_id
- [ ] All repositories implement contracts
- [ ] MercadoPagoService uses HTTP client (not hardcoded curl)
- [ ] API keys stored in config, accessed via env
- [ ] Factories for all models with relevant states
- [ ] Migrations with proper indexes and foreign keys
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Livewire, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **HTTP Controllers:**
   - `WebhookController` (public endpoint for Mercado Pago webhooks)
     - `POST /api/v1/payments/webhooks/mercadopago`
     - No authentication (public endpoint)
     - Rate limiting (100/min)
     - Signature verification
     - Idempotent processing
     - Returns 200 OK always (don't reveal processing errors)

2. **API Routes (routes/api.php):**
   - `POST /api/v1/payments/webhooks/mercadopago` → WebhookController@mercadoPago

3. **Filament Resources (Backoffice):**
   - `PaymentResource` (view-only, limited manual updates)
     - **List Page:**
       - Table: id, order_number, method badge, status badge, amount, created_at
       - Filters: status, method, date range, manual confirmation
       - Sorting: created_at desc by default
     - **View Page:**
       - Payment details (read-only)
       - Order link
       - Payment link (if Mercado Pago)
       - Status history (audit trail)
       - Transaction history
       - Actions: Confirm (if pending), Refund (if paid)
     - **Actions:**
       - `ConfirmPaymentAction` (modal with confirmation, reason textarea)
       - `RefundPaymentAction` (modal with confirmation, reason textarea, requires admin)
     - **Relation Managers:**
       - StatusHistoryRelationManager (display only, chronological)
       - TransactionHistoryRelationManager (display only)

4. **Filament Widgets:**
   - `PaymentStatsWidget` (total payments, by status, by method, revenue)

5. **Public Pages:**
   - None (no public payment pages, only webhook endpoint)

**Feature Tests:**
- Payment creation
- Manual payment confirmation (merchant)
- Manual refund (merchant)
- Webhook processing (signature verification, happy path)
- Webhook duplicate handling (idempotency)
- Webhook signature failure
- Payment link generation
- Payment link expiration
- Status transition validation
- Smoke tests for Filament resource

**Acceptance Criteria:**
- [ ] Webhook endpoint is public (no auth)
- [ ] Webhook endpoint has rate limiting (100/min)
- [ ] Webhook endpoint verifies signature before processing
- [ ] Webhook endpoint returns 200 OK always
- [ ] Webhook endpoint handles duplicates gracefully
- [ ] Filament resource shows status history
- [ ] Confirm action only available for pending payments
- [ ] Refund action only available for paid payments
- [ ] Feature tests cover all critical paths
- [ ] Smoke tests for Filament resource
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - `PaymentCreated` (paymentId, orderId, method, amount, createdAt)
   - `PaymentStatusChanged` (paymentId, orderId, oldStatus, newStatus, changedBy, isWebhookUpdate, changedAt)
   - `PaymentConfirmed` (paymentId, orderId, method, amount, confirmedAt)
   - `PaymentRefunded` (paymentId, orderId, amount, refundedBy, refundedAt)

2. **Listeners:**
   - `UpdateOrderPaymentStatus` (listens to `PaymentStatusChanged`, `PaymentConfirmed`, `PaymentRefunded`)
     - Call Orders::ChangePaymentStatusInterface
     - Update order payment status accordingly
   - `SendPaymentConfirmationNotification` (listens to `PaymentConfirmed`)
     - Only if status = PAID
     - Dispatch job to send WhatsApp message to merchant
     - Message template: "Payment confirmed for order {orderNumber} - Amount: {amount}"
   - `SendPaymentRefundedNotification` (listens to `PaymentRefunded`)
     - Dispatch job to send WhatsApp message to merchant
     - Message template: "Payment refunded for order {orderNumber} - Amount: {amount}"

3. **Jobs:**
   - None directly in Payments module (WhatsApp jobs handled by WhatsApp module)

**Event Tests:**
- Event dispatching on payment creation
- Event dispatching on status changes
- Listener execution (mock jobs)
- Order payment status update triggered correctly
- Idempotency of listeners

**Acceptance Criteria:**
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Listeners implement `ShouldQueue` for async processing
- [ ] UpdateOrderPaymentStatus calls Orders module interface
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
- [ ] Feature tests for webhook processing (signature, duplicates)
- [ ] Feature tests for manual confirmation/refund
- [ ] Integration tests for database operations
- [ ] Smoke tests for Filament resource
- [ ] 100% coverage for critical paths (webhook processing, status transitions)

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex business rules have inline comments
- [ ] README.md in module root (overview, setup, usage)

### Database
- [ ] Migrations have proper indexes
- [ ] Foreign keys with appropriate constraints
- [ ] Unique constraints (order_id, internal_reference, external_id)
- [ ] No updates/deletes on status logs
- [ ] Factories for all models with states

### Security
- [ ] Webhook signature verification implemented
- [ ] API keys stored in config (never hardcoded)
- [ ] Rate limiting on webhook endpoint
- [ ] Idempotent webhook processing
- [ ] No sensitive data in logs

### Performance
- [ ] N+1 queries avoided (eager loading)
- [ ] Indexes on foreign keys and filter fields
- [ ] Webhook processing fast (<100ms)

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Skip webhook signature verification (security risk)
- Process duplicate webhooks (idempotency required)
- Hardcode API keys (use config + env)
- Return processing errors in webhook response (always 200 OK)
- Allow automated refunds (manual only for MVP)
- Update or delete status logs (immutable)
- Use float for money (use integer cents with 2 decimal precision)
- Allow payment status changes without audit logs
- Expose internal errors to external webhooks

✅ **DO:**
- Verify webhook signatures before processing
- Check for duplicate webhooks via external_id
- Store webhook payloads for debugging
- Return 200 OK from webhook endpoint always
- Create audit logs for all status changes
- Use integer cents for money values
- Handle webhook processing errors gracefully
- Emit events for all significant domain changes
- Store API keys in config/env
- Validate state transitions with enum methods

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Payments created for orders automatically
   - Mercado Pago links generated correctly
   - Webhooks processed idempotently
   - Webhook signatures verified
   - Merchants can confirm payments manually
   - Merchants can refund payments manually
   - Payment status updates trigger order updates
   - WhatsApp notifications sent on confirmation/refund
   - Payment links expire after 24h
   - Status history fully audited

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - No N+1 queries
   - Proper indexes on all foreign keys
   - Webhook processing completes in < 100ms
   - Payment link generation completes in < 500ms

4. **Security:**
   - Webhook signatures verified
   - Rate limiting enforced (100/min)
   - API keys never exposed
   - Duplicate webhooks handled
   - Status logs immutable

---

## Configuration

### Environment Variables

```env
# Mercado Pago Configuration
MERCADOPAGO_PUBLIC_KEY=your_public_key
MERCADOPAGO_ACCESS_TOKEN=your_access_token
MERCADOPAGO_WEBHOOK_SECRET=your_webhook_secret

# Payment Configuration
PAYMENT_LINK_EXPIRATION_HOURS=24
PAYMENT_WEBHOOK_RATE_LIMIT=100
```

### Config File: `config/payments.php`

```php
return [
    'mercado_pago' => [
        'public_key' => env('MERCADOPAGO_PUBLIC_KEY'),
        'access_token' => env('MERCADOPAGO_ACCESS_TOKEN'),
        'webhook_secret' => env('MERCADOPAGO_WEBHOOK_SECRET'),
    ],
    'link_expiration_hours' => env('PAYMENT_LINK_EXPIRATION_HOURS', 24),
    'webhook_rate_limit' => env('PAYMENT_WEBHOOK_RATE_LIMIT', 100),
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
./vendor/bin/sail test Modules/Payments

# Test with coverage
./vendor/bin/sail test --coverage
```

---

## Dependencies and Integration

### Modules that Depend on Payments

- **Reports**: Reads Payment data for financial reports

### External Dependencies

- **Orders**: Create payment, update order payment status
- **Auth**: Validate merchant permissions
- **Security**: Rate limiting on webhooks
- **WhatsApp**: Send payment confirmation/refund notifications

### External Services

- **Mercado Pago API**: Payment link generation, webhook processing

---

## Final Notes

- This module is **phase-critical** for Phase 3 (Integraciones).
- **Webhook security is CRITICAL**: verify signatures, handle duplicates.
- **Idempotency is MANDATORY**: use external_id for duplicate detection.
- **Status logs are IMMUTABLE**: no updates or deletes allowed.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 16-20 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
