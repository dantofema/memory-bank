---
task_id: "payments-004-filament-webhooks"
module: "Payments"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Payments - Filament, Webhooks y Tests Feature"
priority: "HIGH"
estimated_time: "10 hours"
dependencies:
  - "payments-001-contracts"
  - "payments-002-actions"
  - "payments-003-persistence"
  - "orders-004-filament-tests"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/payments/agents_prompt.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
phase: "Fase 3 - Integraciones"
---

# Task 004: Payments - Filament, Webhooks y Tests Feature

## 🎯 Objetivo

Implementar Filament resources para backoffice (merchant gestiona pagos), webhook endpoint para Mercado Pago y tests feature completos. NO hay UI pública (pagos gestionados desde Orders/Cart).

## 📦 Artefactos a Crear

### Webhook Controller (1)

1. **MercadoPagoWebhookController** - Endpoint para webhooks
   - Route: POST /api/webhooks/mercado-pago
   - Verify signature (HMAC SHA256)
   - Validate payload
   - Check idempotency (webhook_id)
   - Process payment status update
   - Log webhook attempt
   - Return 200 OK (always, even if duplicate)
   - NO authentication required (verified by signature)

### Filament Resources (1)

1. **PaymentResource** - Gestión de pagos
   - **View Page:** Ver detalle completo del pago
     - Payment info (method, status, amount, transaction_id)
     - Order reference
     - Payment link (if Mercado Pago)
     - Status timeline (audit log)
     - Webhook logs
     - Actions: Update Status, Generate New Link (if expired), Refund
   
   - **Table Page:** Listado de pagos
     - Columns: order_number, payment_method, amount, status, created_at
     - Filters: payment_method, payment_status, date range
     - Actions: view, update status, refund
     - Bulk actions: export to CSV
   
   - **Custom Actions:**
     - UpdatePaymentStatusAction (modal with status selector, only manual methods)
     - RefundPaymentAction (confirmation modal, only PAID status)
     - GenerateNewLinkAction (regenerate link if expired)
   
   - **Relation Managers:**
     - PaymentStatusLogsRelationManager (read-only, audit trail)
     - WebhookLogsRelationManager (read-only, webhook history)

### No Public UI
**Note:** Payments module has NO public frontend. Payment links are generated from Cart checkout and sent via WhatsApp. This task focuses on backoffice management and webhook processing.

### Feature Tests (6)

**PaymentCreationTest.php:**
- Create payment from order
- Generate Mercado Pago link
- Manual payment (Cash/Transfer) created without link
- Payment linked to order (1:1)

**WebhookProcessingTest.php:**
- Valid webhook processed successfully
- Invalid signature rejected (401)
- Duplicate webhook ignored (idempotent)
- Webhook updates payment status
- Webhook creates audit log
- Webhook logs all attempts

**PaymentStatusUpdateTest.php:**
- Manual status update by merchant
- Status transition validation
- Audit log created
- Order payment status synced

**PaymentLinkExpirationTest.php:**
- Link expires after 24h
- Expired link marked as failed
- New link can be generated

**RefundTest.php:**
- Refund PAID payment
- Cannot refund PENDING payment
- Mercado Pago API called (if external)
- Audit log created

**BackofficeTest.php:**
- Merchant can view payments
- Merchant can update status manually
- Merchant can refund payments
- Filters work correctly
- Webhook logs displayed

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Webhook endpoint processes Mercado Pago webhooks
- [ ] Signature verification works
- [ ] Idempotency enforced (webhook_id)
- [ ] Payment status updated from webhook
- [ ] Merchant can view payments in Filament
- [ ] Merchant can update status manually
- [ ] Merchant can refund payments
- [ ] Audit trail complete
- [ ] Webhook logs visible

### Técnicos
- [ ] Webhook controller handles all cases
- [ ] Signature verification tested
- [ ] Idempotency tested
- [ ] Feature tests cover all flows
- [ ] Webhook security tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Integraciones  
**Depende de:** Task 001, 002, 003, Orders 004
