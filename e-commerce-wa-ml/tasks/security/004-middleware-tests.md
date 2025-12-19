---
task_id: "security-004-middleware-tests"
module: "Security"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Security - Middleware, Filament y Tests Feature"
priority: "CRITICAL"
estimated_time: "10 hours"
dependencies:
  - "security-001-contracts"
  - "security-002-actions"
  - "security-003-persistence"
  - "auth-004-http-filament"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/security/agents_prompt.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
phase: "Fase 1 - Fundamentos"
---

# Task 004: Security - Middleware, Filament y Tests Feature

## 🎯 Objetivo

Implementar middleware HTTP para protección de endpoints, Filament resource para gestión de bloqueos y eventos, y tests feature completos. NO hay UI pública.

## 📦 Artefactos a Crear

### HTTP Middleware (5)

1. **RateLimitMiddleware** - Rate limiting global
   - Check rate limit (IP, phone, endpoint)
   - Throw RateLimitExceededException if exceeded
   - Set headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
   - Skip for merchants

2. **CaptchaMiddleware** - Validación de captcha
   - Required on specific routes (checkout, order creation)
   - Validate captcha token
   - Throw CaptchaValidationFailedException if invalid
   - Skip for merchants

3. **BlockedEntityMiddleware** - Verificar bloqueos
   - Check IP, phone blocked
   - Throw EntityBlockedException if blocked
   - Log attempt
   - Skip for merchants

4. **HoneypotMiddleware** - Validación honeypot
   - Check hidden fields empty
   - Silent block if triggered
   - Log event with HIGH severity
   - Skip for merchants

5. **SecurityLoggingMiddleware** - Logging automático
   - Log all requests with security context
   - IP, user agent, endpoint
   - Async (queued)

### Filament Resources (2)

1. **BlockedEntityResource** - Gestión de bloqueos
   - **Table:** entity_type, entity_value, reason, blocked_until, is_permanent
   - **View:** Details + unblock action
   - **Create:** Manual block form (entity, reason, duration)
   - **Actions:** Unblock, Extend block
   - **Filters:** entity_type, reason, active/expired
   - **Bulk actions:** Bulk unblock

2. **SecurityEventResource** - Eventos de seguridad (read-only)
   - **Table:** event_type, severity, ip_address, phone, created_at
   - **View:** Full details (user agent, fingerprint, details JSON)
   - **Filters:** event_type, severity, date range
   - **No create/edit** (append-only)
   - **Charts:** Events by type, Events by severity (timeline)

### Filament Widgets (2)

1. **SecurityAlertsWidget** - Alertas recientes
   - Critical/High severity events (last 24h)
   - Blocked entities count
   - Captcha failure rate

2. **SecurityMetricsWidget** - Métricas de seguridad
   - Rate limit violations today
   - Captcha failures today
   - Honeypot triggers today
   - Active blocks count

### No Public UI
**Note:** Security module is backoffice only. No public components (security is transparent to users).

### Feature Tests (7)

**RateLimitTest.php:**
- Rate limit enforced (IP, phone, endpoint)
- Headers set correctly
- Merchants exempt
- Auto-block after violations
- Manual clear works

**CaptchaValidationTest.php:**
- Valid captcha passes
- Invalid captcha blocked
- Providers tested (hCaptcha, reCAPTCHA, Turnstile)
- Token reuse prevented
- Merchants exempt
- Auto-block after failures

**EntityBlockingTest.php:**
- Blocked IP rejected
- Blocked phone rejected
- Manual block/unblock works
- Expiration works
- Permanent blocks enforced

**HoneypotDetectionTest.php:**
- Empty honeypot passes
- Filled honeypot triggers block
- Silent response (no error)
- Event logged
- Immediate block

**SecurityMiddlewareTest.php:**
- All middleware applied correctly
- Order of execution correct
- Merchants exempt from all
- Error responses correct

**SecurityEventsTest.php:**
- Events logged asynchronously
- Severity levels correct
- Retention enforced

**BackofficeTest.php:**
- Merchant can view blocked entities
- Merchant can block/unblock
- Security events displayed
- Filters work

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Rate limiting works across requests
- [ ] Captcha required on checkout
- [ ] Blocked entities rejected
- [ ] Honeypot detects bots silently
- [ ] Merchants exempt from all checks
- [ ] Security events logged
- [ ] Backoffice fully functional

### Técnicos
- [ ] All middleware work correctly
- [ ] Headers set appropriately
- [ ] Filament resources functional
- [ ] Feature tests cover all flows
- [ ] Redis operations tested
- [ ] Async logging tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001, 002, 003, Auth 004
