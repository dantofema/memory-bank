---
task_id: "security-005-events"
module: "Security"
agent: "Agente E - Events, Listeners y Jobs"
title: "Security - Eventos, Listeners y Jobs"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "security-001-contracts"
  - "security-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/security/agents_prompt.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
phase: "Fase 1 - Fundamentos"
---

# Task 005: Security - Eventos, Listeners y Jobs

## 🎯 Objetivo

Implementar eventos de seguridad, listeners para reaccionar a violaciones y jobs programados para limpieza de datos.

## 📦 Artefactos a Crear

### Domain Events (6)

1. **EntityBlocked** - Entidad bloqueada
   - Payload: entity_type, entity_value, reason, blocked_until, is_permanent

2. **EntityUnblocked** - Entidad desbloqueada
   - Payload: entity_type, entity_value, unblocked_at

3. **RateLimitExceeded** - Rate limit excedido
   - Payload: limit_type, identifier, endpoint, exceeded_by

4. **CaptchaFailed** - Captcha falló
   - Payload: provider, ip_address, score (if applicable), attempt_number

5. **HoneypotTriggered** - Honeypot activado
   - Payload: ip_address, field_name, field_value, user_agent

6. **SecurityEventRecorded** - Evento registrado
   - Payload: event_type, severity, ip_address, phone, details

### Event Listeners (4)

1. **RecordSecurityEventListener** - Listens to all Security* events
   - Records SecurityEvent to database
   - Async (queued)
   - Idempotent

2. **AutoBlockOnViolationsListener** - Listens to RateLimitExceeded, CaptchaFailed
   - Count violations per entity
   - Auto-block after threshold (5 rate limits, 3 captchas)
   - Emit EntityBlocked event

3. **NotifyMerchantOnCriticalEventListener** - Listens to HoneypotTriggered, EntityBlocked (CRITICAL)
   - Send notification to merchant (email/dashboard)
   - Queued
   - Low priority

4. **ClearRateLimitOnUnblockListener** - Listens to EntityUnblocked
   - Clear Redis rate limit counters
   - Allow fresh start

### Scheduled Jobs (3)

1. **CleanExpiredBlocksJob** - Limpia bloqueos expirados
   - Runs every hour
   - Delete expired blocks
   - Emit EntityUnblocked events

2. **CleanOldSecurityEventsJob** - Limpia eventos antiguos
   - Runs daily at 3 AM
   - Delete events older than 90 days
   - Log cleanup stats

3. **GenerateSecurityReportJob** - Genera reporte diario
   - Runs daily at 6 AM
   - Summary of yesterday's security events
   - Send to merchant (optional)
   - Store as file (optional)

## 🔑 Event Flow

```
Rate Limit Exceeded:
  Request → RateLimitMiddleware
    → RateLimitExceeded event
      → RecordSecurityEventListener (log)
      → AutoBlockOnViolationsListener (count, maybe block)

Captcha Failed:
  Form Submit → CaptchaMiddleware
    → CaptchaFailed event
      → RecordSecurityEventListener (log)
      → AutoBlockOnViolationsListener (count, maybe block)

Honeypot Triggered:
  Form Submit → HoneypotMiddleware
    → HoneypotTriggered event
      → RecordSecurityEventListener (log)
      → EntityBlocked event (immediate)
        → NotifyMerchantOnCriticalEventListener

Entity Blocked:
  (Any source) → EntityBlocked event
    → NotifyMerchantOnCriticalEventListener (if CRITICAL)

Entity Unblocked:
  Manual/Expiration → EntityUnblocked event
    → ClearRateLimitOnUnblockListener

Scheduled:
  Every hour → CleanExpiredBlocksJob
  Daily 3 AM → CleanOldSecurityEventsJob
  Daily 6 AM → GenerateSecurityReportJob
```

## 🔑 Business Rules

### Event Dispatching
- All security actions emit events
- Events dispatched synchronously (immediate)
- Listeners queued (async processing)

### Auto-Blocking
- 5 rate limit violations → block 15 minutes
- 3 captcha failures → block 15 minutes
- 1 honeypot trigger → block 1 hour
- Violations reset after successful action

### Cleanup
- Expired blocks deleted hourly
- Security events retained 90 days
- Daily security report generated

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Events dispatched correctly
- [ ] Security events logged asynchronously
- [ ] Auto-block triggers on thresholds
- [ ] Expired blocks cleaned hourly
- [ ] Old events deleted after 90 days
- [ ] Listeners are idempotent

### Técnicos
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] All jobs are `final`
- [ ] Listeners queued (ShouldQueue)
- [ ] Jobs scheduled correctly
- [ ] Event tests verify dispatching
- [ ] Listener tests verify execution
- [ ] Job tests verify cleanup
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001, 003
