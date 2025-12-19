---
task_id: "payments-001-contracts"
module: "Payments"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Payments - Contracts, Value Objects, Enums y Data Objects"
priority: "HIGH"
estimated_time: "8 hours"
dependencies:
  - "orders-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/payments/agents_prompt.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 3 - Integraciones"
---

# Task 001: Payments - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Payments: Value Objects, Enums, Casts y DTOs para gestión de métodos de pago, integraciones externas (Mercado Pago) y procesamiento de webhooks.

## 📋 Contexto

El módulo Payments gestiona pagos externos (Mercado Pago), pagos manuales (Cash, Transfer) y procesa webhooks de forma idempotente con verificación de firma.

## 📦 Artefactos a Crear

### Value Objects (8)

1. **PaymentId** - ID de pago (int-based, Wireable)
2. **TransactionId** - ID de transacción externa (string, Wireable)
3. **PaymentLinkId** - ID de link de pago (UUID, Wireable)
4. **Money** - Valor monetario en cents (shared, Wireable)
5. **PaymentLink** - URL de pago + metadata (Wireable)
6. **WebhookSignature** - Firma de webhook para verificación (readonly)
7. **ExternalPaymentData** - Datos de pago externo (provider, transaction_id, status, Wireable)
8. **PaymentMetadata** - Metadata adicional (JSON, Wireable)

### Enums (3)

1. **PaymentMethodType** - MERCADO_PAGO | CASH | TRANSFER
   - Methods: label(), icon(), requiresExternalLink(), isManual()
   
2. **PaymentStatus** - PENDING | PAID | REFUNDED | FAILED
   - Methods: canTransitionTo(), isTerminal(), label(), color()
   
3. **PaymentProvider** - MERCADO_PAGO | MANUAL
   - Methods: label(), webhookUrl(), requiresSignature()

### Casts (6)

1. **PaymentIdCast** - int ↔ PaymentId
2. **TransactionIdCast** - string ↔ TransactionId
3. **PaymentLinkIdCast** - string ↔ PaymentLinkId
4. **MoneyCast** - cents ↔ Money (shared)
5. **ExternalPaymentDataCast** - json ↔ ExternalPaymentData
6. **PaymentMetadataCast** - json ↔ PaymentMetadata

### Data Objects (5)

1. **PaymentData** - Complete payment info with method, status, amount, links
2. **CreatePaymentData** - Input for payment creation from order
3. **UpdatePaymentStatusData** - Input for manual status update
4. **WebhookPayloadData** - Webhook data with signature verification
5. **PaymentLinkData** - Payment link with expiration and metadata

## 🔑 Business Rules

### Payment Method Rules
- Three methods: Mercado Pago (external), Cash (manual), Transfer (manual)
- Mercado Pago requires external link generation
- Cash and Transfer managed manually by merchant
- Payment method immutable after creation

### Payment Status Rules
- Initial status: PENDING
- Valid transitions: PENDING → PAID | FAILED, PAID → REFUNDED
- Terminal states: PAID, REFUNDED, FAILED
- Status can be updated manually (by merchant) or automatically (webhook)

### Payment Link Rules
- Generated for Mercado Pago payments only
- Expires after 24 hours (configurable)
- One link per payment (immutable)
- Link contains: payment_id, order_id, amount, expiration

### Webhook Rules
- Must verify signature (HMAC SHA256)
- Must be idempotent (handle duplicates)
- Must validate payload structure
- Must check payment exists
- Must log all webhook attempts (success/failure)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] PaymentMethodType distinguishes external vs manual
- [ ] PaymentStatus state machine enforced
- [ ] PaymentLink validates expiration
- [ ] WebhookSignature verifies HMAC correctly
- [ ] ExternalPaymentData maps to provider schema
- [ ] All VOs validate in constructor
- [ ] All Casts bidirectional

### Técnicos
- [ ] All classes are `final`
- [ ] VOs are `readonly`
- [ ] Strong typing everywhere
- [ ] Unit tests for all VOs (100%)
- [ ] Tests for Enums (state machine)
- [ ] Tests for Casts
- [ ] Webhook signature verification tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Integraciones  
**Bloqueante para:** Mercado Pago integration, Payment tracking
