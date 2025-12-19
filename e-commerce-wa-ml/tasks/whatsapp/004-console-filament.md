# Task 004: Console Commands, Filament and Feature Tests

**Agent:** Agente D - HTTP, Livewire/Volt, Filament y Tests Feature  
**Module:** WhatsApp  
**Priority:** HIGH  
**Estimated Time:** 7 hours  
**Dependencies:** 001-contracts, 002-actions, 003-persistence

## Objective

Implement console commands for queue processing, Filament resources for message monitoring, and comprehensive feature tests. This task provides the user interface and automation for the WhatsApp module.

## Context

This task implements the presentation layer: console commands for cron jobs, Filament admin interface for monitoring messages, and feature tests that verify end-to-end functionality.

## References

- **Agents Prompt:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#task-004`
- **Domain Model:** `@e-commerce-wa-ml/whatsapp/domain_model.md`
- **HTTP/Filament Guide:** `@laravel/agents/agent-d-http.md`

## Deliverables

### 1. Console Commands

#### 1.1 ProcessWhatsAppQueueCommand

**Location:** `Modules/WhatsApp/Console/Commands/ProcessWhatsAppQueueCommand.php`

**Signature:** `whatsapp:process-queue`

**Description:** Process pending WhatsApp messages and queue them for sending.

**Dependencies:**
- `WhatsAppMessageService` - Get pending and queue messages
- `GetPendingMessagesAction` - Query pending
- `SendMessageAsyncAction` - Queue messages

**Logic:**
```php
public function handle(): int
{
    $this->info('Processing WhatsApp message queue...');
    
    $limit = $this->option('limit') ?? 100;
    $stats = $this->messageService->processQueue($limit);
    
    $this->table(
        ['Metric', 'Count'],
        [
            ['Processed', $stats['processed']],
            ['Queued', $stats['queued']],
            ['Failed', $stats['failed']],
        ]
    );
    
    $this->info("Queue processing complete. {$stats['queued']} messages queued.");
    
    return self::SUCCESS;
}
```

**Options:**
- `--limit=N` - Max messages to process (default: 100)

**Schedule:** Every 5 minutes via Laravel Task Scheduler

---

#### 1.2 RetryFailedMessagesCommand

**Location:** `Modules/WhatsApp/Console/Commands/RetryFailedMessagesCommand.php`

**Signature:** `whatsapp:retry-failed`

**Description:** Retry failed messages that haven't reached max attempts.

**Dependencies:**
- `WhatsAppMessageService` - Retry failed messages
- `GetFailedMessagesAction` - Query failed
- `RetryFailedMessageAction` - Retry individual message

**Logic:**
```php
public function handle(): int
{
    $this->info('Retrying failed WhatsApp messages...');
    
    $maxAttempts = config('whatsapp.max_attempts', 3);
    $stats = $this->messageService->retryFailedMessages();
    
    $this->table(
        ['Metric', 'Count'],
        [
            ['Found', $stats['found']],
            ['Retried', $stats['retried']],
            ['Skipped', $stats['skipped']],
        ]
    );
    
    $this->info("Retry complete. {$stats['retried']} messages retried.");
    
    return self::SUCCESS;
}
```

**Schedule:** Every 30 minutes via Laravel Task Scheduler

---

### 2. Scheduled Commands Configuration

Add to `app/Console/Kernel.php`:

```php
protected function schedule(Schedule $schedule): void
{
    // Process WhatsApp queue every 5 minutes
    $schedule->command('whatsapp:process-queue')
        ->everyFiveMinutes()
        ->withoutOverlapping()
        ->runInBackground();
    
    // Retry failed messages every 30 minutes
    $schedule->command('whatsapp:retry-failed')
        ->everyThirtyMinutes()
        ->withoutOverlapping()
        ->runInBackground();
}
```

---

### 3. Filament Resources

#### 3.1 WhatsAppMessageResource

**Location:** `Modules/WhatsApp/Filament/Resources/WhatsAppMessageResource.php`

**Purpose:** View-only resource for monitoring WhatsApp messages.

**Table Configuration:**

```php
public static function table(Table $table): Table
{
    return $table
        ->columns([
            TextColumn::make('id')
                ->label('ID')
                ->searchable()
                ->sortable(),
            
            TextColumn::make('recipient.normalized')
                ->label('Recipient')
                ->searchable(),
            
            BadgeColumn::make('template')
                ->colors([
                    'success' => MessageTemplate::ORDER_CREATED,
                    'primary' => MessageTemplate::ORDER_CONFIRMED,
                    'warning' => MessageTemplate::ORDER_IN_DELIVERY,
                ]),
            
            BadgeColumn::make('status')
                ->colors([
                    'danger' => MessageStatus::FAILED,
                    'success' => MessageStatus::SENT,
                    'warning' => MessageStatus::PENDING,
                    'secondary' => MessageStatus::DISCARDED,
                ]),
            
            TextColumn::make('attempt_count')
                ->label('Attempts')
                ->sortable(),
            
            TextColumn::make('sent_at')
                ->label('Sent At')
                ->dateTime()
                ->sortable(),
            
            TextColumn::make('failed_at')
                ->label('Failed At')
                ->dateTime()
                ->sortable(),
        ])
        ->filters([
            SelectFilter::make('status')
                ->options([
                    MessageStatus::PENDING->value => 'Pending',
                    MessageStatus::SENT->value => 'Sent',
                    MessageStatus::FAILED->value => 'Failed',
                    MessageStatus::DISCARDED->value => 'Discarded',
                ]),
            
            SelectFilter::make('template')
                ->options(MessageTemplate::cases()),
            
            DateFilter::make('created_at'),
        ])
        ->actions([
            Action::make('retry')
                ->visible(fn (WhatsAppMessage $record) => $record->canRetry())
                ->action(fn (WhatsAppMessage $record) => 
                    app(RetryFailedMessageAction::class)->execute($record->id)
                ),
            
            Action::make('view')
                ->url(fn (WhatsAppMessage $record) => 
                    WhatsAppMessageResource::getUrl('view', ['record' => $record])
                ),
        ]);
}
```

**Detail Page:**

```php
public static function form(Form $form): Form
{
    return $form
        ->schema([
            Section::make('Message Details')
                ->schema([
                    TextInput::make('id')->disabled(),
                    TextInput::make('recipient.normalized')->disabled(),
                    Select::make('template')->disabled(),
                    Select::make('status')->disabled(),
                    TextInput::make('attempt_count')->disabled(),
                ]),
            
            Section::make('Context Data')
                ->schema([
                    KeyValue::make('context')
                        ->disabled(),
                ]),
            
            Section::make('Timestamps')
                ->schema([
                    DateTimePicker::make('scheduled_at')->disabled(),
                    DateTimePicker::make('sent_at')->disabled(),
                    DateTimePicker::make('failed_at')->disabled(),
                ]),
            
            Section::make('Failure Details')
                ->visible(fn ($record) => $record->status === MessageStatus::FAILED)
                ->schema([
                    Textarea::make('failure_reason')->disabled(),
                ]),
        ]);
}
```

---

#### 3.2 MessageStatsWidget

**Location:** `Modules/WhatsApp/Filament/Widgets/MessageStatsWidget.php`

**Type:** Stats Overview Widget

```php
protected function getStats(): array
{
    $stats = app(GetMessageStatsAction::class)->execute();
    
    return [
        Stat::make('Total Messages', $stats->totalMessages)
            ->icon('heroicon-o-envelope'),
        
        Stat::make('Sent', $stats->sentMessages)
            ->icon('heroicon-o-check-circle')
            ->color('success'),
        
        Stat::make('Failed', $stats->failedMessages)
            ->icon('heroicon-o-x-circle')
            ->color('danger'),
        
        Stat::make('Pending', $stats->pendingMessages)
            ->icon('heroicon-o-clock')
            ->color('warning'),
        
        Stat::make('Discarded', $stats->discardedMessages)
            ->icon('heroicon-o-trash')
            ->color('secondary'),
    ];
}
```

---

#### 3.3 RecentMessagesWidget

**Location:** `Modules/WhatsApp/Filament/Widgets/RecentMessagesWidget.php`

**Type:** Table Widget

```php
protected function getTableQuery(): Builder
{
    return WhatsAppMessage::query()
        ->orderBy('created_at', 'desc')
        ->limit(10);
}

protected function getTableColumns(): array
{
    return [
        TextColumn::make('recipient.normalized'),
        BadgeColumn::make('template'),
        BadgeColumn::make('status'),
        TextColumn::make('created_at')->dateTime(),
    ];
}
```

---

### 4. Feature Tests

Feature tests use real database and test end-to-end flows.

#### Test Files

```
Modules/WhatsApp/Tests/Feature/
├── Commands/
│   ├── ProcessWhatsAppQueueCommandTest.php
│   └── RetryFailedMessagesCommandTest.php
├── Gateways/
│   └── WaMeGatewayTest.php
├── Listeners/
│   ├── OrderCreatedListenerTest.php
│   ├── OrderStatusChangedListenerTest.php
│   └── PaymentConfirmedListenerTest.php
└── Filament/
    └── WhatsAppMessageResourceTest.php
```

#### Test Coverage

**ProcessWhatsAppQueueCommandTest:**
- Command processes pending messages
- Command respects limit option
- Command queues messages correctly
- Command displays stats table
- Command handles empty queue gracefully

**RetryFailedMessagesCommandTest:**
- Command retries failed messages
- Command skips messages with max attempts
- Command displays stats table
- Command handles no failed messages

**WaMeGatewayTest:**
- Generates valid wa.me URL
- Sanitizes special characters
- Truncates long messages
- URL encodes message correctly
- Handles emojis properly

**OrderCreatedListenerTest:**
- Listener creates WhatsApp message on OrderCreated event
- Message has correct template (ORDER_CREATED)
- Message has correct recipient (merchant phone)
- Message context contains order data

**WhatsAppMessageResourceTest:**
- Resource displays messages table
- Filters work correctly
- Retry action works for failed messages
- View action shows detail page

---

## Validation Commands

```bash
# Run feature tests
./vendor/bin/sail test --filter=WhatsApp/Tests/Feature

# Test commands manually
./vendor/bin/sail artisan whatsapp:process-queue
./vendor/bin/sail artisan whatsapp:retry-failed

# Static analysis
./vendor/bin/sail composer run phpstan -- --paths=Modules/WhatsApp/Console,Modules/WhatsApp/Filament

# Code formatting
./vendor/bin/sail bin pint Modules/WhatsApp/Console Modules/WhatsApp/Filament
```

## Acceptance Criteria

- [ ] ProcessWhatsAppQueueCommand processes messages asynchronously
- [ ] RetryFailedMessagesCommand only retries retryable messages
- [ ] Commands show progress and stats
- [ ] Filament resource displays real-time message status
- [ ] Filament widgets show aggregated stats
- [ ] Feature tests cover all scenarios
- [ ] Smoke tests for Filament resource
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## Definition of Done

- [ ] All deliverables implemented
- [ ] All feature tests passing
- [ ] PHPStan level 6+ passes without errors
- [ ] Pint formatting applied
- [ ] Commands scheduled in Kernel
- [ ] Filament resource accessible
- [ ] Code reviewed and approved
- [ ] Documentation complete

---

**Status:** Pending  
**Assignee:** Agente D  
**Due Date:** TBD
