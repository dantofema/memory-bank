---
task_id: "orders-004-filament-tests"
module: "Orders"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Orders - Filament Resources y Tests Feature"
priority: "HIGH"
estimated_time: "14 hours"
dependencies:
  - "orders-001-contracts"
  - "orders-002-actions"
  - "orders-003-persistence"
  - "cart-004-livewire-tests"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/orders/agents_prompt.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 004: Orders - Filament Resources y Tests Feature

## 🎯 Objetivo

Implementar Filament resources para backoffice (merchant gestiona pedidos), sin UI pública (los pedidos se crean desde Cart checkout). Tests feature completos del flujo crítico.

## 📦 Artefactos a Crear

### Filament Resources (1)

1. **OrderResource** - Gestión completa de pedidos
   - **View Page:** Ver detalle completo del pedido
     - Order info, customer data, address
     - Items list with historical prices
     - Status badges (OrderStatus, PaymentStatus)
     - Timeline of status changes (audit log)
     - Actions: Change Status, Add Note, Cancel Order
   
   - **Edit Page:** Edición limitada
     - Can edit: address, phone, observations, item quantities (if stock available)
     - Cannot edit: customer name, add/remove products, prices
     - Validation: only if order editable (not DELIVERED/REFUNDED)
     - Real-time stock validation on quantity changes
   
   - **Table Page:** Listado de pedidos
     - Columns: order_number, customer_name, phone, total, order_status, payment_status, created_at
     - Filters: order_status, payment_status, date range, payment_method
     - Actions: view, edit, change status, cancel
     - Bulk actions: export to CSV
   
   - **Custom Actions:**
     - ChangeOrderStatusAction (modal with status selector)
     - ChangePaymentStatusAction (modal with status selector)
     - CancelOrderAction (confirmation modal)
     - AddNoteAction (modal with text input)
   
   - **Relation Managers:**
     - OrderItemsRelationManager (read-only, show historical prices)
     - OrderStatusLogsRelationManager (read-only, audit trail)

### Filament Widgets (3)

1. **OrderStatsWidget** - Estadísticas de pedidos
   - Total orders (today, week, month)
   - Orders by status (chart)
   - Revenue (today, week, month)
   - Average order value

2. **RecentOrdersWidget** - Pedidos recientes
   - Last 5 orders
   - Quick actions: view, change status

3. **PendingOrdersWidget** - Pedidos pendientes
   - NEW orders needing confirmation
   - Count + quick link to filtered list

### No Public UI
**Note:** Orders module has NO public frontend. Orders are created from Cart module's checkout process. This task focuses ONLY on backoffice management.

### Feature Tests (8)

**OrderCreationTest.php:**
- Create order from cart data
- Stock reserved transactionally
- Historical prices saved
- Order number generated correctly
- Customer data saved
- Address created
- Order created in NEW status
- Payment PENDING status
- Events dispatched

**OrderEditingTest.php:**
- Edit allowed fields (address, phone, quantities)
- Block editing DELIVERED orders
- Block editing REFUNDED orders
- Block adding/removing products
- Block changing historical prices
- Stock validated on quantity changes
- Totals recalculated correctly

**OrderStatusTransitionTest.php:**
- Valid transitions allowed
- Invalid transitions blocked
- Audit log created
- Stock released on CANCELLED/REJECTED
- Events dispatched
- Terminal states cannot transition

**PaymentStatusTest.php:**
- Payment status independent from order status
- Valid transitions allowed
- Invalid transitions blocked
- Audit log created
- Events dispatched

**StockManagementTest.php:**
- Stock reserved on order creation
- Stock released on cancellation
- Stock released on rejection
- Pessimistic locking prevents overselling
- Concurrent order creation handled
- Stock validation on quantity update

**AntiAbuseTest.php:**
- Active orders limit enforced
- Rate limiting by phone
- Phone normalization
- Captcha validation

**AuditTrailTest.php:**
- Status changes logged
- User tracked (if authenticated)
- IP address tracked
- Audit logs immutable
- Timeline display correct

**BackofficeTest.php:**
- Merchant can view orders
- Merchant can edit editable orders
- Merchant can change statuses
- Merchant can cancel orders
- Filters work correctly
- Bulk actions work

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Merchant can view all order details
- [ ] Merchant can edit allowed fields
- [ ] Status changes with validation
- [ ] Stock released on cancellation
- [ ] Audit trail complete and displayed
- [ ] Filters and search work
- [ ] Widgets display correct data
- [ ] No UI for anonymous users (orders created from Cart)

### Técnicos
- [ ] Filament resource fully functional
- [ ] Real-time validation on edits
- [ ] Status transition validation
- [ ] Feature tests cover all flows
- [ ] Concurrent access tested
- [ ] Stock reservation tested
- [ ] Audit trail tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, 002, 003, Cart 004
