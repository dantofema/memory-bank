---
task_id: "payments-003-persistence"
module: "Payments"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Payments - Modelos, Repositorios y Persistencia"
priority: "HIGH"
estimated_time: "8 hours"
dependencies:
  - "payments-001-contracts"
  - "orders-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/payments/agents_prompt.md"
  - "@e-commerce-wa-ml/payments/domain_model.md"
phase: "Fase 3 - Integraciones"
---

# Task 003: Payments - Modelos, Repositorios y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Payments: modelos con Casts, repositorios, migraciones con audit trail y factories.

## 📦 Artefactos a Crear

### Modelos Eloquent (3)

1. **Payment** - Pago principal
   - Columns: id, order_id (fk unique), payment_method (enum), payment_provider (enum), payment_status (enum), amount_cents, transaction_id, payment_link_url, payment_link_expires_at, external_payment_data (json), metadata (json), timestamps
   - Relationships: belongsTo(Order), hasMany(PaymentStatusLog), hasMany(WebhookLog)
   - Casts: PaymentId, Money, PaymentMethodType, PaymentProvider, PaymentStatus, TransactionId, ExternalPaymentData, PaymentMetadata
   - Scopes: byStatus(), byMethod(), expired(), pending()

2. **PaymentStatusLog** - Auditoría de cambios de estado
   - Columns: id, payment_id (fk), user_id (fk nullable), old_status, new_status, change_source (manual|webhook), ip_address, user_agent, created_at
   - Relationships: belongsTo(Payment), belongsTo(User)
   - Note: Immutable (append-only)

3. **WebhookLog** - Log de webhooks recibidos
   - Columns: id, payment_id (fk nullable), webhook_id (unique), provider, payload (json), signature, is_valid, processed_at, error_message, created_at
   - Relationships: belongsTo(Payment)
   - Note: Immutable, used for idempotency

### Repositorios (3)

1. **PaymentRepository** - CRUD pagos
   - Methods: find, findByOrder, findByTransactionId, create, update, delete, expiredLinks

2. **PaymentStatusLogRepository** - Solo escritura
   - Methods: create, findByPayment

3. **WebhookLogRepository** - Solo escritura
   - Methods: create, findByWebhookId, findByPayment

### Migraciones (3)

1. **create_payments_table**
   - order_id FK unique (1:1)
   - payment_method, payment_provider, payment_status enums
   - amount_cents integer
   - transaction_id nullable
   - payment_link_url nullable
   - payment_link_expires_at nullable
   - external_payment_data json nullable
   - metadata json nullable
   - Indexes: order_id (unique), transaction_id, payment_status, payment_link_expires_at

2. **create_payment_status_logs_table**
   - payment_id FK
   - Audit fields
   - created_at only
   - Indexes: payment_id, created_at

3. **create_webhook_logs_table**
   - payment_id FK nullable
   - webhook_id unique (idempotency)
   - provider, payload, signature
   - processed_at, error_message
   - Indexes: webhook_id (unique), payment_id, created_at

### Factories (3)

1. **PaymentFactory** - Generate test payments
   - States: pending, paid, refunded, failed, mercado_pago, cash, transfer, with_link, expired_link

2. **PaymentStatusLogFactory** - Generate test logs
   - States: manual_change, webhook_change

3. **WebhookLogFactory** - Generate test webhook logs
   - States: valid, invalid, processed, failed

## �� Database Rules

### Indexes Required
- payments.order_id (unique)
- payments.transaction_id
- payments.payment_status
- payments.payment_link_expires_at
- payment_status_logs.payment_id
- payment_status_logs.created_at
- webhook_logs.webhook_id (unique)
- webhook_logs.payment_id
- webhook_logs.created_at

### Foreign Keys
- payments.order_id → orders.id (unique, 1:1)
- payment_status_logs.payment_id → payments.id (cascade delete)
- payment_status_logs.user_id → users.id (nullable)
- webhook_logs.payment_id → payments.id (nullable)

### Constraints
- payments.order_id unique (one payment per order)
- payments.amount_cents > 0
- webhook_logs.webhook_id unique (idempotency)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Models use Casts for VOs
- [ ] Repositories implement contracts
- [ ] One payment per order enforced
- [ ] Audit logs immutable
- [ ] Webhook idempotency via unique webhook_id
- [ ] Factories generate valid data

### Técnicos
- [ ] All models are `final`
- [ ] All repositories are `final`
- [ ] Factories with states
- [ ] Integration tests with database
- [ ] Migration rollback works
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Integraciones  
**Depende de:** Task 001, Orders 003
