---
task_id: "orders-005-events"
module: "Orders"
agent: "Agente E - Events, Listeners y Jobs"
title: "Orders - Eventos de Dominio y Listeners"
priority: "HIGH"
estimated_time: "8 hours"
dependencies:
  - "orders-001-contracts"
  - "orders-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/orders/agents_prompt.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 005: Orders - Eventos de Dominio y Listeners

## 🎯 Objetivo

Implementar eventos de dominio para el ciclo de vida del pedido, coordinación con otros módulos (WhatsApp, Stock) y auditoría completa.

## 📦 Artefactos a Crear

### Domain Events (10)

1. **OrderCreated** - Pedido creado
   - Payload: OrderId, order_number, CustomerData, OrderTotals, created_at
   - Triggers: WhatsApp notification, Analytics tracking

2. **OrderUpdated** - Pedido actualizado
   - Payload: OrderId, order_number, changed_fields, updated_at
   - Triggers: Audit log, Merchant notification (if significant changes)

3. **OrderStatusChanged** - Estado del pedido cambió
   - Payload: OrderId, order_number, old_status, new_status, changed_by, changed_at
   - Triggers: WhatsApp notification, Audit log, Stock release (if cancelled)

4. **PaymentStatusChanged** - Estado de pago cambió
   - Payload: OrderId, order_number, old_status, new_status, changed_at
   - Triggers: Merchant notification, Audit log

5. **OrderCancelled** - Pedido cancelado
   - Payload: OrderId, order_number, reason, cancelled_by, cancelled_at
   - Triggers: Stock release, Customer notification, Audit log

6. **OrderDelivered** - Pedido entregado
   - Payload: OrderId, order_number, delivered_at
   - Triggers: Customer feedback request (future), Analytics

7. **OrderItemQuantityUpdated** - Cantidad de item actualizada
   - Payload: OrderId, OrderItemId, old_quantity, new_quantity, updated_at
   - Triggers: Totals recalculation, Audit log

8. **StockReservedForOrder** - Stock reservado
   - Payload: OrderId, items (ProductId, Quantity)[], reserved_at
   - Triggers: Internal tracking, Analytics

9. **StockReleasedForOrder** - Stock liberado
   - Payload: OrderId, items (ProductId, Quantity)[], released_at, reason
   - Triggers: Internal tracking, Analytics

10. **OrderNoteAdded** - Nota agregada al pedido
    - Payload: OrderId, note, added_by, added_at
    - Triggers: Audit log

### Event Listeners (5)

1. **SendOrderWhatsAppNotificationListener** - Listens to OrderCreated
   - Sends WhatsApp message to merchant with order details
   - Queued (ShouldQueue)
   - Idempotent
   - Retry on failure (3 attempts)

2. **SendStatusChangeWhatsAppNotificationListener** - Listens to OrderStatusChanged
   - Sends WhatsApp message to merchant when status changes
   - Different message templates per status
   - Queued
   - Idempotent

3. **ReleaseStockOnCancellationListener** - Listens to OrderCancelled, OrderStatusChanged (REJECTED)
   - Releases reserved stock back to inventory
   - Transactional
   - Idempotent (checks if already released)
   - Critical path (must not fail)

4. **CreateOrderAuditLogListener** - Listens to all Order* events
   - Creates OrderStatusLog entries
   - Captures user, IP, timestamp
   - Synchronous (must succeed before event completes)
   - Never fails (logs error if can't write)

5. **RecalculateOrderTotalsListener** - Listens to OrderItemQuantityUpdated
   - Recalculates subtotal, discount, total
   - Synchronous
   - Transactional

### Jobs

**None for MVP** (notifications handled by listeners)
Future: Async notification retries, Order reminders, Abandoned checkout recovery

## 🔑 Event Flow

```
Order Creation:
  Cart Checkout → CreateOrderAction
    → StockReservedForOrder
    → OrderCreated
      → SendOrderWhatsAppNotificationListener
      → CreateOrderAuditLogListener

Status Change:
  Merchant Action → ChangeOrderStatusAction
    → OrderStatusChanged
      → SendStatusChangeWhatsAppNotificationListener
      → CreateOrderAuditLogListener
      → (if CANCELLED/REJECTED) → ReleaseStockOnCancellationListener

Order Cancellation:
  Merchant/Customer → CancelOrderAction
    → OrderCancelled
      → ReleaseStockOnCancellationListener
      → SendStatusChangeWhatsAppNotificationListener
      → CreateOrderAuditLogListener

Item Quantity Update:
  Merchant → UpdateOrderItemQuantityAction
    → OrderItemQuantityUpdated
      → RecalculateOrderTotalsListener
      → CreateOrderAuditLogListener
```

## 🔑 Business Rules

### Event Timing
- OrderCreated: after order saved and stock reserved
- OrderStatusChanged: after status updated in DB
- OrderCancelled: before actual cancellation (so listeners can act)
- Stock events: synchronously (must complete before proceeding)

### Listener Reliability
- Critical listeners (stock, audit): synchronous, transactional
- Notification listeners: queued, retryable
- All listeners: idempotent
- Stock release: never fails (retries infinitely if needed)

### Audit Trail
- Every order event creates audit log
- Audit logs immutable
- Captured data: user_id, ip_address, user_agent, timestamp
- Events never deleted (compliance)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Events dispatched at correct moments
- [ ] WhatsApp notifications sent
- [ ] Stock released on cancellation
- [ ] Audit logs created for all changes
- [ ] Totals recalculated on quantity update
- [ ] Listeners are idempotent
- [ ] Critical listeners never fail

### Técnicos
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Critical listeners synchronous
- [ ] Notification listeners queued (ShouldQueue)
- [ ] Event tests verify dispatching
- [ ] Listener tests verify execution
- [ ] Idempotency tested
- [ ] Retry logic tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## 🧪 Tests Requeridos

```php
describe('Order Events', function () {
    it('dispatches OrderCreated after order creation');
    it('dispatches OrderStatusChanged on status change');
    it('dispatches OrderCancelled on cancellation');
    it('dispatches StockReservedForOrder on stock reservation');
    it('dispatches StockReleasedForOrder on stock release');
});

describe('Order Listeners', function () {
    it('sends WhatsApp notification on order creation');
    it('releases stock on cancellation');
    it('creates audit log on every change');
    it('recalculates totals on quantity update');
    it('handles queue failures gracefully');
    it('is idempotent (can retry safely)');
});
```

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, 003
