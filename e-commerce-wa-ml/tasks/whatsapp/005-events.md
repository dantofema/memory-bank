# Task 005: Events, Listeners and Jobs

**Agent:** Agente E - Events, Listeners y Jobs  
**Module:** WhatsApp  
**Priority:** HIGH  
**Estimated Time:** 6 hours  
**Dependencies:** 001-contracts, 002-actions, 003-persistence

## Objective

Implement event listeners that react to domain events from Orders and Payments modules, and background jobs for asynchronous message sending.

## Context

This task implements the event-driven architecture. Listeners subscribe to events from other modules and create WhatsApp notifications. Jobs handle asynchronous message sending with retry logic.

## References

- **Agents Prompt:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#task-005`
- **Domain Model:** `@e-commerce-wa-ml/whatsapp/domain_model.md`
- **Events Guide:** `@laravel/agents/agent-e-events.md`

## Deliverables

### 1. Event Listeners

All listeners must implement `ShouldQueue` for asynchronous processing.

#### 1.1 OrderCreatedListener

**Location:** `Modules/WhatsApp/Listeners/OrderCreatedListener.php`

**Listens to:** `Modules\Orders\Events\OrderCreated`

**Queue:** `whatsapp-listeners`

**Dependencies:**
- `WhatsAppMessageService` - Create message
- `WhatsAppConfig` - Get merchant phone

**Logic:**
```php
final class OrderCreatedListener implements ShouldQueue
{
    public string $queue = 'whatsapp-listeners';

    public function __construct(
        private readonly WhatsAppMessageService $messageService,
        private readonly WhatsAppConfig $config,
    ) {}

    public function handle(OrderCreated $event): void
    {
        if (!$this->config->isEnabled()) {
            return;
        }

        $context = $this->buildContextFromOrder($event->order);
        
        $this->messageService->createMessage(
            template: MessageTemplate::ORDER_CREATED,
            context: $context,
            recipient: $this->config->merchantPhone
        );
        
        // Message is automatically queued for async sending
    }

    private function buildContextFromOrder(Order $order): MessageContext
    {
        return new MessageContext(
            orderNumber: $order->number,
            totalAmount: $order->total,
            customerName: $order->customer_name,
            customerPhone: $order->customer_phone,
            deliveryAddress: $order->delivery_address,
            paymentMethod: $order->payment_method->getLabel(),
            orderStatus: $order->status->value,
            paymentStatus: $order->payment_status->value,
            items: $order->items->map(fn($item) => [
                'name' => $item->product_name,
                'quantity' => $item->quantity,
                'price' => $item->price->getAmount(),
            ])->toArray(),
            observations: $order->observations,
        );
    }
}
```

**Registration in EventServiceProvider:**
```php
protected $listen = [
    OrderCreated::class => [
        OrderCreatedListener::class,
    ],
];
```

---

#### 1.2 OrderStatusChangedListener

**Location:** `Modules/WhatsApp/Listeners/OrderStatusChangedListener.php`

**Listens to:** `Modules\Orders\Events\OrderStatusChanged`

**Queue:** `whatsapp-listeners`

**Logic:**
```php
final class OrderStatusChangedListener implements ShouldQueue
{
    public string $queue = 'whatsapp-listeners';

    private const NOTIFIABLE_STATUSES = [
        OrderStatus::CONFIRMED,
        OrderStatus::IN_DELIVERY,
        OrderStatus::DELIVERED,
        OrderStatus::CANCELLED,
        OrderStatus::REJECTED,
    ];

    public function handle(OrderStatusChanged $event): void
    {
        if (!$this->shouldNotify($event->newStatus)) {
            return;
        }

        $template = $this->getTemplateForStatus($event->newStatus);
        $context = $this->buildContextFromOrder($event->order);
        
        $this->messageService->createMessage(
            template: $template,
            context: $context,
            recipient: $this->config->merchantPhone
        );
    }

    private function shouldNotify(OrderStatus $status): bool
    {
        return in_array($status, self::NOTIFIABLE_STATUSES);
    }

    private function getTemplateForStatus(OrderStatus $status): MessageTemplate
    {
        return match($status) {
            OrderStatus::CONFIRMED => MessageTemplate::ORDER_CONFIRMED,
            OrderStatus::IN_DELIVERY => MessageTemplate::ORDER_IN_DELIVERY,
            OrderStatus::DELIVERED => MessageTemplate::ORDER_DELIVERED,
            OrderStatus::CANCELLED => MessageTemplate::ORDER_CANCELLED,
            OrderStatus::REJECTED => MessageTemplate::ORDER_REJECTED,
            default => throw new InvalidArgumentException("Unsupported status: {$status->value}"),
        };
    }
}
```

---

#### 1.3 PaymentConfirmedListener

**Location:** `Modules/WhatsApp/Listeners/PaymentConfirmedListener.php`

**Listens to:** `Modules\Payments\Events\PaymentConfirmed`

**Queue:** `whatsapp-listeners`

**Logic:**
```php
final class PaymentConfirmedListener implements ShouldQueue
{
    public string $queue = 'whatsapp-listeners';

    public function handle(PaymentConfirmed $event): void
    {
        if (!$this->config->isEnabled()) {
            return;
        }

        $context = new MessageContext(
            orderNumber: $event->payment->order->number,
            totalAmount: $event->payment->amount,
            customerName: $event->payment->order->customer_name,
            customerPhone: $event->payment->order->customer_phone,
            deliveryAddress: $event->payment->order->delivery_address,
            paymentMethod: $event->payment->method->getLabel(),
            orderStatus: $event->payment->order->status->value,
            paymentStatus: 'confirmed',
            items: [],
            observations: null,
        );
        
        $this->messageService->createMessage(
            template: MessageTemplate::PAYMENT_CONFIRMED,
            context: $context,
            recipient: $this->config->merchantPhone
        );
    }
}
```

---

#### 1.4 PaymentRefundedListener

**Location:** `Modules/WhatsApp/Listeners/PaymentRefundedListener.php`

**Listens to:** `Modules\Payments\Events\PaymentRefunded`

**Queue:** `whatsapp-listeners`

**Logic:**
```php
final class PaymentRefundedListener implements ShouldQueue
{
    public string $queue = 'whatsapp-listeners';

    public function handle(PaymentRefunded $event): void
    {
        if (!$this->config->isEnabled()) {
            return;
        }

        $context = $this->buildContextFromPayment($event->payment);
        
        $this->messageService->createMessage(
            template: MessageTemplate::PAYMENT_REFUNDED,
            context: $context,
            recipient: $this->config->merchantPhone
        );
    }
}
```

---

### 2. Background Jobs

#### 2.1 SendWhatsAppMessageJob

**Location:** `Modules/WhatsApp/Jobs/SendWhatsAppMessageJob.php`

**Queue:** `whatsapp`

**Configuration:**
- `tries = 3` - Max 3 attempts
- `backoff = [60, 300, 900]` - 1min, 5min, 15min
- `timeout = 30` - 30 seconds max execution

**Dependencies:**
- `WhatsAppMessageService` - Send message
- `SendMessageAction` - Execute sending

**Logic:**
```php
final class SendWhatsAppMessageJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public array $backoff = [60, 300, 900]; // 1min, 5min, 15min
    public int $timeout = 30;
    public string $queue = 'whatsapp';

    public function __construct(
        private readonly string $messageId,
    ) {}

    public function handle(WhatsAppMessageService $messageService): void
    {
        try {
            $result = $messageService->sendMessage($this->messageId);
            
            if ($result->isFailed()) {
                throw new MessageSendFailedException($result->getErrorMessage());
            }
            
            Log::info('WhatsApp message sent successfully', [
                'message_id' => $this->messageId,
                'attempt' => $this->attempts(),
            ]);
            
        } catch (Throwable $e) {
            Log::error('WhatsApp message send failed', [
                'message_id' => $this->messageId,
                'attempt' => $this->attempts(),
                'error' => $e->getMessage(),
            ]);
            
            throw $e; // Re-throw to trigger retry
        }
    }

    public function failed(Throwable $exception): void
    {
        Log::error('WhatsApp message send permanently failed', [
            'message_id' => $this->messageId,
            'attempts' => $this->attempts(),
            'error' => $exception->getMessage(),
        ]);
        
        // Mark message as failed (will be DISCARDED if max attempts reached)
        $messageService = app(WhatsAppMessageService::class);
        $message = $messageService->getMessage($this->messageId);
        
        app(MarkMessageAsFailedAction::class)->execute(
            $message,
            $exception->getMessage()
        );
    }

    public function retryUntil(): DateTime
    {
        return now()->addHour(); // Stop retrying after 1 hour
    }
}
```

---

### 3. Event Registration

Update `EventServiceProvider`:

```php
protected $listen = [
    // Orders Module Events
    OrderCreated::class => [
        OrderCreatedListener::class,
    ],
    OrderStatusChanged::class => [
        OrderStatusChangedListener::class,
    ],
    
    // Payments Module Events
    PaymentConfirmed::class => [
        PaymentConfirmedListener::class,
    ],
    PaymentRefunded::class => [
        PaymentRefundedListener::class,
    ],
];
```

---

### 4. Queue Configuration

Add to `config/queue.php`:

```php
'connections' => [
    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
    ],
],

'whatsapp' => [
    'driver' => 'database',
    'table' => 'jobs',
    'queue' => 'whatsapp',
    'retry_after' => 90,
],
```

---

## Event Tests

### Test Files

```
Modules/WhatsApp/Tests/Feature/
└── Listeners/
    ├── OrderCreatedListenerTest.php
    ├── OrderStatusChangedListenerTest.php
    ├── PaymentConfirmedListenerTest.php
    └── PaymentRefundedListenerTest.php

Modules/WhatsApp/Tests/Feature/
└── Jobs/
    └── SendWhatsAppMessageJobTest.php
```

### Test Coverage

**OrderCreatedListenerTest:**
- Listener creates WhatsApp message on event
- Message has correct template (ORDER_CREATED)
- Message context contains all order data
- Listener skipped if module disabled

**OrderStatusChangedListenerTest:**
- Listener creates message for notifiable statuses
- Listener skips non-notifiable statuses
- Correct template selected for each status

**PaymentConfirmedListenerTest:**
- Listener creates message on payment confirmed
- Message has PAYMENT_CONFIRMED template
- Context contains payment data

**SendWhatsAppMessageJobTest:**
- Job sends message successfully
- Job marks message as SENT on success
- Job marks message as FAILED on error
- Job retries with exponential backoff
- Job stops after 3 attempts
- failed() method called on final failure

---

## Validation Commands

```bash
# Run event tests
./vendor/bin/sail test --filter=WhatsApp/Tests/Feature/Listeners
./vendor/bin/sail test --filter=WhatsApp/Tests/Feature/Jobs

# Process queue manually
./vendor/bin/sail artisan queue:work --queue=whatsapp

# Static analysis
./vendor/bin/sail composer run phpstan -- --paths=Modules/WhatsApp/Listeners,Modules/WhatsApp/Jobs

# Code formatting
./vendor/bin/sail bin pint Modules/WhatsApp/Listeners Modules/WhatsApp/Jobs
```

## Acceptance Criteria

- [ ] All listeners are `final`
- [ ] Listeners implement `ShouldQueue` for async processing
- [ ] OrderStatusChangedListener filters notifiable statuses
- [ ] SendWhatsAppMessageJob has correct backoff configuration
- [ ] SendWhatsAppMessageJob updates message status correctly
- [ ] Job failed() method marks message as FAILED
- [ ] Event tests verify message creation
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## Definition of Done

- [ ] All deliverables implemented
- [ ] All event tests passing
- [ ] PHPStan level 6+ passes without errors
- [ ] Pint formatting applied
- [ ] Events registered in EventServiceProvider
- [ ] Queue configured correctly
- [ ] Code reviewed and approved
- [ ] Documentation complete

---

**Status:** Pending  
**Assignee:** Agente E  
**Due Date:** TBD
