---
task_id: "payments-005-events"
module: "Payments"
agent: "Agente E - Events, Listeners y Jobs"
title: "Payments - Eventos de Dominio y Listeners"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "payments-001-contracts"
  - "payments-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/payments/agents_prompt.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
phase: "Fase 3 - Integraciones"
---

# Task 005: Payments - Eventos de Dominio y Listeners

## 🎯 Objetivo

Implementar eventos de dominio para el ciclo de vida del pago, coordinación con Orders module y notificaciones a merchant.

## 📦 Artefactos a Crear

### Domain Events (8)

1. **PaymentCreated** - Pago creado
   - Payload: PaymentId, OrderId, payment_method, amount, created_at

2. **PaymentStatusChanged** - Estado cambió
   - Payload: PaymentId, old_status, new_status, change_source (manual|webhook), changed_at

3. **PaymentLinkGenerated** - Link generado
   - Payload: PaymentId, payment_link, expires_at

4. **PaymentLinkExpired** - Link expiró
   - Payload: PaymentId, expired_at

5. **PaymentPaid** - Pago confirmado
   - Payload: PaymentId, OrderId, transaction_id, paid_at

6. **PaymentRefunded** - Pago reembolsado
   - Payload: PaymentId, OrderId, refund_amount, refunded_at

7. **WebhookProcessed** - Webhook procesado
   - Payload: webhook_id, PaymentId, provider, processed_at

8. **WebhookFailed** - Webhook falló
   - Payload: webhook_id, provider, error_message, failed_at

### Event Listeners (5)

1. **SyncPaymentStatusWithOrderListener** - Listens to PaymentStatusChanged
   - Updates Order.payment_status
   - Synchronous (critical)
   - Transactional

2. **SendPaymentLinkWhatsAppListener** - Listens to PaymentLinkGenerated
   - Sends payment link via WhatsApp to customer
   - Queued (ShouldQueue)
   - Idempotent
   - Retry on failure

3. **SendPaymentConfirmationWhatsAppListener** - Listens to PaymentPaid
   - Notifies merchant of payment confirmation
   - Queued
   - Idempotent

4. **CreatePaymentAuditLogListener** - Listens to all Payment* events
   - Creates PaymentStatusLog entries
   - Synchronous (must succeed)
   - Never fails (logs error if can't write)

5. **NotifyMerchantPaymentFailureListener** - Listens to PaymentLinkExpired
   - Notifies merchant of payment failure
   - Queued
   - Low priority

### Jobs

1. **CheckExpiredPaymentLinksJob** - Scheduled job
   - Runs every hour
   - Finds expired payment links
   - Marks as failed
   - Emits PaymentLinkExpired events

## 🔑 Event Flow

```
Payment Creation:
  CreatePaymentAction → PaymentCreated
    → CreatePaymentAuditLogListener
    
Payment Link Generation:
  GeneratePaymentLinkAction → PaymentLinkGenerated
    → SendPaymentLinkWhatsAppListener
    → CreatePaymentAuditLogListener

Webhook Processing:
  ProcessWebhookAction → PaymentStatusChanged
    → SyncPaymentStatusWithOrderListener (critical)
    → CreatePaymentAuditLogListener
    (if PAID) → PaymentPaid
      → SendPaymentConfirmationWhatsAppListener

Manual Status Update:
  UpdatePaymentStatusAction → PaymentStatusChanged
    → SyncPaymentStatusWithOrderListener
    → CreatePaymentAuditLogListener

Payment Expiration:
  CheckExpiredPaymentLinksJob → PaymentLinkExpired
    → NotifyMerchantPaymentFailureListener
    → CreatePaymentAuditLogListener
```

## 🔑 Business Rules

### Event Timing
- PaymentCreated: after payment saved
- PaymentStatusChanged: after status updated
- PaymentPaid: when status becomes PAID
- WebhookProcessed: after webhook processed successfully

### Listener Reliability
- Critical listeners (sync with order): synchronous, transactional
- Notification listeners: queued, retryable
- All listeners: idempotent
- Audit log listener: never fails (logs errors)

### Audit Trail
- Every payment event creates audit log
- Logs are immutable
- Captured data: user_id (if manual), change_source, timestamp

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Events dispatched at correct moments
- [ ] Payment status synced with order
- [ ] WhatsApp notifications sent
- [ ] Audit logs created for all changes
- [ ] Expired links handled by scheduled job
- [ ] Listeners are idempotent

### Técnicos
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Critical listeners synchronous
- [ ] Notification listeners queued (ShouldQueue)
- [ ] Event tests verify dispatching
- [ ] Listener tests verify execution
- [ ] Idempotency tested
- [ ] Scheduled job tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## 🧪 Tests Requeridos

```php
describe('Payment Events', function () {
    it('dispatches PaymentCreated on payment creation');
    it('dispatches PaymentStatusChanged on status change');
    it('dispatches PaymentPaid when status becomes PAID');
    it('dispatches WebhookProcessed on webhook success');
});

describe('Payment Listeners', function () {
    it('syncs payment status with order');
    it('sends payment link via WhatsApp');
    it('sends payment confirmation via WhatsApp');
    it('creates audit log on every change');
    it('handles queue failures gracefully');
});

describe('Scheduled Jobs', function () {
    it('checks expired payment links hourly');
    it('marks expired links as failed');
    it('emits events for expired links');
});
```

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Integraciones  
**Depende de:** Task 001, 003
