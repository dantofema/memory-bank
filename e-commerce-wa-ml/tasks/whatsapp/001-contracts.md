# Task 001: Contracts, Data Objects, Value Objects and Enums

**Agent:** Agente A - Contratos, Data, VOs y Enums  
**Module:** WhatsApp  
**Priority:** HIGH  
**Estimated Time:** 6 hours  
**Dependencies:** None

## Objective

Implement the foundation layer of the WhatsApp module: Value Objects, Enums, Custom Casts, Data Objects, and Contract interfaces. This layer defines the domain primitives and data structures used throughout the module.

## Context

This is the foundational task for the WhatsApp module. It establishes:
- Type-safe representations of domain concepts (VOs)
- Enums for message templates and statuses
- Data Transfer Objects using Spatie Laravel Data
- Contract interfaces for Actions
- Custom Eloquent Casts for VOs

All VOs must be `final readonly` and implement `Wireable` for Livewire/Volt compatibility. All classes must use strict types and pass PHPStan level 6+.

## References

- **Agents Prompt:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#task-001`
- **Domain Model:** `@e-commerce-wa-ml/whatsapp/domain_model.md`
- **Value Objects Guide:** `@laravel/conventions/value-objects.md`
- **Project Definition:** `@e-commerce-wa-ml/project_definition.md`

## Deliverables

### 1. Value Objects

All Value Objects must be:
- `final readonly` classes
- Implement `Wireable` interface
- Have strict type declarations
- Include validation in constructor
- Include `equals()` method for comparison
- Include factory methods where appropriate

#### 1.1 PhoneNumber

**Location:** `Modules/WhatsApp/ValueObjects/PhoneNumber.php`

**Properties:**
```php
public string $countryCode;  // e.g., "+54"
public string $number;        // e.g., "9111234567"
public string $normalized;    // e.g., "+5491112345678" (E.164)
```

**Methods:**
- `__construct(string $phone)` - Parse and validate phone number
- `toString(): string` - Returns normalized format
- `toInternational(): string` - Returns with country code prefix
- `equals(PhoneNumber $other): bool` - Compare two phone numbers
- `isValid(): bool` - Check if phone number is valid
- `toLivewire(): string` - Wireable implementation
- `static fromLivewire(mixed $value): static` - Wireable implementation

**Validation Rules:**
- Must follow E.164 format
- Length between 8 and 15 digits (excluding country code)
- Only numeric characters (no spaces, hyphens, parentheses)
- Country code required (defaults to +54 for Argentina if not provided)

**Example:**
```php
$phone = new PhoneNumber('+5491112345678');
$phone->normalized; // "+5491112345678"
$phone->toInternational(); // "+54 9 11 1234-5678"
```

---

#### 1.2 MessageContext

**Location:** `Modules/WhatsApp/ValueObjects/MessageContext.php`

**Properties:**
```php
public readonly string $orderNumber;
public readonly Money $totalAmount;
public readonly string $customerName;
public readonly PhoneNumber $customerPhone;
public readonly string $deliveryAddress;
public readonly string $paymentMethod;
public readonly string $orderStatus;
public readonly string $paymentStatus;
public readonly array $items;  // Array of OrderItem data
public readonly ?string $observations;
```

**Methods:**
- `__construct(...)` - Validate required fields
- `toArray(): array` - Convert to array for template interpolation
- `fillTemplate(string $template): string` - Fill a template with context data
- `validate(MessageTemplate $template): bool` - Check if context has required fields
- `equals(MessageContext $other): bool` - Compare contexts
- `toLivewire(): array` - Wireable implementation
- `static fromLivewire(mixed $value): static` - Wireable implementation

**Validation Rules:**
- `orderNumber` cannot be empty
- `totalAmount` must be positive
- `customerPhone` must be valid PhoneNumber
- `items` cannot be empty for ORDER_CREATED template
- `deliveryAddress` cannot be empty for ORDER_CREATED

---

#### 1.3 SendResult

**Location:** `Modules/WhatsApp/ValueObjects/SendResult.php`

**Properties:**
```php
public readonly bool $success;
public readonly ?string $messageId;
public readonly ?string $errorMessage;
public readonly DateTime $timestamp;
```

**Methods:**
- `__construct(bool $success, ?string $messageId, ?string $errorMessage)` - Create result
- `static success(string $messageId): self` - Factory for success
- `static failure(string $errorMessage): self` - Factory for failure
- `isSuccess(): bool` - Check if successful
- `isFailed(): bool` - Check if failed
- `getErrorMessage(): ?string` - Get error message
- `equals(SendResult $other): bool` - Compare results
- `toLivewire(): array` - Wireable implementation
- `static fromLivewire(mixed $value): static` - Wireable implementation

---

#### 1.4 WhatsAppConfig

**Location:** `Modules/WhatsApp/ValueObjects/WhatsAppConfig.php`

**Properties:**
```php
public readonly string $gateway;             // 'wa_me' | 'business_api'
public readonly PhoneNumber $merchantPhone;
public readonly bool $enabled;
public readonly int $maxAttempts;
public readonly int $retryDelay;            // seconds
public readonly string $queuePriority;       // 'high' | 'normal'
public readonly bool $sendToCustomers;
```

**Methods:**
- `static fromConfig(): self` - Load from config/whatsapp.php
- `getGateway(): string` - Get gateway type
- `isEnabled(): bool` - Check if module is enabled
- `shouldSendToCustomers(): bool` - Check if customer messages enabled
- `getMaxAttempts(): int` - Get max retry attempts
- `getRetryDelay(int $attemptNumber): int` - Get backoff delay for attempt
- `equals(WhatsAppConfig $other): bool` - Compare configs
- `toLivewire(): array` - Wireable implementation
- `static fromLivewire(mixed $value): static` - Wireable implementation

**Backoff Calculation:**
- Attempt 1: 60 seconds (1 min)
- Attempt 2: 300 seconds (5 min)
- Attempt 3: 900 seconds (15 min)

---

### 2. Enums

All Enums must use PHP 8.1+ enum syntax with `string` backing type.

#### 2.1 MessageTemplate

**Location:** `Modules/WhatsApp/Enums/MessageTemplate.php`

**Values:**
```php
enum MessageTemplate: string
{
    case ORDER_CREATED = 'order_created';
    case ORDER_CONFIRMED = 'order_confirmed';
    case ORDER_IN_DELIVERY = 'order_in_delivery';
    case ORDER_DELIVERED = 'order_delivered';
    case ORDER_CANCELLED = 'order_cancelled';
    case ORDER_REJECTED = 'order_rejected';
    case PAYMENT_CONFIRMED = 'payment_confirmed';
    case PAYMENT_REFUNDED = 'payment_refunded';
}
```

**Methods:**
- `getTemplateName(): string` - Human-readable name
- `getTemplateContent(): string` - Template text content
- `getRequiredFields(): array` - Required context fields
- `getLabel(): string` - For Filament display
- `getColor(): string` - Badge color for Filament

**Template Content:**

See `@e-commerce-wa-ml/whatsapp/agents_prompt.md#message-templates` for full template strings.

**Required Fields Mapping:**
```php
public function getRequiredFields(): array
{
    return match($this) {
        self::ORDER_CREATED => [
            'orderNumber', 'customerName', 'customerPhone',
            'deliveryAddress', 'items', 'totalAmount', 'paymentMethod'
        ],
        self::ORDER_CONFIRMED => [
            'orderNumber', 'totalAmount', 'deliveryAddress'
        ],
        self::PAYMENT_CONFIRMED => [
            'orderNumber', 'totalAmount'
        ],
        // ... etc
    };
}
```

---

#### 2.2 MessageStatus

**Location:** `Modules/WhatsApp/Enums/MessageStatus.php`

**Values:**
```php
enum MessageStatus: string
{
    case PENDING = 'pending';
    case QUEUED = 'queued';
    case SENDING = 'sending';
    case SENT = 'sent';
    case FAILED = 'failed';
    case DISCARDED = 'discarded';
}
```

**Methods:**
- `isPending(): bool` - Check if pending
- `isFinal(): bool` - Check if final state (SENT or DISCARDED)
- `canTransitionTo(MessageStatus $newStatus): bool` - Validate state transition
- `isRetryable(): bool` - Check if can be retried
- `getLabel(): string` - For Filament display
- `getColor(): string` - Badge color for Filament
- `getIcon(): string` - Icon for Filament

**State Transition Rules:**
```php
public function canTransitionTo(MessageStatus $newStatus): bool
{
    return match($this) {
        self::PENDING => in_array($newStatus, [self::QUEUED, self::FAILED, self::DISCARDED]),
        self::QUEUED => in_array($newStatus, [self::SENDING, self::FAILED, self::DISCARDED]),
        self::SENDING => in_array($newStatus, [self::SENT, self::FAILED]),
        self::FAILED => in_array($newStatus, [self::PENDING, self::DISCARDED]),
        self::SENT => false,    // Final state
        self::DISCARDED => false,  // Final state
    };
}
```

---

### 3. Custom Casts

All custom casts must implement Laravel's `CastsAttributes` interface.

#### 3.1 PhoneNumberCast

**Location:** `Modules/WhatsApp/Casts/PhoneNumberCast.php`

Casts between database string and `PhoneNumber` Value Object.

```php
public function get(Model $model, string $key, mixed $value, array $attributes): ?PhoneNumber
{
    return $value ? new PhoneNumber($value) : null;
}

public function set(Model $model, string $key, mixed $value, array $attributes): ?string
{
    return $value instanceof PhoneNumber ? $value->normalized : null;
}
```

---

#### 3.2 MessageContextCast

**Location:** `Modules/WhatsApp/Casts/MessageContextCast.php`

Casts between database JSON and `MessageContext` Value Object.

Uses `AsArrayObject` cast internally for JSON storage.

---

#### 3.3 MessageTemplateCast

**Location:** `Modules/WhatsApp/Casts/MessageTemplateCast.php`

Casts between database string and `MessageTemplate` Enum.

---

#### 3.4 MessageStatusCast

**Location:** `Modules/WhatsApp/Casts/MessageStatusCast.php`

Casts between database string and `MessageStatus` Enum.

---

### 4. Data Objects (Spatie Laravel Data)

All Data Objects must extend `Spatie\LaravelData\Data`.

#### 4.1 WhatsAppMessageData

**Location:** `Modules/WhatsApp/Contracts/Data/WhatsAppMessageData.php`

```php
final class WhatsAppMessageData extends Data
{
    public function __construct(
        public readonly string $id,
        public readonly PhoneNumber $recipient,
        public readonly MessageTemplate $template,
        public readonly MessageContext $context,
        public readonly MessageStatus $status,
        public readonly int $attemptCount,
        public readonly ?DateTime $scheduledAt,
        public readonly ?DateTime $sentAt,
        public readonly ?DateTime $failedAt,
        public readonly ?string $failureReason,
        public readonly DateTime $createdAt,
        public readonly DateTime $updatedAt,
    ) {}
}
```

---

#### 4.2 MessageContextData

**Location:** `Modules/WhatsApp/Contracts/Data/MessageContextData.php`

```php
final class MessageContextData extends Data
{
    public function __construct(
        public readonly string $orderNumber,
        public readonly Money $totalAmount,
        public readonly string $customerName,
        public readonly PhoneNumber $customerPhone,
        public readonly string $deliveryAddress,
        public readonly string $paymentMethod,
        public readonly string $orderStatus,
        public readonly string $paymentStatus,
        public readonly array $items,
        public readonly ?string $observations,
    ) {}
}
```

---

#### 4.3 SendResultData

**Location:** `Modules/WhatsApp/Contracts/Data/SendResultData.php`

```php
final class SendResultData extends Data
{
    public function __construct(
        public readonly bool $success,
        public readonly ?string $messageId,
        public readonly ?string $errorMessage,
        public readonly DateTime $timestamp,
    ) {}
}
```

---

#### 4.4 MessageStatsData

**Location:** `Modules/WhatsApp/Contracts/Data/MessageStatsData.php`

```php
final class MessageStatsData extends Data
{
    public function __construct(
        public readonly int $totalMessages,
        public readonly int $sentMessages,
        public readonly int $failedMessages,
        public readonly int $pendingMessages,
        public readonly int $discardedMessages,
    ) {}
}
```

---

### 5. Contract Interfaces

All contracts must be interfaces with a single public method.

#### 5.1 Command Contracts

**Location:** `Modules/WhatsApp/Contracts/Commands/`

```php
interface CreateMessageInterface
{
    public function execute(
        MessageTemplate $template,
        MessageContext $context,
        PhoneNumber $recipient
    ): WhatsAppMessageData;
}

interface SendMessageInterface
{
    public function execute(string $messageId): SendResultData;
}

interface RetryFailedMessageInterface
{
    public function execute(string $messageId): WhatsAppMessageData;
}
```

---

#### 5.2 Query Contracts

**Location:** `Modules/WhatsApp/Contracts/Queries/`

```php
interface GetMessageInterface
{
    public function execute(string $messageId): WhatsAppMessageData;
}

interface GetPendingMessagesInterface
{
    public function execute(int $limit = 100): array; // array<WhatsAppMessageData>
}

interface GetFailedMessagesInterface
{
    public function execute(int $maxAttempts = 3): array; // array<WhatsAppMessageData>
}
```

---

## Unit Tests

All VOs, Enums, and Casts must have comprehensive unit tests.

### Test Files

```
Modules/WhatsApp/Tests/Unit/
├── ValueObjects/
│   ├── PhoneNumberTest.php
│   ├── MessageContextTest.php
│   ├── SendResultTest.php
│   └── WhatsAppConfigTest.php
├── Enums/
│   ├── MessageTemplateTest.php
│   └── MessageStatusTest.php
├── Casts/
│   ├── PhoneNumberCastTest.php
│   ├── MessageContextCastTest.php
│   ├── MessageTemplateCastTest.php
│   └── MessageStatusCastTest.php
└── Data/
    ├── WhatsAppMessageDataTest.php
    ├── MessageContextDataTest.php
    ├── SendResultDataTest.php
    └── MessageStatsDataTest.php
```

### Test Coverage Requirements

- **PhoneNumber:**
  - Valid E.164 format parsing
  - Invalid format throws exception
  - Normalization with and without country code
  - Comparison with `equals()`
  - Wireable serialization/deserialization

- **MessageContext:**
  - Validation of required fields per template
  - `fillTemplate()` interpolation
  - `toArray()` conversion
  - Missing fields throw exception

- **MessageTemplate:**
  - `getRequiredFields()` for each template
  - `getTemplateContent()` returns valid template
  - Template interpolation with context

- **MessageStatus:**
  - State transition validation
  - `isFinal()` for SENT and DISCARDED
  - `isRetryable()` logic

- **Casts:**
  - Bidirectional conversion (get/set)
  - Null handling
  - Invalid data throws exception

## Validation Commands

```bash
# Run unit tests
./vendor/bin/sail test --filter=WhatsApp/Tests/Unit

# Static analysis
./vendor/bin/sail composer run phpstan -- --paths=Modules/WhatsApp/ValueObjects,Modules/WhatsApp/Enums,Modules/WhatsApp/Casts,Modules/WhatsApp/Contracts

# Code formatting
./vendor/bin/sail bin pint Modules/WhatsApp/ValueObjects Modules/WhatsApp/Enums Modules/WhatsApp/Casts Modules/WhatsApp/Contracts
```

## Acceptance Criteria

- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] PhoneNumber validates and normalizes to E.164
- [ ] MessageContext validates required fields per template
- [ ] MessageStatus validates state transitions
- [ ] MessageTemplate includes all template strings
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] Unit tests for all VOs with 100% coverage
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## Definition of Done

- [ ] All deliverables implemented
- [ ] All unit tests passing
- [ ] PHPStan level 6+ passes without errors
- [ ] Pint formatting applied
- [ ] Code reviewed and approved
- [ ] Documentation complete with PHPDoc blocks
- [ ] No `@phpstan-ignore` comments

---

**Status:** Pending  
**Assignee:** Agente A  
**Due Date:** TBD
