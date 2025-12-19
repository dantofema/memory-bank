---
task_id: "payments-002-actions"
module: "Payments"
agent: "Agente B - Actions y Tests Unitarios"
title: "Payments - Actions de Lógica de Negocio"
priority: "HIGH"
estimated_time: "12 hours"
dependencies:
  - "payments-001-contracts"
  - "orders-002-actions"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/payments/agents_prompt.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
phase: "Fase 3 - Integraciones"
---

# Task 002: Payments - Actions de Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Payments: integración con Mercado Pago, procesamiento de webhooks idempotente, gestión manual de pagos y lifecycle de estados.

## 📦 Artefactos a Crear

### Actions Commands (7)

1. **CreatePaymentAction** - Crear pago desde orden
   - Determine payment method
   - Generate payment link if Mercado Pago
   - Create payment record
   - Initial status: PENDING
   - Emit PaymentCreated event

2. **GeneratePaymentLinkAction** - Generar link de Mercado Pago
   - Call Mercado Pago API
   - Create preference with order data
   - Set expiration (24h)
   - Store link + external_id
   - Emit PaymentLinkGenerated event

3. **UpdatePaymentStatusAction** - Actualizar estado manualmente
   - Validate merchant permission
   - Validate status transition
   - Update status
   - Create audit log
   - Emit PaymentStatusChanged event
   - Update order payment status

4. **ProcessWebhookAction** - Procesar webhook (idempotente)
   - Verify webhook signature
   - Validate payload structure
   - Check idempotency (webhook_id)
   - Extract payment status
   - Update payment status
   - Create webhook log
   - Emit WebhookProcessed event

5. **RefundPaymentAction** - Procesar reembolso
   - Validate payment can be refunded (status PAID)
   - Call Mercado Pago refund API (if applicable)
   - Update status to REFUNDED
   - Create audit log
   - Emit PaymentRefunded event

6. **ExpirePaymentLinkAction** - Expirar link vencido
   - Check expiration date
   - Mark link as expired
   - Update payment status to FAILED (if still PENDING)
   - Emit PaymentLinkExpired event

7. **RetryFailedPaymentAction** - Reintentar pago fallido
   - Validate payment can be retried
   - Generate new payment link
   - Update expiration
   - Emit PaymentRetried event

### Actions Queries (5)

8. **GetPaymentAction** - Obtener pago por ID
9. **ListPaymentsAction** - Listar pagos con filtros
10. **GetPaymentByOrderAction** - Pago por orden
11. **ValidateWebhookSignatureAction** - Validar firma de webhook
12. **CheckPaymentLinkExpirationAction** - Verificar expiración

### Actions Internal (4)

13. **CallMercadoPagoAPIAction** - Wrapper para API de Mercado Pago
14. **ValidatePaymentStatusTransitionAction** - Validar transición válida
15. **CreatePaymentAuditLogAction** - Crear entrada de auditoría
16. **SyncPaymentWithOrderAction** - Sincronizar estado con orden

### Excepciones de Dominio (9)

- PaymentNotFoundException (404)
- InvalidPaymentMethodException (422)
- InvalidPaymentStatusTransitionException (422)
- PaymentLinkExpiredException (422)
- WebhookSignatureInvalidException (401)
- WebhookAlreadyProcessedException (409)
- MercadoPagoAPIException (502)
- PaymentAlreadyRefundedException (422)
- CannotRefundPendingPaymentException (422)

## 🔑 Key Business Rules

### Payment Creation (Rules 1-5)
1. One payment per order
2. Payment method from order
3. Mercado Pago → generate external link
4. Cash/Transfer → manual management
5. Initial status: PENDING

### Payment Link (Rules 6-10)
6. Only for Mercado Pago
7. Expires after 24h (configurable)
8. Contains: order data, amount, return URLs
9. Immutable after creation
10. One link per payment (no regeneration unless retry)

### Status Transitions (Rules 11-16)
11. PENDING → PAID | FAILED
12. PAID → REFUNDED
13. FAILED → terminal (no transitions)
14. REFUNDED → terminal (no transitions)
15. Manual update by merchant allowed
16. Webhook update automatic

### Webhook Processing (Rules 17-23)
17. Signature verification mandatory (HMAC SHA256)
18. Idempotency by webhook_id (ignore duplicates)
19. Payload validation before processing
20. Payment must exist
21. Log all webhook attempts
22. Emit event on success
23. Retry on transient failures (queue)

### Refunds (Rules 24-27)
24. Only PAID payments can be refunded
25. Call Mercado Pago API if external
26. Manual refunds for Cash/Transfer
27. Create audit log entry

## ✅ Criterios de Aceptación

### Funcionales
- [ ] CreatePayment handles all methods correctly
- [ ] Payment link generated for Mercado Pago
- [ ] Webhook processing is idempotent
- [ ] Webhook signature verified
- [ ] Status transitions validated
- [ ] Refunds processed correctly
- [ ] Payment link expiration handled
- [ ] Audit trail complete

### Técnicos
- [ ] All Actions are `final`
- [ ] One public method per Action
- [ ] Dependency injection
- [ ] Unit tests with mocks
- [ ] 100% coverage critical paths (webhook, transitions)
- [ ] Idempotency tested
- [ ] Signature verification tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Integraciones  
**Depende de:** Task 001, Orders 002
