---
title: "Security Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Security module following the 5-agent architecture"
module: "Security"
phase: "1 - Fundamentos"
module_type: "TRANSVERSAL"
dependencies: []
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
---

# Security Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Security module**, a **TRANSVERSAL module** in Phase 1 (Fundamentos) of the e-commerce platform. This is a **critical infrastructure module** that protects all system endpoints and operations from abuse and malicious activities.

This module provides:

- **Rate limiting** (IP-based, phone-based, endpoint-based, combined)
- **Captcha validation** (hCaptcha, reCAPTCHA v3, Turnstile)
- **Active order limits** per phone number (configurable 2-5)
- **Honeypot fields** for bot detection
- **Entity blocking** (IP, phone, email, user agent, fingerprint)
- **Security event logging** with severity levels
- **Form validation** with dynamic rules
- **Security middleware** for HTTP request protection

This module is **dependency-free** and provides services that other modules consume (Orders, Cart, Payments, etc.).

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
- **Transversal module** (used by all modules)
- **No dependencies** on other modules
- **Redis required** for distributed rate limiting
- Merchants **exempt** from rate limits and captcha
- Security events logged **asynchronously** via queue

---

## Domain Model Overview

### Key Services

**RateLimitService:**
- Check if identifier exceeded limit
- Record attempts against limits
- Calculate remaining attempts
- Reset limits manually
- Manage rate limit rules

**CaptchaService:**
- Validate captcha tokens with provider
- Evaluate scores against thresholds
- Support multiple providers (hCaptcha, reCAPTCHA v3, Turnstile)
- Handle provider failures gracefully

**PhoneLimitService:**
- Check if phone reached active order limit
- Increment/decrement counter on order lifecycle
- Enforce configurable maximum (2-5)

**BlockingService:**
- Check if entity is blocked
- Block entities (manual or automatic)
- Unblock entities
- Clean expired blocks

**SecurityEventLogger:**
- Log all security events with full context
- Categorize by type and severity
- Query recent events for analysis
- Trigger alerts for high-severity events

**HoneypotService:**
- Generate honeypot fields for forms
- Validate form submissions
- Track honeypot triggers

**ValidationService:**
- Validate data against dynamic rules
- Apply validation rules by priority
- Return structured validation results

### Key Entities

**RateLimitRule**, **RateLimitAttempt**, **CaptchaValidation**, **PhoneNumberLimit**, **SecurityEvent**, **HoneypotField**, **BlockedEntity**, **ValidationRule**

### Key Value Objects

**PhoneNumber** (normalized E.164), **IpAddress** (IPv4/IPv6, version detection), **RateLimitIdentifier** (composite identifier), **SecurityContext** (full request context), **ValidationResult** (errors + validated data), **CaptchaScore** (0.0-1.0 with threshold)

### Key Enums

**RateLimitType** (IP_BASED, PHONE_BASED, USER_BASED, ENDPOINT_BASED, COMBINED)
**CaptchaProvider** (HCAPTCHA, RECAPTCHA_V3, TURNSTILE, NONE)
**SecurityEventType** (RATE_LIMIT_EXCEEDED, INVALID_CAPTCHA, HONEYPOT_TRIGGERED, BLOCKED_ENTITY_ATTEMPT, etc.)
**SecuritySeverity** (LOW, MEDIUM, HIGH, CRITICAL)
**BlockedEntityType** (IP_ADDRESS, PHONE_NUMBER, EMAIL, USER_AGENT, FINGERPRINT)

---

## Business Rules (CRITICAL)

### Rate Limiting (Rules 1-6)
1. IP-based rate limit: **5 orders per hour** (default, configurable)
2. Phone-based rate limit: **3 orders per hour** (default, configurable)
3. Rate limits are **independent** (both must pass)
4. Exceeded limits return **HTTP 429** with retry-after header
5. Rate limit counters reset after decay period (60 minutes default)
6. **Merchants are exempt** from rate limits

### Active Order Limits (Rules 7-12)
7. Default: **2 active orders per phone** (configurable 2-5)
8. Active states: NEW, CONFIRMED, IN_DELIVERY
9. Counter decrements on: DELIVERED, REJECTED, CANCELLED, REFUNDED
10. Check happens **before** order creation
11. Phone numbers **normalized** before comparison (E.164 format)
12. Limit adjustable per merchant (future)

### Captcha Validation (Rules 13-18)
13. Required for all **order creation** (public checkout)
14. **Invisible captcha** (minimal UX friction)
15. Score threshold: **0.5** (default, configurable 0.0-1.0)
16. Failed validation **blocks** order creation
17. Can be disabled for testing (`CAPTCHA_PROVIDER=none`)
18. **Merchants exempt** from captcha

### Honeypot Protection (Rules 19-22)
19. All public forms include honeypot field(s)
20. Field names **rotated every 24 hours** (configurable)
21. Hidden via **CSS** (not HTML hidden attribute)
22. Any value in field triggers **security event** (HIGH severity)

### Entity Blocking (Rules 23-28)
23. **Automatic blocking** after 5 failed attempts (configurable)
24. **Manual blocking** by merchants (permanent by default)
25. Blocked attempts return **HTTP 403**
26. Temporary blocks expire after configured duration (24h default)
27. Permanent blocks require manual unblock
28. Cleanup job removes expired blocks **daily**

### Security Event Logging (Rules 29-33)
29. All security events logged **regardless of outcome**
30. Retention: **90 days** (configurable)
31. **HIGH/CRITICAL** severity may trigger alerts (email/database)
32. Events indexed for pattern detection
33. Logged **asynchronously** via queue

### Data Integrity (Rules 34-36)
34. All phone numbers stored in **E.164 format** (+5491234567890)
35. All IP addresses **normalized** before storage
36. Rate limit attempts **expire** after decay period (auto-cleanup)

---

## Module Structure

```
Modules/Security/
├── Contracts/                             # Agent A
│   ├── Commands/
│   │   ├── BlockEntityInterface.php
│   │   ├── UnblockEntityInterface.php
│   │   ├── ResetRateLimitInterface.php
│   │   └── IncrementPhoneLimitInterface.php
│   ├── Queries/
│   │   ├── CheckRateLimitInterface.php
│   │   ├── CheckPhoneLimitInterface.php
│   │   ├── ValidateCaptchaInterface.php
│   │   ├── CheckBlockedEntityInterface.php
│   │   └── ValidateHoneypotInterface.php
│   └── Data/
│       ├── RateLimitResult.php
│       ├── CaptchaValidationResult.php
│       ├── PhoneLimitResult.php
│       ├── BlockedEntityData.php
│       ├── SecurityEventData.php
│       └── ValidationResultData.php
├── ValueObjects/                          # Agent A
│   ├── PhoneNumber.php
│   ├── IpAddress.php
│   ├── RateLimitIdentifier.php
│   ├── SecurityContext.php
│   ├── ValidationResult.php
│   └── CaptchaScore.php
├── Enums/                                 # Agent A
│   ├── RateLimitType.php
│   ├── CaptchaProvider.php
│   ├── SecurityEventType.php
│   ├── SecuritySeverity.php
│   └── BlockedEntityType.php
├── Casts/                                 # Agent A
│   ├── PhoneNumberCast.php
│   ├── IpAddressCast.php
│   ├── RateLimitIdentifierCast.php
│   ├── SecurityContextCast.php
│   └── CaptchaScoreCast.php
├── Actions/                               # Agent B
│   ├── Commands/
│   │   ├── BlockEntityAction.php
│   │   ├── UnblockEntityAction.php
│   │   ├── ResetRateLimitAction.php
│   │   ├── IncrementPhoneLimitAction.php
│   │   ├── DecrementPhoneLimitAction.php
│   │   └── LogSecurityEventAction.php
│   ├── Queries/
│   │   ├── CheckRateLimitAction.php
│   │   ├── CheckPhoneLimitAction.php
│   │   ├── ValidateCaptchaAction.php
│   │   ├── CheckBlockedEntityAction.php
│   │   ├── ValidateHoneypotAction.php
│   │   └── GetSecurityEventsAction.php
│   └── Internal/
│       ├── RecordRateLimitAttemptAction.php
│       ├── CalculateRemainingAttemptsAction.php
│       ├── VerifyCaptchaWithProviderAction.php
│       ├── GenerateHoneypotFieldAction.php
│       ├── NormalizePhoneNumberAction.php
│       ├── NormalizeIpAddressAction.php
│       └── BuildSecurityContextAction.php
├── Services/                              # Agent B
│   ├── RateLimitService.php
│   ├── CaptchaService.php
│   ├── PhoneLimitService.php
│   ├── BlockingService.php
│   ├── SecurityEventLogger.php
│   ├── HoneypotService.php
│   └── ValidationService.php
├── Exceptions/                            # Agent B
│   ├── RateLimitExceededException.php
│   ├── PhoneLimitExceededException.php
│   ├── CaptchaValidationFailedException.php
│   ├── HoneypotTriggeredEx exception.php
│   ├── EntityBlockedException.php
│   ├── InvalidPhoneNumberException.php
│   ├── InvalidIpAddressException.php
│   └── SecurityEventException.php
├── Models/                                # Agent C
│   ├── RateLimitRule.php
│   ├── RateLimitAttempt.php
│   ├── CaptchaValidation.php
│   ├── PhoneNumberLimit.php
│   ├── SecurityEvent.php
│   ├── HoneypotField.php
│   ├── BlockedEntity.php
│   └── ValidationRule.php
├── Repositories/                          # Agent C
│   ├── RateLimitRuleRepository.php
│   ├── RateLimitAttemptRepository.php
│   ├── CaptchaValidationRepository.php
│   ├── PhoneNumberLimitRepository.php
│   ├── SecurityEventRepository.php
│   ├── HoneypotFieldRepository.php
│   ├── BlockedEntityRepository.php
│   └── ValidationRuleRepository.php
├── Database/                              # Agent C
│   ├── Factories/
│   │   ├── RateLimitRuleFactory.php
│   │   ├── RateLimitAttemptFactory.php
│   │   ├── CaptchaValidationFactory.php
│   │   ├── PhoneNumberLimitFactory.php
│   │   ├── SecurityEventFactory.php
│   │   ├── HoneypotFieldFactory.php
│   │   ├── BlockedEntityFactory.php
│   │   └── ValidationRuleFactory.php
│   ├── Migrations/
│   │   ├── xxxx_create_rate_limit_rules_table.php
│   │   ├── xxxx_create_rate_limit_attempts_table.php
│   │   ├── xxxx_create_captcha_validations_table.php
│   │   ├── xxxx_create_phone_number_limits_table.php
│   │   ├── xxxx_create_security_events_table.php
│   │   ├── xxxx_create_honeypot_fields_table.php
│   │   ├── xxxx_create_blocked_entities_table.php
│   │   └── xxxx_create_validation_rules_table.php
│   └── Seeders/
│       └── SecuritySeeder.php
├── Http/                                  # Agent D
│   ├── Middleware/
│   │   ├── SecurityMiddleware.php
│   │   ├── RateLimitMiddleware.php
│   │   ├── CaptchaMiddleware.php
│   │   └── BlockedEntityMiddleware.php
│   └── Controllers/
│       └── SecurityAdminController.php
├── Filament/                              # Agent D
│   └── Resources/
│       ├── SecurityEventResource.php
│       ├── BlockedEntityResource.php
│       └── RateLimitRuleResource.php
├── routes/                                # Agent D
│   └── api.php
├── Events/                                # Agent E
│   ├── RateLimitExceeded.php
│   ├── CaptchaValidationFailed.php
│   ├── HoneypotTriggered.php
│   ├── EntityBlocked.php
│   ├── EntityUnblocked.php
│   ├── PhoneLimitReached.php
│   └── SecurityEventCreated.php
├── Listeners/                             # Agent E
│   ├── NotifyHighSeverityEvent.php
│   └── NotifyEntityAutoBlocked.php
├── Jobs/                                  # Agent E
│   ├── CleanupExpiredRateLimitAttempts.php
│   ├── CleanupExpiredBlocks.php
│   ├── RotateHoneypotFields.php
│   ├── CleanupOldSecurityEvents.php
│   └── AnalyzeSecurityPatterns.php
└── Tests/
    ├── Unit/                              # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   ├── Actions/
    │   └── Services/
    └── Feature/                           # Agent D
        ├── RateLimitingTest.php
        ├── PhoneLimitTest.php
        ├── CaptchaValidationTest.php
        ├── HoneypotDetectionTest.php
        ├── EntityBlockingTest.php
        ├── SecurityEventLoggingTest.php
        └── SecurityMiddlewareTest.php
```

---

## Interfaces Exposed (Communication with Other Modules)

### CheckRateLimitInterface

```php
interface CheckRateLimitInterface
{
    /**
     * Check if identifier is within rate limit.
     * 
     * @throws RateLimitExceededException
     */
    public function check(RateLimitIdentifier $identifier, RateLimitType $type): RateLimitResult;
}
```

**Consumed by:** Orders, Cart, Payments

---

### CheckPhoneLimitInterface

```php
interface CheckPhoneLimitInterface
{
    /**
     * Check if phone can create new order.
     * 
     * @throws PhoneLimitExceededException
     */
    public function check(PhoneNumber $phoneNumber): PhoneLimitResult;
}
```

**Consumed by:** Orders

---

### ValidateCaptchaInterface

```php
interface ValidateCaptchaInterface
{
    /**
     * Validate captcha token.
     * 
     * @throws CaptchaValidationFailedException
     */
    public function validate(string $token, IpAddress $ipAddress, string $action): CaptchaValidationResult;
}
```

**Consumed by:** Cart (checkout)

---

### IncrementPhoneLimitInterface

```php
interface IncrementPhoneLimitInterface
{
    /**
     * Increment active orders count for phone.
     */
    public function increment(PhoneNumber $phoneNumber): void;
}
```

**Consumed by:** Orders (on order created)

---

### DecrementPhoneLimitInterface

```php
interface DecrementPhoneLimitInterface
{
    /**
     * Decrement active orders count for phone.
     */
    public function decrement(PhoneNumber $phoneNumber): void;
}
```

**Consumed by:** Orders (on order completed/cancelled)

---

### ValidateHoneypotInterface

```php
interface ValidateHoneypotInterface
{
    /**
     * Validate honeypot fields in form data.
     * 
     * @throws HoneypotTriggeredEx exception
     */
    public function validate(string $formName, array $data): boolean;
}
```

**Consumed by:** Cart (checkout form)

---

## Events Emitted (Asynchronous Communication)

### RateLimitExceeded

```php
final readonly class RateLimitExceeded
{
    public function __construct(
        public RateLimitIdentifier $identifier,
        public RateLimitType $type,
        public int $attempts,
        public SecurityContext $context
    ) {}
}
```

---

### PhoneLimitReached

```php
final readonly class PhoneLimitReached
{
    public function __construct(
        public PhoneNumber $phoneNumber,
        public int $activeCount,
        public int $maxAllowed,
        public Carbon $occurredAt
    ) {}
}
```

---

### EntityBlocked

```php
final readonly class EntityBlocked
{
    public function __construct(
        public BlockedEntityType $entityType,
        public string $entityValue,
        public string $reason,
        public ?Carbon $expiresAt,
        public bool $isAutomatic
    ) {}
}
```

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `PhoneNumber` (countryCode, number, normalizedNumber, Wireable)
     - Methods: `toString()`, `format()`, `normalize()`, `equals()`, `isValid()`
     - Validation: E.164 format, country code required
     - Normalization: +5491234567890
   - `IpAddress` (value, version, isPrivate, isReserved, Wireable)
     - Methods: `toString()`, `normalize()`, `isValid()`, `getVersion()`, `isPrivate()`, `isReserved()`
     - Supports IPv4 and IPv6
   - `RateLimitIdentifier` (value, type, metadata, Wireable)
     - Methods: `toString()`, `hash()`, `equals()`
     - Hash used as Redis cache key
   - `SecurityContext` (ipAddress, userAgent, endpoint, method, headers, timestamp, Wireable)
     - Methods: `toArray()`, `getFingerprint()`
     - Immutable after creation
   - `ValidationResult` (isValid, errors, validatedData, Wireable)
     - Methods: `hasErrors()`, `getErrors()`, `getData()`, `merge()`
   - `CaptchaScore` (value 0.0-1.0, provider, action, Wireable)
     - Methods: `toFloat()`, `meetsThreshold()`, `isHuman()`, `isBot()`

2. **Enums:**
   - `RateLimitType` (IP_BASED, PHONE_BASED, USER_BASED, ENDPOINT_BASED, COMBINED)
   - `CaptchaProvider` (HCAPTCHA, RECAPTCHA_V3, TURNSTILE, NONE)
   - `SecurityEventType` (RATE_LIMIT_EXCEEDED, INVALID_CAPTCHA, HONEYPOT_TRIGGERED, BLOCKED_ENTITY_ATTEMPT, SUSPICIOUS_ACTIVITY, VALIDATION_FAILED, CSRF_MISMATCH, XSS_ATTEMPT, SQL_INJECTION_ATTEMPT)
   - `SecuritySeverity` (LOW, MEDIUM, HIGH, CRITICAL)
   - `BlockedEntityType` (IP_ADDRESS, PHONE_NUMBER, EMAIL, USER_AGENT, FINGERPRINT)

3. **Casts:**
   - `PhoneNumberCast` (string ↔ PhoneNumber)
   - `IpAddressCast` (string ↔ IpAddress)
   - `RateLimitIdentifierCast` (JSON ↔ RateLimitIdentifier)
   - `SecurityContextCast` (JSON ↔ SecurityContext)
   - `CaptchaScoreCast` (float ↔ CaptchaScore)

4. **Data Objects (Spatie Laravel Data):**
   - `RateLimitResult` (isExceeded, remainingAttempts, resetsAt, rule)
   - `CaptchaValidationResult` (isValid, score, provider, errorCodes)
   - `PhoneLimitResult` (canCreateOrder, activeCount, maxAllowed)
   - `BlockedEntityData` (entityType, entityValue, reason, blockedAt, expiresAt, isPermanent)
   - `SecurityEventData` (type, severity, ipAddress, description, context, wasBlocked)
   - `ValidationResultData` (isValid, errors, validatedData)

5. **Contracts (Commands):**
   - `BlockEntityInterface`
   - `UnblockEntityInterface`
   - `ResetRateLimitInterface`
   - `IncrementPhoneLimitInterface`
   - `DecrementPhoneLimitInterface`

6. **Contracts (Queries):**
   - `CheckRateLimitInterface`
   - `CheckPhoneLimitInterface`
   - `ValidateCaptchaInterface`
   - `CheckBlockedEntityInterface`
   - `ValidateHoneypotInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] PhoneNumber validates and normalizes to E.164 format
- [ ] IpAddress supports IPv4 and IPv6 with version detection
- [ ] RateLimitIdentifier hash() generates consistent Redis keys
- [ ] SecurityContext fingerprint combines IP + user agent hash
- [ ] CaptchaScore validates 0.0-1.0 range
- [ ] All Enums have descriptive methods (label(), color(), etc.)
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] Unit tests for all VOs (validation, normalization, equality)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions, Services and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**

1. **Actions Commands:**
   - `BlockEntityAction` (manual or automatic blocking with reason and expiry)
   - `UnblockEntityAction` (remove block, log event)
   - `ResetRateLimitAction` (clear rate limit for identifier)
   - `IncrementPhoneLimitAction` (increment active orders count)
   - `DecrementPhoneLimitAction` (decrement active orders count)
   - `LogSecurityEventAction` (create security event, queue logging)

2. **Actions Queries:**
   - `CheckRateLimitAction` (check Redis + DB, return RateLimitResult)
   - `CheckPhoneLimitAction` (check active orders count, return PhoneLimitResult)
   - `ValidateCaptchaAction` (call provider API, validate score, store result)
   - `CheckBlockedEntityAction` (check if entity is blocked, return boolean)
   - `ValidateHoneypotAction` (check honeypot fields, trigger event if filled)
   - `GetSecurityEventsAction` (query security events with filters)

3. **Actions Internal:**
   - `RecordRateLimitAttemptAction` (store attempt in DB, set Redis key with TTL)
   - `CalculateRemainingAttemptsAction` (count non-expired attempts, calculate remaining)
   - `VerifyCaptchaWithProviderAction` (HTTP call to hCaptcha/reCAPTCHA/Turnstile API)
   - `GenerateHoneypotFieldAction` (generate hidden field name, store in DB)
   - `NormalizePhoneNumberAction` (convert to E.164 format)
   - `NormalizeIpAddressAction` (standardize IPv4/IPv6)
   - `BuildSecurityContextAction` (extract request data into SecurityContext VO)

4. **Services:**
   - `RateLimitService`
     - Dependencies: Redis, RateLimitRuleRepository, RateLimitAttemptRepository
     - Methods: `checkLimit()`, `recordAttempt()`, `getRemainingAttempts()`, `resetLimit()`, `isExceeded()`
   - `CaptchaService`
     - Dependencies: HTTP client, CaptchaValidationRepository, Config
     - Methods: `validate()`, `verifyScore()`, `getProvider()`, `isEnabled()`
   - `PhoneLimitService`
     - Dependencies: PhoneNumberLimitRepository
     - Methods: `checkLimit()`, `canCreateOrder()`, `incrementActiveOrders()`, `decrementActiveOrders()`, `getActiveOrdersCount()`
   - `BlockingService`
     - Dependencies: BlockedEntityRepository
     - Methods: `isBlocked()`, `blockEntity()`, `unblockEntity()`, `getBlockedEntity()`, `cleanExpiredBlocks()`
   - `SecurityEventLogger`
     - Dependencies: SecurityEventRepository, Queue
     - Methods: `log()`, `logRateLimitExceeded()`, `logCaptchaFailed()`, `logHoneypotTriggered()`, `getRecentEvents()`
   - `HoneypotService`
     - Dependencies: HoneypotFieldRepository
     - Methods: `generateField()`, `validate()`, `isTriggered()`, `getActiveFields()`
   - `ValidationService`
     - Dependencies: ValidationRuleRepository, Laravel Validator
     - Methods: `validate()`, `validateField()`, `getActiveRules()`, `addRule()`

5. **Exceptions:**
   - `RateLimitExceededException` (429, "Rate limit exceeded. Try again in {minutes} minutes.")
   - `PhoneLimitExceededException` (422, "Phone {phone} has {count} active orders. Maximum allowed: {max}")
   - `CaptchaValidationFailedException` (422, "Captcha validation failed")
   - `HoneypotTriggeredEx exception` (403, "Honeypot triggered")
   - `EntityBlockedException` (403, "Entity is blocked: {reason}")
   - `InvalidPhoneNumberException` (422)
   - `InvalidIpAddressException` (422)
   - `SecurityEventException` (500)

**Unit Tests:**
- Mock all repositories, Redis, HTTP clients
- Test RateLimitService with various scenarios (within limit, exceeded, reset)
- Test CaptchaService with mocked provider responses
- Test PhoneLimitService increment/decrement logic
- Test BlockingService automatic and manual blocking
- Test HoneypotService field generation and validation
- 100% coverage of business logic

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method
- [ ] All Services are `final`
- [ ] RateLimitService uses Redis for distributed rate limiting
- [ ] CaptchaService handles provider failures gracefully
- [ ] PhoneLimitService thread-safe (uses database locks)
- [ ] BlockingService cleans expired blocks correctly
- [ ] SecurityEventLogger queues events asynchronously
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database, no Redis, no external APIs)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**

1. **Models:**
   - `RateLimitRule` (id ulid, key unique, type enum, max_attempts, decay_minutes, is_active, timestamps)
   - `RateLimitAttempt` (id ulid, rate_limit_rule_id, identifier, endpoint, ip_address inet, user_agent, metadata jsonb, attempted_at, expires_at, indexes)
   - `CaptchaValidation` (id ulid, provider enum, token, ip_address inet, score decimal, is_valid, action, error_codes jsonb, metadata jsonb, validated_at, created_at, indexes)
   - `PhoneNumberLimit` (id ulid, phone_number unique, active_orders_count, max_allowed_orders, last_order_at, timestamps)
   - `SecurityEvent` (id ulid, type enum, ip_address inet, identifier nullable, endpoint, user_agent, severity enum, description, context jsonb, was_blocked, occurred_at, created_at, indexes)
   - `HoneypotField` (id ulid, form_name, field_name, field_type, expected_value, is_active, trigger_count, timestamps, indexes)
   - `BlockedEntity` (id ulid, entity_type enum, entity_value unique composite, reason, blocked_at, expires_at nullable, blocked_by_user_id nullable, is_permanent, is_active, timestamps, indexes)
   - `ValidationRule` (id ulid, field, validator_class, rules jsonb, messages jsonb, priority, is_active, timestamps, indexes)

2. **Repositories:**
   - All repositories implement contracts
   - All use Eloquent query builder
   - Rate limit queries use Redis + DB
   - Phone limit queries use pessimistic locking

3. **Migrations:**
   - All tables use ULID primary keys
   - All enum columns use string type (not database enums)
   - All IP columns use inet type (PostgreSQL) or string (MySQL)
   - All JSON columns use jsonb (PostgreSQL) or json (MySQL)
   - Proper indexes on all foreign keys and filter columns
   - Unique constraints where specified

4. **Factories:**
   - Factories for all models with relevant states
   - RateLimitRule: active/inactive states
   - RateLimitAttempt: expired/active states
   - CaptchaValidation: valid/invalid, high/low score states
   - PhoneNumberLimit: at limit/below limit states
   - SecurityEvent: all severity levels, all types
   - HoneypotField: active/inactive, triggered states
   - BlockedEntity: all entity types, expired/active, permanent/temporary states

5. **Seeders:**
   - `SecuritySeeder`: Default rate limit rules, sample honeypot fields

**Integration Tests:**
- Rate limit attempt expiration
- Phone limit concurrent updates (pessimistic locking)
- Blocked entity expiration
- Security event querying with filters
- Factory states

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs and Enums
- [ ] All models are `final`
- [ ] PhoneNumberLimit uses pessimistic locking for updates
- [ ] RateLimitAttempt auto-expires via expires_at timestamp
- [ ] BlockedEntity unique composite key (entity_type + entity_value)
- [ ] SecurityEvent indexed for fast filtering by type, severity, timestamp
- [ ] All repositories implement contracts
- [ ] Factories for all models with states
- [ ] Migrations with proper indexes and constraints
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Middleware, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **HTTP Middleware:**
   - `SecurityMiddleware` (master middleware, calls all services)
     - Checks: rate limit, captcha, honeypot, blocked entity
     - Logs all security events
     - Builds SecurityContext from request
   - `RateLimitMiddleware` (standalone rate limiting)
   - `CaptchaMiddleware` (standalone captcha validation)
   - `BlockedEntityMiddleware` (standalone entity blocking)

2. **HTTP Controllers:**
   - `SecurityAdminController` (admin endpoints for blocked entities, security events)
     - `GET /api/v1/security/events` - List security events with filters
     - `GET /api/v1/security/blocked` - List blocked entities
     - `POST /api/v1/security/blocked` - Block entity (manual)
     - `DELETE /api/v1/security/blocked/{id}` - Unblock entity
     - `POST /api/v1/security/rate-limit/reset` - Reset rate limit (admin)

3. **API Routes (routes/api.php):**
   - All admin routes protected by Auth middleware
   - Rate limit check endpoint (internal use by other modules)
   - Captcha validation endpoint (internal use by Cart)

4. **Filament Resources (Backoffice):**
   - `SecurityEventResource` (view-only)
     - Table: type badge, severity badge, IP, endpoint, timestamp
     - Filters: type, severity, date range, IP
     - Detail page: full context JSON viewer
   - `BlockedEntityResource` (full CRUD for manual blocks)
     - Table: entity type, entity value, reason, expiry, is_permanent
     - Filters: entity type, expired/active
     - Actions: Unblock, Extend block
   - `RateLimitRuleResource` (view/edit rules)
     - Table: key, type, max attempts, decay minutes, is_active
     - Edit: max attempts, decay minutes, is_active toggle

5. **Filament Widgets:**
   - None (security data too sensitive for dashboard)

**Feature Tests:**
- Rate limiting (IP-based, phone-based, both enforced)
- Phone limit (increment, decrement, reached limit)
- Captcha validation (mock provider, various scores)
- Honeypot detection (empty field passes, filled field blocks)
- Entity blocking (automatic after X attempts, manual by merchant, expiration)
- Security event logging (all event types, severity levels)
- Middleware integration (all checks applied in order)
- Smoke tests for Filament resources

**Acceptance Criteria:**
- [ ] SecurityMiddleware builds SecurityContext from request
- [ ] SecurityMiddleware exempts merchants from rate limits and captcha
- [ ] Rate limit exceeded returns HTTP 429 with Retry-After header
- [ ] Blocked entity returns HTTP 403
- [ ] Honeypot triggered returns HTTP 403
- [ ] Captcha failed returns HTTP 422
- [ ] All security events logged asynchronously
- [ ] Filament resources show real-time data
- [ ] Feature tests cover all scenarios
- [ ] Smoke tests for all Filament resources
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - `RateLimitExceeded` (identifier, type, attempts, context)
   - `CaptchaValidationFailed` (token, score, provider, ipAddress)
   - `HoneypotTriggered` (formName, fieldName, value, context)
   - `EntityBlocked` (entityType, entityValue, reason, expiresAt, isAutomatic)
   - `EntityUnblocked` (entityType, entityValue)
   - `PhoneLimitReached` (phoneNumber, activeCount, maxAllowed)
   - `SecurityEventCreated` (type, severity, context)

2. **Listeners:**
   - `NotifyHighSeverityEvent` (listens to `SecurityEventCreated`)
     - Only if severity = HIGH or CRITICAL
     - Send email to merchant
     - Store database notification
   - `NotifyEntityAutoBlocked` (listens to `EntityBlocked`)
     - Only if isAutomatic = true
     - Send email to merchant
     - Store database notification

3. **Jobs (Scheduled):**
   - `CleanupExpiredRateLimitAttempts` (hourly)
     - Delete rate_limit_attempts where expires_at < now()
   - `CleanupExpiredBlocks` (daily at 3:00 AM)
     - Update blocked_entities set is_active = false where expires_at < now() and !is_permanent
   - `RotateHoneypotFields` (every 24 hours)
     - Deactivate old honeypot fields
     - Generate new honeypot fields with random names
   - `CleanupOldSecurityEvents` (weekly)
     - Delete security_events where created_at < (now() - retention_days)
   - `AnalyzeSecurityPatterns` (daily at 4:00 AM)
     - Detect patterns in security events
     - Identify potential threats
     - Recommend automatic blocks (future)

**Event Tests:**
- Event dispatching on rate limit exceeded
- Event dispatching on captcha failed
- Event dispatching on honeypot triggered
- Event dispatching on entity blocked/unblocked
- Listener execution (mock notifications)
- Job execution (cleanup, rotation)

**Acceptance Criteria:**
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] High-severity listeners send email and database notifications
- [ ] Listeners implement `ShouldQueue` for async processing
- [ ] All jobs are scheduled in kernel
- [ ] CleanupExpiredRateLimitAttempts runs hourly
- [ ] CleanupExpiredBlocks runs daily at 3:00 AM
- [ ] RotateHoneypotFields runs every 24 hours
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
- [ ] Unit tests for all Value Objects
- [ ] Unit tests for all Services
- [ ] Feature tests for all security scenarios
- [ ] Integration tests for database operations
- [ ] Smoke tests for Filament resources
- [ ] 100% coverage for critical paths

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex security rules have inline comments
- [ ] README.md in module root

### Database
- [ ] All migrations with proper indexes
- [ ] Unique constraints enforced
- [ ] Factories for all models with states

### Performance
- [ ] Redis used for rate limiting (distributed)
- [ ] Security events logged asynchronously
- [ ] Proper indexes on all filter columns
- [ ] Cleanup jobs scheduled correctly

### Security
- [ ] Phone numbers normalized to E.164
- [ ] IP addresses normalized
- [ ] Merchants exempt from rate limits and captcha
- [ ] High-severity events trigger alerts
- [ ] All security events logged

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Store phone numbers without normalization (must be E.164)
- Use database enums (use string columns with PHP enums)
- Skip Redis for rate limiting (performance killer)
- Block merchants (they must be exempt)
- Log security events synchronously (use queue)
- Forget to clean up expired data
- Hardcode captcha provider credentials
- Skip honeypot field rotation
- Allow permanent blocks without manual intervention
- Use float for captcha scores (use decimal)

✅ **DO:**
- Normalize phone numbers to E.164 before storage
- Use Redis for distributed rate limiting
- Exempt merchants from all restrictions
- Log security events asynchronously via queue
- Schedule cleanup jobs (attempts, blocks, events)
- Rotate honeypot fields regularly
- Store captcha credentials in config/env
- Use pessimistic locking for phone limit updates
- Allow manual unblock of all entities
- Support multiple captcha providers

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Rate limiting works (IP-based, phone-based, combined)
   - Phone active order limit enforced
   - Captcha validation integrated
   - Honeypot fields detect bots
   - Entity blocking works (automatic and manual)
   - Security events logged for all scenarios
   - Merchants exempt from all restrictions
   - Cleanup jobs scheduled and working

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - Redis used for rate limiting
   - Security events logged asynchronously
   - Proper indexes on all queries
   - Cleanup jobs efficient

4. **Security:**
   - All security scenarios covered
   - High-severity events trigger alerts
   - Phone numbers normalized
   - IP addresses validated
   - No merchant restrictions

---

## Configuration

### Environment Variables

```env
# Rate Limiting
ORDER_RATE_LIMIT_IP=5
ORDER_RATE_LIMIT_PHONE=3
RATE_LIMIT_DECAY_MINUTES=60

# Phone Number Limits
MAX_ACTIVE_ORDERS_PER_PHONE=2

# Captcha
CAPTCHA_ENABLED=true
CAPTCHA_PROVIDER=recaptcha_v3
CAPTCHA_SITE_KEY=your_site_key
CAPTCHA_SECRET_KEY=your_secret_key
CAPTCHA_SCORE_THRESHOLD=0.5

# Honeypot
HONEYPOT_ENABLED=true
HONEYPOT_FIELD_ROTATION_HOURS=24

# Security Events
SECURITY_EVENT_RETENTION_DAYS=90
SECURITY_HIGH_SEVERITY_NOTIFY=true
SECURITY_CRITICAL_SEVERITY_NOTIFY=true

# Blocking
AUTO_BLOCK_THRESHOLD=5
AUTO_BLOCK_DURATION_HOURS=24
CLEANUP_EXPIRED_BLOCKS_ENABLED=true
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
./vendor/bin/sail test Modules/Security
```

---

## Dependencies and Integration

### Modules that Depend on Security

- **Orders**: Rate limiting, phone limits, captcha
- **Cart**: Captcha, honeypot, rate limiting
- **Payments**: Rate limiting
- **All modules**: Security middleware

### External Dependencies

- **Redis**: Distributed rate limiting
- **Captcha providers**: hCaptcha, reCAPTCHA v3, Turnstile APIs
- **Queue**: Async security event logging

---

## Final Notes

- This module is **Phase 1 (Fundamentos)** - implement first as it's used by all modules.
- This is a **transversal module** with no dependencies on other modules.
- **Redis is required** for distributed rate limiting.
- **Merchants must be exempt** from all restrictions.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 20-24 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
