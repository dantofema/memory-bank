# Task 002: Actions, Services and Business Logic

**Agent:** Agente B - Actions y Tests Unitarios  
**Module:** WhatsApp  
**Priority:** HIGH  
**Estimated Time:** 8 hours  
**Dependencies:** 001-contracts

## Objective

Implement all Actions (Commands, Queries, Internal) and Services that contain the business logic for the WhatsApp module. This includes message creation, sending, retry logic, template rendering, and gateway implementations.

## Context

This task implements the core business logic layer. All Actions must be `final` classes with a single public `execute()` method. Services coordinate Actions and implement complex workflows. The gateway pattern allows switching between WA.ME and WhatsApp Business API.

## References

- **Agents Prompt:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#task-002`
- **Domain Model:** `@e-commerce-wa-ml/whatsapp/domain_model.md`
- **Actions Guide:** `@laravel/agents/agent-b-actions.md`
- **Business Rules:** `@e-commerce-wa-ml/whatsapp/agents_prompt.md#business-rules-critical`

## Deliverables

### 1. Actions - Commands

All Command Actions must implement their corresponding interface from Task 001.

#### 1.1 CreateMessageAction

**Location:** `Modules/WhatsApp/Actions/Commands/CreateMessageAction.php`

**Implements:** `CreateMessageInterface`

**Responsibility:** Create and validate a new WhatsApp message.

**Dependencies:**
- `MessageTemplateRenderer` - Validate template-context compatibility
- `NormalizePhoneNumberAction` - Normalize recipient phone
- `WhatsAppMessageRepository` - Persist message

**Logic:**
1. Validate template-context compatibility
2. Normalize recipient phone number
3. Create WhatsAppMessage entity with status PENDING
4. Persist to repository
5. Return WhatsAppMessageData

**Exceptions:**
- `InvalidMessageTemplateException` - Invalid template
- `InvalidMessageContextException` - Missing required context fields
- `InvalidPhoneNumberException` - Invalid phone number

---

#### 1.2 SendMessageAction

**Location:** `Modules/WhatsApp/Actions/Commands/SendMessageAction.php`

**Implements:** `SendMessageInterface`

**Responsibility:** Send a message synchronously via gateway.

**Dependencies:**
- `WhatsAppMessageRepository` - Get message
- `WhatsAppGateway` - Send message
- `MarkMessageAsSentAction` - Update on success
- `MarkMessageAsFailedAction` - Update on failure

**Logic:**
1. Get message from repository
2. Validate message is ready to send (PENDING or QUEUED)
3. Change status to SENDING
4. Send via gateway
5. If success: call MarkMessageAsSentAction
6. If failure: call MarkMessageAsFailedAction
7. Return SendResultData

**Exceptions:**
- `MessageSendFailedException` - Gateway error

---

#### 1.3 SendMessageAsyncAction

**Location:** `Modules/WhatsApp/Actions/Commands/SendMessageAsyncAction.php`

**Responsibility:** Queue a message for asynchronous sending.

**Dependencies:**
- `WhatsAppMessageRepository` - Get message
- `SendWhatsAppMessageJob` - Queue job

**Logic:**
1. Get message from repository
2. Validate message is in PENDING status
3. Change status to QUEUED
4. Dispatch SendWhatsAppMessageJob with backoff configuration
5. Return WhatsAppMessageData

---

#### 1.4 MarkMessageAsSentAction

**Location:** `Modules/WhatsApp/Actions/Commands/MarkMessageAsSentAction.php`

**Responsibility:** Mark message as successfully sent.

**Dependencies:**
- `WhatsAppMessageRepository` - Update message

**Logic:**
1. Get message from repository
2. Validate status transition to SENT is allowed
3. Set status to SENT
4. Set sent_at timestamp
5. Clear failure_reason
6. Persist changes
7. Return WhatsAppMessageData

**Exceptions:**
- `InvalidStateTransitionException` - Cannot transition from current state

---

#### 1.5 MarkMessageAsFailedAction

**Location:** `Modules/WhatsApp/Actions/Commands/MarkMessageAsFailedAction.php`

**Responsibility:** Mark message as failed and determine if retryable.

**Dependencies:**
- `WhatsAppMessageRepository` - Update message
- `WhatsAppConfig` - Get max attempts

**Logic:**
1. Get message from repository
2. Increment attempt_count
3. Set status to FAILED or DISCARDED (if attempts >= max)
4. Set failed_at timestamp
5. Set failure_reason
6. Persist changes
7. Return WhatsAppMessageData

**Business Rule:** After 3rd failure, message automatically transitions to DISCARDED.

---

#### 1.6 RetryFailedMessageAction

**Location:** `Modules/WhatsApp/Actions/Commands/RetryFailedMessageAction.php`

**Implements:** `RetryFailedMessageInterface`

**Responsibility:** Retry a failed message.

**Dependencies:**
- `WhatsAppMessageRepository` - Update message
- `SendMessageAsyncAction` - Requeue message

**Logic:**
1. Get message from repository
2. Validate message can be retried (FAILED status, attempts < max)
3. Reset status to PENDING
4. Clear failure_reason
5. Call SendMessageAsyncAction to requeue
6. Return WhatsAppMessageData

**Exceptions:**
- `MessageNotRetryableException` - Message has reached max attempts or is not in FAILED status

---

### 2. Actions - Queries

#### 2.1 GetMessageAction

**Location:** `Modules/WhatsApp/Actions/Queries/GetMessageAction.php`

**Implements:** `GetMessageInterface`

**Responsibility:** Get a single message by ID.

**Dependencies:**
- `WhatsAppMessageRepository` - Query repository

**Logic:**
1. Query repository by ID
2. Convert model to WhatsAppMessageData
3. Return data

**Exceptions:**
- `ModelNotFoundException` - Message not found

---

#### 2.2 GetPendingMessagesAction

**Location:** `Modules/WhatsApp/Actions/Queries/GetPendingMessagesAction.php`

**Implements:** `GetPendingMessagesInterface`

**Responsibility:** Get pending messages ready to send.

**Dependencies:**
- `WhatsAppMessageRepository` - Query repository

**Logic:**
1. Query repository for messages in PENDING status
2. Filter by scheduled_at <= now
3. Order by priority (ORDER_CREATED first) then created_at
4. Limit to specified amount (default 100)
5. Convert models to array of WhatsAppMessageData
6. Return array

---

#### 2.3 GetFailedMessagesAction

**Location:** `Modules/WhatsApp/Actions/Queries/GetFailedMessagesAction.php`

**Implements:** `GetFailedMessagesInterface`

**Responsibility:** Get failed messages that can be retried.

**Dependencies:**
- `WhatsAppMessageRepository` - Query repository

**Logic:**
1. Query repository for messages in FAILED status
2. Filter by attempt_count < maxAttempts
3. Order by failed_at ASC
4. Convert models to array of WhatsAppMessageData
5. Return array

---

#### 2.4 GetMessageStatsAction

**Location:** `Modules/WhatsApp/Actions/Queries/GetMessageStatsAction.php`

**Responsibility:** Get aggregated message statistics.

**Dependencies:**
- `WhatsAppMessageRepository` - Query repository

**Logic:**
1. Count messages by status
2. Build MessageStatsData with counts
3. Return data

---

### 3. Actions - Internal

Internal Actions are used by other Actions/Services and are not exposed via contracts.

#### 3.1 RenderTemplateAction

**Location:** `Modules/WhatsApp/Actions/Internal/RenderTemplateAction.php`

**Responsibility:** Render a template with context data.

**Dependencies:** None (uses Blade-like syntax or simple string replacement)

**Logic:**
1. Get template content from MessageTemplate enum
2. Interpolate variables from context
3. Handle conditionals ({{#if}})
4. Handle loops ({{#each}})
5. Return rendered string

**Example:**
```php
$template = "Pedido #{{orderNumber}} - Total: ${{totalAmount}}";
$context = ['orderNumber' => '12345', 'totalAmount' => '1500'];
$rendered = "Pedido #12345 - Total: $1500";
```

---

#### 3.2 ValidateContextAction

**Location:** `Modules/WhatsApp/Actions/Internal/ValidateContextAction.php`

**Responsibility:** Validate context has required fields for template.

**Dependencies:** None

**Logic:**
1. Get required fields from MessageTemplate
2. Check all required fields exist in context
3. Validate field types (if applicable)
4. Throw exception if any field is missing
5. Return true if valid

**Exceptions:**
- `InvalidMessageContextException` - Missing required field

---

#### 3.3 NormalizePhoneNumberAction

**Location:** `Modules/WhatsApp/Actions/Internal/NormalizePhoneNumberAction.php`

**Responsibility:** Normalize phone number to E.164 format.

**Dependencies:** None (uses libphonenumber or similar)

**Logic:**
1. Parse phone number string
2. Detect country code (default +54 for Argentina)
3. Validate format
4. Normalize to E.164 format (+5491112345678)
5. Return PhoneNumber VO

**Exceptions:**
- `InvalidPhoneNumberException` - Invalid phone format

---

#### 3.4 SanitizeMessageAction

**Location:** `Modules/WhatsApp/Actions/Internal/SanitizeMessageAction.php`

**Responsibility:** Remove URL-breaking characters from message.

**Dependencies:** None

**Logic:**
1. Remove or encode characters: %, &, =, #
2. Preserve line breaks and emojis
3. Return sanitized string

**Characters to sanitize:**
- `%` → remove or encode
- `&` → remove or encode
- `=` → remove or encode
- `#` → remove or encode

---

#### 3.5 TruncateMessageAction

**Location:** `Modules/WhatsApp/Actions/Internal/TruncateMessageAction.php`

**Responsibility:** Truncate message to wa.me URL limit.

**Dependencies:** None

**Logic:**
1. Check if message length > 2048 characters
2. If yes, truncate to 2045 characters
3. Append "..." to truncated message
4. Return truncated string

---

#### 3.6 GenerateWaMeLinkAction

**Location:** `Modules/WhatsApp/Actions/Internal/GenerateWaMeLinkAction.php`

**Responsibility:** Generate wa.me link with pre-filled message.

**Dependencies:**
- `SanitizeMessageAction` - Sanitize message
- `TruncateMessageAction` - Truncate if needed

**Logic:**
1. Sanitize message text
2. Truncate if exceeds limit
3. URL-encode message
4. Build wa.me URL: `https://wa.me/{phone}?text={encoded_message}`
5. Return URL string

**Example:**
```php
$link = "https://wa.me/5491112345678?text=Pedido%20%2312345%20confirmado";
```

---

### 4. Services

Services coordinate Actions and implement complex workflows.

#### 4.1 WhatsAppMessageService

**Location:** `Modules/WhatsApp/Services/WhatsAppMessageService.php`

**Responsibility:** Main application service for message management.

**Dependencies:**
- `WhatsAppMessageRepository` - Data access
- `WhatsAppGateway` - Message sending
- `MessageTemplateRenderer` - Template rendering
- `CreateMessageAction` - Create messages
- `SendMessageAction` - Send messages
- `SendMessageAsyncAction` - Queue messages
- `RetryFailedMessageAction` - Retry messages
- `GetPendingMessagesAction` - Query pending
- `GetFailedMessagesAction` - Query failed

**Public Methods:**

```php
public function createMessage(
    MessageTemplate $template,
    MessageContext $context,
    PhoneNumber $recipient
): WhatsAppMessageData;

public function sendMessage(string $messageId): SendResultData;

public function sendMessageAsync(string $messageId): void;

public function processQueue(int $limit = 100): array; // Stats

public function retryFailedMessages(): array; // Stats
```

**processQueue() Logic:**
1. Get pending messages (limit)
2. For each message:
   - Call SendMessageAsyncAction
3. Return stats (processed count, failed count)

**retryFailedMessages() Logic:**
1. Get failed messages that can be retried
2. For each message:
   - Call RetryFailedMessageAction
3. Return stats (retried count)

---

#### 4.2 MessageTemplateRenderer

**Location:** `Modules/WhatsApp/Services/MessageTemplateRenderer.php`

**Responsibility:** Render message templates with context.

**Dependencies:**
- `RenderTemplateAction` - Render logic
- `ValidateContextAction` - Validation logic

**Public Methods:**

```php
public function render(MessageTemplate $template, MessageContext $context): string;

public function validate(MessageTemplate $template, MessageContext $context): bool;

private function loadTemplate(MessageTemplate $template): string;

private function interpolate(string $template, array $data): string;
```

**render() Logic:**
1. Validate context has required fields
2. Load template content
3. Interpolate variables
4. Handle conditionals
5. Handle loops
6. Return rendered string

---

#### 4.3 WhatsAppGateway (Interface)

**Location:** `Modules/WhatsApp/Services/Gateways/WhatsAppGateway.php`

**Responsibility:** Abstraction for different sending strategies.

**Public Methods:**

```php
public function send(WhatsAppMessage $message): SendResult;

public function generateLink(WhatsAppMessage $message): string;

public function validateRecipient(PhoneNumber $phone): bool;
```

---

#### 4.4 WaMeGateway

**Location:** `Modules/WhatsApp/Services/Gateways/WaMeGateway.php`

**Implements:** `WhatsAppGateway`

**Responsibility:** MVP implementation using wa.me links.

**Dependencies:**
- `WhatsAppConfig` - Configuration
- `MessageTemplateRenderer` - Render message
- `GenerateWaMeLinkAction` - Build URL

**Public Methods:**

```php
public function send(WhatsAppMessage $message): SendResult;

public function generateLink(WhatsAppMessage $message): string;

public function validateRecipient(PhoneNumber $phone): bool;

private function buildUrl(PhoneNumber $phone, string $message): string;

private function sanitizeMessage(string $message): string;

private function truncateMessage(string $message): string;
```

**send() Logic:**
1. Render message from template and context
2. Generate wa.me link
3. Return SendResult with success=true (no actual API call)
4. In real implementation, this would trigger browser opening or notification

**Limitations:**
- No delivery confirmation
- No API call (just generates link)
- Merchant must manually click link to send
- 2048 character limit

---

#### 4.5 WhatsAppBusinessApiGateway

**Location:** `Modules/WhatsApp/Services/Gateways/WhatsAppBusinessApiGateway.php`

**Implements:** `WhatsAppGateway`

**Responsibility:** Future implementation for WhatsApp Business API.

**Dependencies:**
- `HttpClient` - HTTP requests
- `WhatsAppConfig` - API credentials

**Public Methods:**

```php
public function send(WhatsAppMessage $message): SendResult;

public function generateLink(WhatsAppMessage $message): string;

public function validateRecipient(PhoneNumber $phone): bool;

private function buildApiRequest(WhatsAppMessage $message): array;

private function handleApiResponse(Response $response): SendResult;
```

**Note:** This is a placeholder for Phase 2. MVP only uses WaMeGateway.

---

### 5. Exceptions

All domain exceptions must extend `DomainException` and include HTTP status code.

#### 5.1 InvalidMessageTemplateException

**Location:** `Modules/WhatsApp/Exceptions/InvalidMessageTemplateException.php`

**HTTP Status:** 422 (Unprocessable Entity)

**Message:** "Invalid message template: {template}"

---

#### 5.2 InvalidMessageContextException

**Location:** `Modules/WhatsApp/Exceptions/InvalidMessageContextException.php`

**HTTP Status:** 422

**Message:** "Missing required field: {field}"

---

#### 5.3 MessageNotRetryableException

**Location:** `Modules/WhatsApp/Exceptions/MessageNotRetryableException.php`

**HTTP Status:** 422

**Message:** "Message has reached max attempts or is not in retryable state"

---

#### 5.4 MessageSendFailedException

**Location:** `Modules/WhatsApp/Exceptions/MessageSendFailedException.php`

**HTTP Status:** 500 (Internal Server Error)

**Message:** "Failed to send WhatsApp message: {reason}"

---

#### 5.5 InvalidPhoneNumberException

**Location:** `Modules/WhatsApp/Exceptions/InvalidPhoneNumberException.php`

**HTTP Status:** 422

**Message:** "Invalid phone number: {phone}"

---

#### 5.6 GatewayNotConfiguredException

**Location:** `Modules/WhatsApp/Exceptions/GatewayNotConfiguredException.php`

**HTTP Status:** 500

**Message:** "WhatsApp gateway not configured"

---

## Unit Tests

All Actions and Services must have comprehensive unit tests with mocks.

### Test Files

```
Modules/WhatsApp/Tests/Unit/
├── Actions/
│   ├── Commands/
│   │   ├── CreateMessageActionTest.php
│   │   ├── SendMessageActionTest.php
│   │   ├── SendMessageAsyncActionTest.php
│   │   ├── MarkMessageAsSentActionTest.php
│   │   ├── MarkMessageAsFailedActionTest.php
│   │   └── RetryFailedMessageActionTest.php
│   ├── Queries/
│   │   ├── GetMessageActionTest.php
│   │   ├── GetPendingMessagesActionTest.php
│   │   ├── GetFailedMessagesActionTest.php
│   │   └── GetMessageStatsActionTest.php
│   └── Internal/
│       ├── RenderTemplateActionTest.php
│       ├── ValidateContextActionTest.php
│       ├── NormalizePhoneNumberActionTest.php
│       ├── SanitizeMessageActionTest.php
│       ├── TruncateMessageActionTest.php
│       └── GenerateWaMeLinkActionTest.php
└── Services/
    ├── WhatsAppMessageServiceTest.php
    ├── MessageTemplateRendererTest.php
    └── Gateways/
        ├── WaMeGatewayTest.php
        └── WhatsAppBusinessApiGatewayTest.php
```

### Test Coverage Requirements

**CreateMessageAction:**
- Creates message with valid template and context
- Throws exception for missing context fields
- Normalizes phone number correctly
- Persists message with PENDING status

**SendMessageAction:**
- Sends message successfully via gateway
- Marks message as SENT on success
- Marks message as FAILED on gateway error
- Validates message status before sending

**RetryFailedMessageAction:**
- Retries FAILED message successfully
- Throws exception if attempts >= max
- Throws exception if message not in FAILED status
- Increments attempt count correctly

**WaMeGateway:**
- Generates valid wa.me URL
- Sanitizes message correctly (removes %, &, =, #)
- Truncates message at 2048 characters
- Appends "..." to truncated messages
- URL-encodes message properly

**MessageTemplateRenderer:**
- Renders template with context successfully
- Throws exception for missing required fields
- Handles conditionals ({{#if}})
- Handles loops ({{#each}})
- Interpolates variables correctly

**WhatsAppMessageService:**
- Creates and sends message successfully
- Processes queue with correct limit
- Retries failed messages with backoff
- Returns correct stats

### Mocking Strategy

Use Pest mocks for:
- `WhatsAppMessageRepository` - Mock all repository methods
- `WhatsAppGateway` - Mock send() method
- `MessageTemplateRenderer` - Mock render() method

Do NOT use real database or HTTP calls in unit tests.

## Validation Commands

```bash
# Run unit tests
./vendor/bin/sail test --filter=WhatsApp/Tests/Unit/Actions
./vendor/bin/sail test --filter=WhatsApp/Tests/Unit/Services

# Static analysis
./vendor/bin/sail composer run phpstan -- --paths=Modules/WhatsApp/Actions,Modules/WhatsApp/Services,Modules/WhatsApp/Exceptions

# Code formatting
./vendor/bin/sail bin pint Modules/WhatsApp/Actions Modules/WhatsApp/Services Modules/WhatsApp/Exceptions
```

## Acceptance Criteria

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

## Definition of Done

- [ ] All deliverables implemented
- [ ] All unit tests passing with 100% coverage
- [ ] PHPStan level 6+ passes without errors
- [ ] Pint formatting applied
- [ ] Code reviewed and approved
- [ ] Documentation complete with PHPDoc blocks
- [ ] No `@phpstan-ignore` comments

---

**Status:** Pending  
**Assignee:** Agente B  
**Due Date:** TBD
