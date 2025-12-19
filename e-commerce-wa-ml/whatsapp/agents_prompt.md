---
title: "WhatsApp Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the WhatsApp module following the 5-agent architecture"
module: "WhatsApp"
phase: "3 - Integraciones"
module_type: "TRANSVERSAL"
dependencies:
  - "Orders"
  - "Payments"
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/whatsapp/domain_model.md"
---

# WhatsApp Module - Professional Agents Prompt

## Context

You are tasked with implementing the **WhatsApp module**, a **TRANSVERSAL module** in Phase 3 (Integraciones) of the e-commerce platform. This module manages:

- **Asynchronous WhatsApp notifications** to merchant and customers
- **Message templates** for order and payment events
- **Queue-based message processing** with retry logic
- **WA.ME link generation** (MVP implementation)
- **Message status tracking** and failure handling
- **Phone number normalization** (E.164 format)
- **Exponential backoff** for retries (1min, 5min, 15min)
- **Future-ready architecture** for WhatsApp Business API migration

This module **reacts to domain events** from Orders and Payments modules and sends contextual notifications asynchronously.

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
- **Frontend public:** None (this is a backend/notification module)
- **Backoffice:** Filament v4 (message monitoring)
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
- **MVP uses WA.ME links** (no official API yet)
- **Merchant-only notifications** for MVP (no customer messages yet)
- **No delivery confirmation** (limitation of wa.me)
- **No bidirectional conversations** (future phase)
- **Max 3 retry attempts** with exponential backoff
- **Max 2048 characters** per message (URL limit)

---

## Domain Model Overview

### Key Entities

**WhatsAppMessage (Aggregate Root):**
- Message entity with recipient, template, context, status
- Tracks attempt count, sent/failed timestamps
- Lifecycle: PENDING → QUEUED → SENDING → SENT/FAILED/DISCARDED
- Immutable after SENT or DISCARDED

### Key Enums

**MessageTemplate** (ORDER_CREATED, ORDER_CONFIRMED, ORDER_IN_DELIVERY, ORDER_DELIVERED, ORDER_CANCELLED, ORDER_REJECTED, PAYMENT_CONFIRMED, PAYMENT_REFUNDED)

**MessageStatus** (PENDING, QUEUED, SENDING, SENT, FAILED, DISCARDED)

### Key Value Objects

**PhoneNumber** (normalized E.164 format), **MessageContext** (order data for template interpolation), **SendResult** (success/failure result), **WhatsAppConfig** (module configuration)

### Key Services

**WhatsAppMessageService:**
- Create messages from templates
- Send messages (sync or async)
- Process message queue
- Retry failed messages

**MessageTemplateRenderer:**
- Render templates with context data
- Validate context completeness
- Interpolate variables

**WhatsAppGateway (Interface):**
- Abstraction for different sending strategies
- MVP: WaMeGateway (wa.me links)
- Future: WhatsAppBusinessApiGateway

**WaMeGateway:**
- Generate wa.me URLs with pre-formatted messages
- Sanitize and truncate messages (2048 char limit)
- No API key required
- No delivery confirmation

---

## Business Rules (CRITICAL)

### Message Lifecycle (Rules 1-4)
1. A message cannot have more than **3 retry attempts**
2. After 3rd failure, message transitions to **DISCARDED** (no more retries)
3. Messages in **SENT** or **DISCARDED** status are **immutable**
4. Only messages in PENDING or QUEUED can transition to SENDING

### Template Validation (Rules 5-7)
5. Context must contain **all required fields** for template
6. Missing fields cause message creation to **fail immediately**
7. Template-context compatibility validated **before** creating message

### Phone Normalization (Rules 8-10)
8. All phone numbers **normalized to E.164** format (+54...)
9. Normalization happens **before** storage and sending
10. Invalid phone numbers cause message to be **discarded**

### Retry Logic (Rules 11-14)
11. Retry backoff: **1 min, 5 min, 15 min** (exponential)
12. Retries only for **FAILED** messages with attempts < max
13. Retries reset status to **PENDING** and increment attempt count
14. **No automatic retries** after 3rd failure (manual intervention required)

### Message Content (Rules 15-18)
15. Messages truncated at **2048 characters** (wa.me URL limit)
16. Truncation adds "..." suffix
17. Messages **sanitized** to remove URL-breaking characters (%, &, =, #)
18. No sensitive data (passwords, tokens, full card numbers)

### Queue Priority (Rules 19-20)
19. **ORDER_CREATED** messages have **HIGH priority**
20. All other messages have **NORMAL priority**

### MVP Limitations (Rules 21-24)
21. **Merchant-only** notifications (no customer messages)
22. **No delivery confirmation** (wa.me limitation)
23. **No conversation tracking** (one-way notifications only)
24. **No read receipts** (wa.me limitation)

---

## Module Structure

```
Modules/WhatsApp/
├── Contracts/                             # Agent A
│   ├── Commands/
│   │   ├── CreateMessageInterface.php
│   │   ├── SendMessageInterface.php
│   │   └── RetryFailedMessageInterface.php
│   ├── Queries/
│   │   ├── GetMessageInterface.php
│   │   ├── GetPendingMessagesInterface.php
│   │   └── GetFailedMessagesInterface.php
│   └── Data/
│       ├── WhatsAppMessageData.php
│       ├── MessageContextData.php
│       ├── SendResultData.php
│       └── MessageStatsData.php
├── ValueObjects/                          # Agent A
│   ├── PhoneNumber.php
│   ├── MessageContext.php
│   ├── SendResult.php
│   └── WhatsAppConfig.php
├── Enums/                                 # Agent A
│   ├── MessageTemplate.php
│   └── MessageStatus.php
├── Casts/                                 # Agent A
│   ├── PhoneNumberCast.php
│   ├── MessageContextCast.php
│   ├── MessageTemplateCast.php
│   └── MessageStatusCast.php
├── Actions/                               # Agent B
│   ├── Commands/
│   │   ├── CreateMessageAction.php
│   │   ├── SendMessageAction.php
│   │   ├── SendMessageAsyncAction.php
│   │   ├── MarkMessageAsSentAction.php
│   │   ├── MarkMessageAsFailedAction.php
│   │   └── RetryFailedMessageAction.php
│   ├── Queries/
│   │   ├── GetMessageAction.php
│   │   ├── GetPendingMessagesAction.php
│   │   ├── GetFailedMessagesAction.php
│   │   └── GetMessageStatsAction.php
│   └── Internal/
│       ├── RenderTemplateAction.php
│       ├── ValidateContextAction.php
│       ├── NormalizePhoneNumberAction.php
│       ├── SanitizeMessageAction.php
│       ├── TruncateMessageAction.php
│       └── GenerateWaMeLinkAction.php
├── Services/                              # Agent B
│   ├── WhatsAppMessageService.php
│   ├── MessageTemplateRenderer.php
│   ├── Gateways/
│   │   ├── WhatsAppGateway.php (interface)
│   │   ├── WaMeGateway.php
│   │   └── WhatsAppBusinessApiGateway.php (future)
│   └── WhatsAppConfigService.php
├── Exceptions/                            # Agent B
│   ├── InvalidMessageTemplateException.php
│   ├── InvalidMessageContextException.php
│   ├── MessageNotRetryableException.php
│   ├── MessageSendFailedException.php
│   ├── InvalidPhoneNumberException.php
│   └── GatewayNotConfiguredException.php
├── Models/                                # Agent C
│   └── WhatsAppMessage.php
├── Repositories/                          # Agent C
│   └── WhatsAppMessageRepository.php
├── Database/                              # Agent C
│   ├── Factories/
│   │   └── WhatsAppMessageFactory.php
│   ├── Migrations/
│   │   └── xxxx_create_whatsapp_messages_table.php
│   └── Seeders/
│       └── WhatsAppSeeder.php
├── Filament/                              # Agent D
│   └── Resources/
│       └── WhatsAppMessageResource.php
│           └── Widgets/
│               ├── MessageStatsWidget.php
│               └── RecentMessagesWidget.php
├── Console/                               # Agent D
│   └── Commands/
│       ├── ProcessWhatsAppQueueCommand.php
│       └── RetryFailedMessagesCommand.php
├── Listeners/                             # Agent E
│   ├── OrderCreatedListener.php
│   ├── OrderStatusChangedListener.php
│   ├── PaymentConfirmedListener.php
│   └── PaymentRefundedListener.php
├── Jobs/                                  # Agent E
│   └── SendWhatsAppMessageJob.php
└── Tests/
    ├── Unit/                              # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   ├── Actions/
    │   └── Services/
    └── Feature/                           # Agent D
        ├── SendWhatsAppMessageTest.php
        ├── ProcessWhatsAppQueueTest.php
        ├── RetryFailedMessagesTest.php
        ├── OrderCreatedListenerTest.php
        └── WaMeGatewayTest.php
```

---

## Message Templates

### ORDER_CREATED Template

```
🛒 *Nuevo Pedido #{{orderNumber}}*

👤 Cliente: {{customerName}}
📱 Teléfono: {{customerPhone}}
📍 Dirección: {{deliveryAddress}}

*Productos:*
{{#each items}}
- {{quantity}}x {{name}} - ${{price}}
{{/each}}

💰 *Total: ${{totalAmount}}*

💳 Pago: {{paymentMethod}}

{{#if observations}}
📝 Observaciones: {{observations}}
{{/if}}
```

**Required context fields:** orderNumber, customerName, customerPhone, deliveryAddress, items (with quantity, name, price), totalAmount, paymentMethod

---

### ORDER_CONFIRMED Template

```
✅ *Pedido #{{orderNumber}} Confirmado*

Tu pedido ha sido confirmado y está siendo preparado.

💰 Total: ${{totalAmount}}
📍 Dirección: {{deliveryAddress}}
```

**Required context fields:** orderNumber, totalAmount, deliveryAddress

---

### PAYMENT_CONFIRMED Template

```
💳 *Pago Confirmado*

El pago de tu pedido #{{orderNumber}} ha sido confirmado.

💰 Monto: ${{totalAmount}}
```

**Required context fields:** orderNumber, totalAmount

---

## Events Consumed

### From Orders Module

- `OrderCreated` → Create ORDER_CREATED message
- `OrderStatusChanged` → Create ORDER_CONFIRMED/IN_DELIVERY/DELIVERED/CANCELLED/REJECTED message
- `OrderCancelled` → Create ORDER_CANCELLED message

### From Payments Module

- `PaymentConfirmed` → Create PAYMENT_CONFIRMED message
- `PaymentRefunded` → Create PAYMENT_REFUNDED message

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `PhoneNumber` (countryCode, number, normalized, Wireable)
     - Properties: `string $countryCode`, `string $number`, `string $normalized`
     - Methods: `toString()`, `toInternational()`, `equals()`, `isValid()`
     - Validation: E.164 format, 8-15 digits, numeric only
     - Normalization: +5491234567890
   - `MessageContext` (order data, Wireable)
     - Properties: all order data fields
     - Methods: `toArray()`, `fillTemplate()`, `validate()`
     - Validation: required fields per template
   - `SendResult` (success, messageId, errorMessage, timestamp, Wireable)
     - Methods: `isSuccess()`, `isFailed()`, `getErrorMessage()`
   - `WhatsAppConfig` (gateway, merchantPhone, enabled, maxAttempts, etc., Wireable)
     - Loaded from env variables
     - Methods: `getGateway()`, `isEnabled()`, `shouldSendToCustomers()`

2. **Enums:**
   - `MessageTemplate` (ORDER_CREATED, ORDER_CONFIRMED, ORDER_IN_DELIVERY, ORDER_DELIVERED, ORDER_CANCELLED, ORDER_REJECTED, PAYMENT_CONFIRMED, PAYMENT_REFUNDED)
     - Methods: `getTemplateName()`, `getTemplateContent()`, `requiresContext()`
   - `MessageStatus` (PENDING, QUEUED, SENDING, SENT, FAILED, DISCARDED)
     - Methods: `isPending()`, `isFinal()`, `canTransitionTo()`

3. **Casts:**
   - `PhoneNumberCast` (string ↔ PhoneNumber)
   - `MessageContextCast` (JSON ↔ MessageContext)
   - `MessageTemplateCast` (string ↔ MessageTemplate)
   - `MessageStatusCast` (string ↔ MessageStatus)

4. **Data Objects (Spatie Laravel Data):**
   - `WhatsAppMessageData` (id, recipient, template, context, status, attemptCount, scheduledAt, sentAt, failedAt, failureReason)
   - `MessageContextData` (all order/payment data fields)
   - `SendResultData` (success, messageId, errorMessage, timestamp)
   - `MessageStatsData` (totalMessages, sentMessages, failedMessages, pendingMessages, discardedMessages)

5. **Contracts (Commands):**
   - `CreateMessageInterface`
   - `SendMessageInterface`
   - `RetryFailedMessageInterface`

6. **Contracts (Queries):**
   - `GetMessageInterface`
   - `GetPendingMessagesInterface`
   - `GetFailedMessagesInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] PhoneNumber validates and normalizes to E.164
- [ ] MessageContext validates required fields per template
- [ ] MessageStatus validates state transitions
- [ ] MessageTemplate includes all template strings
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] Unit tests for all VOs
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions, Services and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**

1. **Actions Commands:**
   - `CreateMessageAction` (validate template-context, normalize phone, create message)
   - `SendMessageAction` (send synchronously via gateway)
   - `SendMessageAsyncAction` (queue message for async sending)
   - `MarkMessageAsSentAction` (update status, set sent_at timestamp)
   - `MarkMessageAsFailedAction` (update status, increment attempts, set failure reason)
   - `RetryFailedMessageAction` (reset to PENDING, increment attempts, requeue)

2. **Actions Queries:**
   - `GetMessageAction` (get by ID)
   - `GetPendingMessagesAction` (get messages ready to send, with limit)
   - `GetFailedMessagesAction` (get retryable failed messages)
   - `GetMessageStatsAction` (count messages by status)

3. **Actions Internal:**
   - `RenderTemplateAction` (interpolate template with context data)
   - `ValidateContextAction` (check all required fields present)
   - `NormalizePhoneNumberAction` (convert to E.164)
   - `SanitizeMessageAction` (remove URL-breaking characters)
   - `TruncateMessageAction` (limit to 2048 chars with "...")
   - `GenerateWaMeLinkAction` (build wa.me URL with message)

4. **Services:**
   - `WhatsAppMessageService`
     - Dependencies: WhatsAppMessageRepository, WhatsAppGateway, MessageTemplateRenderer
     - Methods: `createMessage()`, `sendMessage()`, `sendMessageAsync()`, `processQueue()`, `retryFailedMessages()`
   - `MessageTemplateRenderer`
     - Dependencies: Config (template strings)
     - Methods: `render()`, `validate()`, `loadTemplate()`, `interpolate()`
   - `WhatsAppGateway` (interface)
     - Methods: `send()`, `generateLink()`, `validateRecipient()`
   - `WaMeGateway` (implements WhatsAppGateway)
     - Dependencies: Config
     - Methods: `send()`, `generateLink()`, `validateRecipient()`
     - Private: `buildUrl()`, `sanitizeMessage()`, `truncateMessage()`
   - `WhatsAppBusinessApiGateway` (implements WhatsAppGateway, future)
     - For WhatsApp Business API integration (Phase 2)

5. **Exceptions:**
   - `InvalidMessageTemplateException` (422)
   - `InvalidMessageContextException` (422, "Missing required field: {field}")
   - `MessageNotRetryableException` (422, "Message has reached max attempts")
   - `MessageSendFailedException` (500)
   - `InvalidPhoneNumberException` (422)
   - `GatewayNotConfiguredException` (500)

**Unit Tests:**
- Mock all repositories, gateways
- Test WhatsAppMessageService message creation and sending
- Test MessageTemplateRenderer template interpolation
- Test WaMeGateway URL generation, sanitization, truncation
- Test retry logic (backoff, max attempts)
- Test phone normalization
- 100% coverage of business logic

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method
- [ ] All Services are `final`
- [ ] WhatsAppMessageService validates context before creating message
- [ ] WaMeGateway generates correct wa.me URLs
- [ ] WaMeGateway sanitizes messages (removes %, &, =, #)
- [ ] WaMeGateway truncates messages at 2048 chars
- [ ] Retry logic implements exponential backoff (1min, 5min, 15min)
- [ ] Max 3 retry attempts enforced
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database, no HTTP)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**

1. **Models:**
   - `WhatsAppMessage` (id ulid, recipient, template enum, context jsonb, status enum, attempt_count, scheduled_at, sent_at, failed_at, failure_reason, timestamps)
     - Uses Casts for Value Objects and Enums
     - Scopes: `pending()`, `failed()`, `sent()`, `retryable()`
     - Methods: `isReadyToSend()`, `canRetry()`, `markAsSent()`, `markAsFailed()`, `incrementAttempt()`

2. **Repositories:**
   - `WhatsAppMessageRepository` (save, findById, findPendingMessages, findFailedMessages, findByRecipient, findByStatus)
     - `findPendingMessages()` with limit, ordered by priority (HIGH first)
     - `findFailedMessages()` where status = FAILED and attempts < max

3. **Migrations:**
   - `create_whatsapp_messages_table`
     - Columns: id (ulid), recipient (varchar 20), template (varchar 50), context (jsonb), status (varchar 20), attempt_count (int default 0), scheduled_at, sent_at, failed_at, failure_reason (text), timestamps
     - Indexes: status, recipient, scheduled_at, (status, attempt_count) for retryable queries

4. **Factories:**
   - `WhatsAppMessageFactory` (states: pending, queued, sending, sent, failed, discarded, withAttempts, forTemplate, forRecipient)

5. **Seeders:**
   - `WhatsAppSeeder` (sample messages for testing)

**Integration Tests:**
- Message persistence and retrieval
- Query scopes (pending, failed, retryable)
- Factory states

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs and Enums
- [ ] WhatsAppMessage model has scopes for filtering
- [ ] Repository implements all query methods
- [ ] Factory has states for all MessageStatus values
- [ ] Migration has proper indexes
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - Console Commands, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **Console Commands:**
   - `ProcessWhatsAppQueueCommand` (php artisan whatsapp:process-queue)
     - Gets pending messages (limit 100)
     - Changes status to QUEUED
     - Dispatches SendWhatsAppMessageJob for each
     - Shows progress and stats
     - Scheduled: every 5 minutes (cron)
   - `RetryFailedMessagesCommand` (php artisan whatsapp:retry-failed)
     - Gets failed messages with attempts < max
     - Resets status to PENDING
     - Increments attempt count
     - Requeues messages
     - Scheduled: every 30 minutes (cron)

2. **Filament Resources (Backoffice):**
   - `WhatsAppMessageResource` (view-only, monitoring)
     - Table: id, recipient, template badge, status badge, attempt count, sent_at, failed_at
     - Filters: status, template, date range, recipient
     - Detail page: full context JSON viewer, failure reason, timeline
     - Actions: Retry (if retryable), Discard
     - Relation managers: none

3. **Filament Widgets:**
   - `MessageStatsWidget` (total, sent, failed, pending, discarded counts)
   - `RecentMessagesWidget` (last 10 messages with status)

**Feature Tests:**
- Send WhatsApp message (success and failure)
- Process queue command (pending → queued → sent)
- Retry failed messages command (failed → pending → sent)
- OrderCreatedListener creates message
- OrderStatusChangedListener creates message
- PaymentConfirmedListener creates message
- WaMeGateway URL generation
- Message sanitization and truncation
- Smoke tests for Filament resource

**Acceptance Criteria:**
- [ ] ProcessWhatsAppQueueCommand processes messages asynchronously
- [ ] RetryFailedMessagesCommand only retries retryable messages
- [ ] Commands show progress and stats
- [ ] Filament resource displays real-time message status
- [ ] Filament widgets show aggregated stats
- [ ] Feature tests cover all scenarios
- [ ] Smoke tests for Filament resource
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - None (this module only consumes events, does not emit)

2. **Listeners:**
   - `OrderCreatedListener` (listens to `OrderCreated` from Orders)
     - Build MessageContext from Order
     - Create WhatsAppMessage with template ORDER_CREATED
     - Queue message for async sending
   - `OrderStatusChangedListener` (listens to `OrderStatusChanged` from Orders)
     - Check if status change requires notification (confirmed, in_delivery, delivered, cancelled, rejected)
     - Build MessageContext with new status
     - Select template based on new status
     - Create and queue message
   - `PaymentConfirmedListener` (listens to `PaymentConfirmed` from Payments)
     - Build MessageContext from Payment
     - Create WhatsAppMessage with template PAYMENT_CONFIRMED
     - Queue message
   - `PaymentRefundedListener` (listens to `PaymentRefunded` from Payments)
     - Build MessageContext from Payment
     - Create WhatsAppMessage with template PAYMENT_REFUNDED
     - Queue message

3. **Jobs:**
   - `SendWhatsAppMessageJob` (queued job for async sending)
     - Configuration: tries = 3, backoff = [60, 300, 900] seconds
     - Queue: whatsapp (dedicated queue)
     - Handle: get message by ID, validate QUEUED status, send via gateway, update status
     - Failed: mark message as FAILED with error reason

**Event Tests:**
- OrderCreatedListener creates message correctly
- OrderStatusChangedListener only creates messages for notifiable statuses
- PaymentConfirmedListener creates message correctly
- Job execution (success and failure paths)
- Job retry with exponential backoff

**Acceptance Criteria:**
- [ ] All listeners are `final`
- [ ] Listeners implement `ShouldQueue` for async processing
- [ ] OrderStatusChangedListener filters notifiable statuses
- [ ] SendWhatsAppMessageJob has correct backoff configuration
- [ ] SendWhatsAppMessageJob updates message status correctly
- [ ] Job failed() method marks message as FAILED
- [ ] Event tests verify message creation
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
- [ ] Unit tests for all Value Objects
- [ ] Unit tests for all Services
- [ ] Feature tests for all sending scenarios
- [ ] Feature tests for listeners
- [ ] Integration tests for repository
- [ ] Smoke tests for Filament resource
- [ ] 100% coverage for critical paths

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Template strings documented
- [ ] README.md in module root

### Database
- [ ] Migration with proper indexes
- [ ] Factory with all states

### Performance
- [ ] Messages sent asynchronously via queue
- [ ] Queue priority for ORDER_CREATED messages
- [ ] Commands limited to 100 messages per run

### Security
- [ ] Phone numbers normalized
- [ ] Messages sanitized
- [ ] No sensitive data in messages

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Send messages synchronously (use queue)
- Store phone numbers without normalization (E.164 required)
- Skip message sanitization (URL-breaking characters)
- Forget to truncate long messages (2048 char limit)
- Retry messages indefinitely (max 3 attempts)
- Include sensitive data in messages (passwords, tokens, full card numbers)
- Use database enums (use string columns with PHP enums)
- Skip context validation before creating message
- Block application flow for WhatsApp sending

✅ **DO:**
- Send all messages asynchronously via queue
- Normalize phone numbers to E.164 before storage
- Sanitize messages (remove %, &, =, #)
- Truncate messages at 2048 chars with "..."
- Implement exponential backoff (1min, 5min, 15min)
- Mark messages as DISCARDED after 3 failures
- Validate context completeness before creating message
- Use dedicated queue for WhatsApp messages
- Log all failures with reasons for debugging

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - OrderCreated events trigger WhatsApp notifications to merchant
   - OrderStatusChanged events trigger notifications
   - PaymentConfirmed events trigger notifications
   - Messages sent asynchronously via queue
   - Failed messages retry with exponential backoff (max 3 times)
   - Phone numbers normalized to E.164
   - Messages truncated and sanitized correctly
   - WA.ME links generated correctly
   - Filament resource shows message status

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - Messages queued asynchronously
   - Queue processed every 5 minutes
   - Failed messages retried every 30 minutes
   - No blocking operations

4. **Reliability:**
   - Retry logic with exponential backoff
   - Max 3 attempts enforced
   - Failed messages logged with reasons
   - Discarded messages tracked

---

## Configuration

### Environment Variables

```env
# WhatsApp Module
WHATSAPP_ENABLED=true
WHATSAPP_GATEWAY=wa_me
WHATSAPP_MERCHANT_PHONE=+5491112345678
WHATSAPP_MAX_ATTEMPTS=3
WHATSAPP_RETRY_DELAY=60
WHATSAPP_QUEUE_PRIORITY=high
WHATSAPP_SEND_TO_CUSTOMERS=false

# WA.ME Gateway
WAME_BASE_URL=https://wa.me/
WAME_MAX_MESSAGE_LENGTH=2048

# WhatsApp Business API (future)
WHATSAPP_API_URL=https://graph.facebook.com/v18.0/
WHATSAPP_API_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
```

### Config File: `config/whatsapp.php`

```php
return [
    'enabled' => env('WHATSAPP_ENABLED', true),
    'gateway' => env('WHATSAPP_GATEWAY', 'wa_me'),
    'merchant_phone' => env('WHATSAPP_MERCHANT_PHONE'),
    'max_attempts' => env('WHATSAPP_MAX_ATTEMPTS', 3),
    'retry_delay' => env('WHATSAPP_RETRY_DELAY', 60),
    'queue_priority' => env('WHATSAPP_QUEUE_PRIORITY', 'high'),
    'send_to_customers' => env('WHATSAPP_SEND_TO_CUSTOMERS', false),
    
    'wame' => [
        'base_url' => env('WAME_BASE_URL', 'https://wa.me/'),
        'max_message_length' => env('WAME_MAX_MESSAGE_LENGTH', 2048),
    ],
];
```

---

## Scheduled Commands

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule): void
{
    // Process WhatsApp queue every 5 minutes
    $schedule->command('whatsapp:process-queue')
        ->everyFiveMinutes()
        ->withoutOverlapping();
    
    // Retry failed messages every 30 minutes
    $schedule->command('whatsapp:retry-failed')
        ->everyThirtyMinutes()
        ->withoutOverlapping();
}
```

---

## Validation Commands

```bash
# Static analysis
./vendor/bin/sail composer run phpstan

# Code style
./vendor/bin/sail bin pint --dirty

# Unit tests
./vendor/bin/sail test --filter=Unit

# Feature tests
./vendor/bin/sail test --filter=Feature

# All tests
./vendor/bin/sail test

# Specific module tests
./vendor/bin/sail test Modules/WhatsApp

# Process queue manually
./vendor/bin/sail artisan whatsapp:process-queue

# Retry failed messages manually
./vendor/bin/sail artisan whatsapp:retry-failed
```

---

## Dependencies and Integration

### Modules that Depend on WhatsApp

- None (this is a terminal module)

### External Dependencies

- **Orders**: Consumes OrderCreated, OrderStatusChanged events
- **Payments**: Consumes PaymentConfirmed, PaymentRefunded events
- **Queue**: Laravel queue system for async processing

---

## Final Notes

- This module is **Phase 3 (Integraciones)** - implement after Orders and Payments are complete.
- This is a **transversal notification module** - reacts to events from other modules.
- **MVP uses WA.ME links** - no official API integration yet (future phase).
- **Merchant-only notifications** for MVP (customer notifications in Phase 2).
- **No delivery confirmation** (limitation of wa.me approach).
- Architecture is **future-ready** for WhatsApp Business API migration.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 12-16 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
