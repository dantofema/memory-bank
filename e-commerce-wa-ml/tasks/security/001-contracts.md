---
task_id: "security-001-contracts"
module: "Security"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Security - Contracts, Value Objects, Enums y Data Objects"
priority: "CRITICAL"
estimated_time: "8 hours"
dependencies: []
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/security/agents_prompt.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 1 - Fundamentos"
---

# Task 001: Security - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Security: Value Objects, Enums y DTOs para rate limiting, captcha, bloqueos y eventos de seguridad. Módulo transversal sin dependencias.

## 📋 Contexto

El módulo Security es TRANSVERSAL y fundacional (Fase 1). Protege todos los endpoints del sistema contra abuso y ataques. Sin dependencias de otros módulos.

## 📦 Artefactos a Crear

### Value Objects (10)

1. **IpAddress** - IP con normalización (IPv4/IPv6, Wireable)
2. **PhoneNumber** - Teléfono normalizado (shared, Wireable)
3. **RateLimitKey** - Clave compuesta (ip:endpoint:phone, Wireable)
4. **RateLimitQuota** - Cuota (max, window, Wireable)
5. **UserAgent** - User agent parsed (browser, OS, device, Wireable)
6. **Fingerprint** - Browser fingerprint (hash, Wireable)
7. **CaptchaToken** - Token de captcha (provider, token, Wireable)
8. **SecurityEventId** - ID de evento (UUID, Wireable)
9. **BlockDuration** - Duración de bloqueo (minutes, Wireable)
10. **SeverityScore** - Score de severidad (0-100, Wireable)

### Enums (5)

1. **RateLimitType** - IP_BASED | PHONE_BASED | ENDPOINT_BASED | COMBINED
   - Methods: label(), cachePrefix(), defaultQuota()
   
2. **CaptchaProvider** - HCAPTCHA | RECAPTCHA_V3 | TURNSTILE
   - Methods: label(), verifyUrl(), requiresScore()
   
3. **SecurityEventType** - RATE_LIMIT_EXCEEDED | CAPTCHA_FAILED | BOT_DETECTED | BLOCKED_ENTITY | HONEYPOT_TRIGGERED | SUSPICIOUS_BEHAVIOR
   - Methods: label(), severity(), color()
   
4. **BlockReason** - RATE_LIMIT | CAPTCHA_FAILED | MANUAL | BOT_DETECTED | MULTIPLE_VIOLATIONS
   - Methods: label(), defaultDuration()
   
5. **EntityType** - IP_ADDRESS | PHONE_NUMBER | EMAIL | USER_AGENT | FINGERPRINT
   - Methods: label(), normalizationMethod()

### Casts (7)

1. **IpAddressCast** - string ↔ IpAddress
2. **PhoneNumberCast** - string ↔ PhoneNumber (shared)
3. **RateLimitKeyCast** - string ↔ RateLimitKey
4. **UserAgentCast** - string ↔ UserAgent
5. **FingerprintCast** - string ↔ Fingerprint
6. **CaptchaTokenCast** - json ↔ CaptchaToken
7. **BlockDurationCast** - int ↔ BlockDuration

### Data Objects (7)

1. **RateLimitCheckData** - Input for rate limit check
   - Fields: ip, phone, endpoint, user_agent

2. **RateLimitResultData** - Result of rate limit check
   - Fields: allowed, remaining, reset_at, limit_type

3. **CaptchaValidationData** - Input for captcha validation
   - Fields: token, provider, ip_address

4. **SecurityEventData** - Security event for logging
   - Fields: event_type, severity, ip, phone, details, timestamp

5. **BlockedEntityData** - Blocked entity info
   - Fields: entity_type, entity_value, reason, blocked_until, notes

6. **HoneypotCheckData** - Honeypot validation result
   - Fields: triggered, field_name, field_value, timestamp

7. **SecurityMetricsData** - Metrics for dashboard
   - Fields: blocked_ips, blocked_phones, events_today, captcha_failures

## 🔑 Business Rules

### Rate Limiting Rules
- IP-based: 60 requests/minute (default)
- Phone-based: 3 orders/hour
- Combined: IP + phone + endpoint
- Window: sliding 60 seconds
- Merchants exempt from limits

### Captcha Rules
- Required on: checkout, order creation
- Threshold: score >= 0.5 (reCAPTCHA v3)
- Merchants exempt
- Max 3 failed attempts before block
- Token single-use only

### Blocking Rules
- Auto-block on: 5 rate limit violations
- Block duration: 15 minutes (configurable)
- Manual unblock by merchant
- Block types: temporary, permanent
- Block inheritance (IP blocks all associated phones)

### Honeypot Rules
- Fields: email_confirm, phone_confirm (hidden)
- Trigger on any value
- Silent fail (no error shown)
- Log event with severity HIGH
- Immediate block

## ✅ Criterios de Aceptación

### Funcionales
- [ ] IpAddress normalizes IPv4/IPv6
- [ ] PhoneNumber shared with Cart/Orders
- [ ] RateLimitKey combines multiple identifiers
- [ ] CaptchaToken validates provider-specific format
- [ ] SeverityScore ranges 0-100
- [ ] All VOs validate in constructor
- [ ] All Casts bidirectional

### Técnicos
- [ ] All classes are `final`
- [ ] VOs are `readonly`
- [ ] Strong typing everywhere
- [ ] Unit tests for all VOs (100%)
- [ ] Tests for Enums
- [ ] Tests for Casts
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Bloqueante para:** All modules (transversal)
