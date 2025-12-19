---
task_id: "security-002-actions"
module: "Security"
agent: "Agente B - Actions y Tests Unitarios"
title: "Security - Actions de Lógica de Negocio"
priority: "CRITICAL"
estimated_time: "12 hours"
dependencies:
  - "security-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/security/agents_prompt.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
phase: "Fase 1 - Fundamentos"
---

# Task 002: Security - Actions de Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Security: rate limiting con Redis, validación de captcha, bloqueos de entidades, detección de honeypots y logging de eventos. Transversal sin dependencias.

## 📦 Artefactos a Crear

### Actions Commands (7)

1. **BlockEntityAction** - Bloquear entidad
   - Block type (IP, phone, email, etc.)
   - Duration or permanent
   - Reason
   - Store in database + cache
   - Emit EntityBlocked event

2. **UnblockEntityAction** - Desbloquear entidad
   - Check exists
   - Remove from database + cache
   - Emit EntityUnblocked event

3. **RecordSecurityEventAction** - Registrar evento
   - Event type + severity
   - IP, phone, user agent, details
   - Queue for async processing
   - Emit SecurityEventRecorded event

4. **IncrementRateLimitCounterAction** - Incrementar contador
   - Redis INCR with TTL
   - Sliding window
   - Return remaining

5. **ClearRateLimitAction** - Limpiar rate limit
   - Manual clear by merchant
   - Useful for testing
   - Emit RateLimitCleared event

6. **RecordCaptchaAttemptAction** - Registrar intento captcha
   - Success/failure
   - Provider, score
   - Auto-block after 3 failures

7. **TriggerHoneypotAction** - Activar honeypot
   - Log HIGH severity event
   - Block entity immediately
   - Silent (no user notification)

### Actions Queries (8)

8. **CheckRateLimitAction** - Verificar rate limit
   - Check Redis counter
   - Return allowed + remaining
   - Multiple limit types (IP, phone, combined)

9. **ValidateCaptchaAction** - Validar captcha
   - Call provider API (hCaptcha/reCAPTCHA/Turnstile)
   - Check score (if applicable)
   - Check token not reused
   - Cache valid tokens (5 min)

10. **CheckEntityBlockedAction** - Verificar bloqueo
    - Check cache first (fast)
    - Fallback to database
    - Return blocked status + reason

11. **CheckActiveOrdersLimitAction** - Verificar límite órdenes
    - Count active orders by phone
    - Max N active orders (default: 2)
    - Used by Orders module

12. **ValidateHoneypotFieldsAction** - Validar honeypots
    - Check hidden fields empty
    - Return triggered status

13. **GetRateLimitStatusAction** - Obtener estado rate limit
    - Current count
    - Remaining
    - Reset time

14. **GetSecurityMetricsAction** - Métricas de seguridad
    - Blocked entities count
    - Events today
    - Captcha failure rate

15. **GetBlockedEntitiesAction** - Listar entidades bloqueadas
    - Filter by type
    - Active only or all
    - Pagination

### Actions Internal (5)

16. **NormalizeIdentifierAction** - Normalizar identificador
    - IP normalization (IPv4/IPv6)
    - Phone normalization
    - Email normalization

17. **BuildRateLimitCacheKeyAction** - Construir clave cache
    - Combine identifiers
    - Prefix by type

18. **CalculateSeverityScoreAction** - Calcular score severidad
    - Based on event type
    - Historical violations
    - Return 0-100

19. **ShouldExemptFromSecurityAction** - Verificar exención
    - Merchants exempt
    - Admin users exempt
    - Testing mode exempt

20. **ParseUserAgentAction** - Parsear user agent
    - Extract browser, OS, device
    - Detect bots

### Excepciones de Dominio (8)

- RateLimitExceededException (429)
- CaptchaValidationFailedException (422)
- EntityBlockedException (403)
- HoneypotTriggered Exception (403)
- InvalidCaptchaTokenException (422)
- BlockedEntityNotFoundException (404)
- SecurityEventNotRecordedException (500)
- CaptchaProviderUnavailableException (503)

## 🔑 Key Business Rules

### Rate Limiting (Rules 1-8)
1. IP-based: 60 requests/minute
2. Phone-based: 3 orders/hour
3. Endpoint-specific limits configurable
4. Sliding window implementation
5. Redis storage for distributed systems
6. Merchants exempt from all limits
7. Auto-block after 5 violations
8. Reset counter manually allowed

### Captcha (Rules 9-14)
9. Required on checkout, order creation
10. Three providers supported (hCaptcha, reCAPTCHA, Turnstile)
11. Score threshold >= 0.5 (reCAPTCHA v3)
12. Token single-use (cached 5 minutes)
13. Max 3 failed attempts before block
14. Merchants exempt from captcha

### Blocking (Rules 15-20)
15. Auto-block triggers: 5 rate limit violations, 3 captcha failures, honeypot
16. Manual block by merchant
17. Block duration: 15 minutes (configurable) or permanent
18. Block types: IP, phone, email, user agent, fingerprint
19. Unblock: manual only or expiration
20. Block cascade: blocked IP blocks associated phones

### Honeypot (Rules 21-24)
21. Hidden fields: email_confirm, phone_confirm
22. Trigger on any value
23. Silent block (no error to user)
24. HIGH severity event logged

### Security Events (Rules 25-28)
25. All security actions logged
26. Severity levels: LOW, MEDIUM, HIGH, CRITICAL
27. Async logging (queued)
28. Retention: 90 days

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Rate limiting works with Redis
- [ ] Captcha validates with all providers
- [ ] Entity blocking enforced
- [ ] Honeypot detects bots
- [ ] Security events logged asynchronously
- [ ] Merchants exempt from all checks
- [ ] Auto-block triggers correctly

### Técnicos
- [ ] All Actions are `final`
- [ ] One public method per Action
- [ ] Dependency injection
- [ ] Unit tests with mocks
- [ ] 100% coverage critical paths
- [ ] Redis operations tested
- [ ] API calls mocked
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001
