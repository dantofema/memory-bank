---
task_id: "whatsapp-003-persistence"
module: "WhatsApp"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "WhatsApp - Modelos, Repositorios y Persistencia"
priority: "HIGH"
estimated_time: "5 hours"
dependencies:
  - "whatsapp-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/whatsapp/domain_model.md"
  - "@laravel/conventions/conventions.md"
phase: "Fase 2 - Persistencia"
---

# Task 003: Models, Repositories and Persistence

## Objective

Implement the persistence layer for the WhatsApp module: Eloquent models, repositories, migrations, factories, and seeders. This layer handles database storage and retrieval of WhatsApp messages.

## Context

This task implements the data access layer using Laravel's Eloquent ORM. The `WhatsAppMessage` model uses custom casts from Task 001 to store Value Objects and Enums. The repository pattern abstracts database queries.

## References

- **Agents Prompt:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#task-003`
- **Domain Model:** `@e-commerce-wa-ml/whatsapp/domain_model.md`
- **Persistence Guide:** `@laravel/agents/agent-c-persistence.md`

## Deliverables

### 1. Eloquent Model

#### WhatsAppMessage

**Location:** `Modules/WhatsApp/Models/WhatsAppMessage.php`

**Table:** `whatsapp_messages`

**Properties:**
```php
final class WhatsAppMessage extends Model
{
    use HasFactory, HasUlids;

    protected $fillable = [
        'recipient',
        'template',
        'context',
        'status',
        'attempt_count',
        'scheduled_at',
        'sent_at',
        'failed_at',
        'failure_reason',
    ];

    protected $casts = [
        'recipient' => PhoneNumberCast::class,
        'template' => MessageTemplateCast::class,
        'context' => MessageContextCast::class,
        'status' => MessageStatusCast::class,
        'attempt_count' => 'integer',
        'scheduled_at' => 'datetime',
        'sent_at' => 'datetime',
        'failed_at' => 'datetime',
    ];
}
```

**Attributes:**
- `id` (ULID) - Primary key
- `recipient` (PhoneNumber VO) - Destination phone number
- `template` (MessageTemplate enum) - Message template
- `context` (MessageContext VO) - Template context data
- `status` (MessageStatus enum) - Current message status
- `attempt_count` (int) - Number of send attempts
- `scheduled_at` (datetime) - When to send
- `sent_at` (datetime) - When successfully sent
- `failed_at` (datetime) - When last failed
- `failure_reason` (string) - Reason for failure
- `created_at` (datetime) - Creation timestamp
- `updated_at` (datetime) - Last update timestamp

**Query Scopes:**

```php
public function scopePending(Builder $query): void
{
    $query->where('status', MessageStatus::PENDING)
          ->where(fn($q) => $q->whereNull('scheduled_at')
                             ->orWhere('scheduled_at', '<=', now()));
}

public function scopeFailed(Builder $query): void
{
    $query->where('status', MessageStatus::FAILED);
}

public function scopeSent(Builder $query): void
{
    $query->where('status', MessageStatus::SENT);
}

public function scopeRetryable(Builder $query, int $maxAttempts = 3): void
{
    $query->where('status', MessageStatus::FAILED)
          ->where('attempt_count', '<', $maxAttempts);
}

public function scopeDiscarded(Builder $query): void
{
    $query->where('status', MessageStatus::DISCARDED);
}
```

**Business Methods:**

```php
public function isReadyToSend(): bool
{
    return $this->status === MessageStatus::PENDING
        && ($this->scheduled_at === null || $this->scheduled_at <= now());
}

public function canRetry(int $maxAttempts = 3): bool
{
    return $this->status === MessageStatus::FAILED
        && $this->attempt_count < $maxAttempts;
}

public function markAsSent(DateTime $sentAt = null): void
{
    $this->status = MessageStatus::SENT;
    $this->sent_at = $sentAt ?? now();
    $this->failure_reason = null;
    $this->save();
}

public function markAsFailed(string $reason): void
{
    $this->attempt_count++;
    $this->status = $this->attempt_count >= 3 
        ? MessageStatus::DISCARDED 
        : MessageStatus::FAILED;
    $this->failed_at = now();
    $this->failure_reason = $reason;
    $this->save();
}

public function incrementAttempt(): void
{
    $this->attempt_count++;
    $this->save();
}
```

---

### 2. Repository

#### WhatsAppMessageRepository

**Location:** `Modules/WhatsApp/Repositories/WhatsAppMessageRepository.php`

**Responsibility:** Data access layer for WhatsAppMessage.

**Methods:**

```php
final class WhatsAppMessageRepository
{
    public function save(WhatsAppMessage $message): void
    {
        $message->save();
    }

    public function findById(string $id): WhatsAppMessage
    {
        return WhatsAppMessage::findOrFail($id);
    }

    public function findPendingMessages(int $limit = 100): Collection
    {
        return WhatsAppMessage::pending()
            ->orderByRaw("CASE WHEN template = ? THEN 1 ELSE 2 END", [MessageTemplate::ORDER_CREATED->value])
            ->orderBy('created_at')
            ->limit($limit)
            ->get();
    }

    public function findFailedMessages(int $maxAttempts = 3): Collection
    {
        return WhatsAppMessage::retryable($maxAttempts)
            ->orderBy('failed_at', 'asc')
            ->get();
    }

    public function findByRecipient(PhoneNumber $phone): Collection
    {
        return WhatsAppMessage::where('recipient', $phone->normalized)
            ->orderBy('created_at', 'desc')
            ->get();
    }

    public function findByStatus(MessageStatus $status): Collection
    {
        return WhatsAppMessage::where('status', $status)
            ->orderBy('created_at', 'desc')
            ->get();
    }

    public function countByStatus(MessageStatus $status): int
    {
        return WhatsAppMessage::where('status', $status)->count();
    }

    public function getStats(): array
    {
        return [
            'total' => WhatsAppMessage::count(),
            'sent' => $this->countByStatus(MessageStatus::SENT),
            'failed' => $this->countByStatus(MessageStatus::FAILED),
            'pending' => $this->countByStatus(MessageStatus::PENDING),
            'discarded' => $this->countByStatus(MessageStatus::DISCARDED),
        ];
    }
}
```

---

### 3. Migration

#### create_whatsapp_messages_table

**Location:** `Modules/WhatsApp/Database/Migrations/xxxx_xx_xx_xxxxxx_create_whatsapp_messages_table.php`

```php
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('whatsapp_messages', function (Blueprint $table) {
            $table->ulid('id')->primary();
            $table->string('recipient', 20)->index();
            $table->string('template', 50)->index();
            $table->jsonb('context');
            $table->string('status', 20)->index();
            $table->unsignedTinyInteger('attempt_count')->default(0);
            $table->timestamp('scheduled_at')->nullable();
            $table->timestamp('sent_at')->nullable()->index();
            $table->timestamp('failed_at')->nullable();
            $table->text('failure_reason')->nullable();
            $table->timestamps();

            // Composite index for retryable queries
            $table->index(['status', 'attempt_count'], 'idx_retryable');
            
            // Index for scheduled messages
            $table->index(['status', 'scheduled_at'], 'idx_scheduled');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('whatsapp_messages');
    }
};
```

**Indexes:**
- `recipient` - Query by phone number
- `template` - Filter by template type
- `status` - Filter by status
- `sent_at` - Order by sent timestamp
- `(status, attempt_count)` - Retryable queries
- `(status, scheduled_at)` - Scheduled messages

---

### 4. Factory

#### WhatsAppMessageFactory

**Location:** `Modules/WhatsApp/Database/Factories/WhatsAppMessageFactory.php`

```php
final class WhatsAppMessageFactory extends Factory
{
    protected $model = WhatsAppMessage::class;

    public function definition(): array
    {
        return [
            'recipient' => new PhoneNumber('+5491' . $this->faker->numerify('#########')),
            'template' => $this->faker->randomElement(MessageTemplate::cases()),
            'context' => new MessageContext(
                orderNumber: 'ORD-' . $this->faker->numerify('######'),
                totalAmount: Money::of($this->faker->randomFloat(2, 100, 5000), 'ARS'),
                customerName: $this->faker->name(),
                customerPhone: new PhoneNumber('+5491' . $this->faker->numerify('#########')),
                deliveryAddress: $this->faker->address(),
                paymentMethod: 'Efectivo',
                orderStatus: 'pending',
                paymentStatus: 'pending',
                items: [
                    [
                        'name' => $this->faker->word(),
                        'quantity' => $this->faker->numberBetween(1, 5),
                        'price' => $this->faker->randomFloat(2, 50, 1000),
                    ],
                ],
                observations: $this->faker->optional()->sentence(),
            ),
            'status' => MessageStatus::PENDING,
            'attempt_count' => 0,
            'scheduled_at' => null,
            'sent_at' => null,
            'failed_at' => null,
            'failure_reason' => null,
        ];
    }

    public function pending(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::PENDING,
            'attempt_count' => 0,
        ]);
    }

    public function queued(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::QUEUED,
        ]);
    }

    public function sending(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::SENDING,
        ]);
    }

    public function sent(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::SENT,
            'sent_at' => now(),
        ]);
    }

    public function failed(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::FAILED,
            'attempt_count' => $this->faker->numberBetween(1, 2),
            'failed_at' => now(),
            'failure_reason' => 'Network timeout',
        ]);
    }

    public function discarded(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => MessageStatus::DISCARDED,
            'attempt_count' => 3,
            'failed_at' => now(),
            'failure_reason' => 'Max attempts reached',
        ]);
    }

    public function withAttempts(int $count): static
    {
        return $this->state(fn (array $attributes) => [
            'attempt_count' => $count,
        ]);
    }

    public function forTemplate(MessageTemplate $template): static
    {
        return $this->state(fn (array $attributes) => [
            'template' => $template,
        ]);
    }

    public function forRecipient(PhoneNumber $phone): static
    {
        return $this->state(fn (array $attributes) => [
            'recipient' => $phone,
        ]);
    }

    public function scheduled(DateTime $scheduledAt): static
    {
        return $this->state(fn (array $attributes) => [
            'scheduled_at' => $scheduledAt,
        ]);
    }
}
```

---

### 5. Seeder

#### WhatsAppSeeder

**Location:** `Modules/WhatsApp/Database/Seeders/WhatsAppSeeder.php`

```php
final class WhatsAppSeeder extends Seeder
{
    public function run(): void
    {
        // Sample messages for testing
        WhatsAppMessage::factory()
            ->sent()
            ->count(10)
            ->create();

        WhatsAppMessage::factory()
            ->pending()
            ->count(5)
            ->create();

        WhatsAppMessage::factory()
            ->failed()
            ->count(3)
            ->create();

        WhatsAppMessage::factory()
            ->discarded()
            ->count(2)
            ->create();

        // Specific template examples
        WhatsAppMessage::factory()
            ->pending()
            ->forTemplate(MessageTemplate::ORDER_CREATED)
            ->create();

        WhatsAppMessage::factory()
            ->sent()
            ->forTemplate(MessageTemplate::PAYMENT_CONFIRMED)
            ->create();
    }
}
```

---

## Integration Tests

Integration tests use the real database (via RefreshDatabase trait).

### Test Files

```
Modules/WhatsApp/Tests/Feature/
└── Persistence/
    ├── WhatsAppMessageModelTest.php
    ├── WhatsAppMessageRepositoryTest.php
    └── WhatsAppMessageFactoryTest.php
```

### Test Coverage Requirements

**WhatsAppMessageModelTest:**
- Creates message with valid data
- Uses custom casts correctly
- Scopes filter correctly (pending, failed, retryable)
- `isReadyToSend()` logic
- `canRetry()` logic
- `markAsSent()` updates status and timestamp
- `markAsFailed()` increments attempts and updates status
- Transitions to DISCARDED after 3 failures

**WhatsAppMessageRepositoryTest:**
- `save()` persists message
- `findById()` retrieves message
- `findPendingMessages()` returns pending with priority order
- `findFailedMessages()` returns retryable only
- `findByRecipient()` filters by phone
- `findByStatus()` filters correctly
- `getStats()` returns correct counts

**WhatsAppMessageFactoryTest:**
- Factory creates valid message
- States work correctly (pending, sent, failed, etc.)
- `withAttempts()` sets attempt count
- `forTemplate()` sets template
- `forRecipient()` sets recipient

---

## Validation Commands

```bash
# Run integration tests
./vendor/bin/sail test --filter=WhatsApp/Tests/Feature/Persistence

# Run migrations
./vendor/bin/sail artisan migrate:fresh

# Static analysis
./vendor/bin/sail composer run phpstan -- --paths=Modules/WhatsApp/Models,Modules/WhatsApp/Repositories,Modules/WhatsApp/Database

# Code formatting
./vendor/bin/sail bin pint Modules/WhatsApp/Models Modules/WhatsApp/Repositories Modules/WhatsApp/Database
```

## Acceptance Criteria

- [ ] All models use Eloquent Casts for VOs and Enums
- [ ] WhatsAppMessage model has scopes for filtering
- [ ] Repository implements all query methods
- [ ] Factory has states for all MessageStatus values
- [ ] Migration has proper indexes
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## Definition of Done

- [ ] All deliverables implemented
- [ ] All integration tests passing
- [ ] PHPStan level 6+ passes without errors
- [ ] Pint formatting applied
- [ ] Migration runs successfully
- [ ] Factory creates valid test data
- [ ] Code reviewed and approved
- [ ] Documentation complete with PHPDoc blocks

---

**Status:** Pending  
**Assignee:** Agente C  
**Due Date:** TBD
