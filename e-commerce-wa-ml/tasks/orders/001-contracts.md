---
task_id: "orders-001-contracts"
module: "Orders"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Orders - Contracts, Value Objects, Enums y Data Objects"
priority: "CRITICAL"
estimated_time: "12 hours"
dependencies:
  - "catalog-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/orders/agents_prompt.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 001: Orders - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Orders: Value Objects, Enums, Eloquent Casts y Data Transfer Objects. Este es el módulo más crítico del sistema que orquesta la creación de pedidos con validación transaccional de stock.

## 📋 Contexto

El módulo Orders coordina Catalog (stock), Payments (métodos de pago), WhatsApp (notificaciones) y Security (anti-abuso). Es el corazón del flujo de compra.

## 📦 Artefactos a Crear

### Value Objects (10)

1. **OrderId** - ID de pedido (int-based, Wireable)
2. **OrderNumber** - Número único de pedido (formato: ORD-YYYYMMDD-XXXXX, Wireable)
3. **OrderItemId** - ID de item de pedido (int-based, Wireable)
4. **CustomerData** - Datos del cliente (name, phone, email, consent, Wireable)
5. **AddressData** - Dirección de entrega (street, number, city, state, etc., Wireable)
6. **PhoneNumber** - Teléfono normalizado (shared with Cart/Security, Wireable)
7. **Money** - Valor monetario en cents (shared, Wireable)
8. **Quantity** - Cantidad validada (shared, Wireable)
9. **OrderTotals** - Totales del pedido (subtotal, discount, total, Wireable)
10. **ValidationResult** - Resultado de validación (shared with Cart, Wireable)

### Enums (2)

1. **OrderStatus** - NEW | CONFIRMED | IN_DELIVERY | DELIVERED | REJECTED | CANCELLED
   - Methods: canTransitionTo(), isTerminal(), isEditable(), label(), color()
   
2. **PaymentStatus** - PENDING | PAID | REFUNDED
   - Methods: canTransitionTo(), label(), color()

### Casts (7)

1. **OrderIdCast** - int ↔ OrderId
2. **OrderNumberCast** - string ↔ OrderNumber
3. **OrderItemIdCast** - int ↔ OrderItemId
4. **CustomerDataCast** - json ↔ CustomerData
5. **AddressDataCast** - json ↔ AddressData
6. **PhoneNumberCast** - string ↔ PhoneNumber (shared)
7. **MoneyCast** - cents ↔ Money (shared)

### Data Objects (6)

1. **OrderData** - Complete order with items, customer, address, totals
2. **OrderItemData** - Order line item with product, price, quantity, totals
3. **CreateOrderData** - Input for order creation from cart
4. **UpdateOrderData** - Input for limited order editing
5. **OrderSummaryData** - Order summary for listings
6. **OrderStatusChangeData** - Data for status transitions with audit

## 🔑 Business Rules

### Order Rules
- Order number unique: ORD-{date}-{sequence}
- Two independent states: OrderStatus, PaymentStatus
- Historical prices never change
- Limited editing based on status
- Cannot edit DELIVERED or REFUNDED orders
- Full audit trail for status changes

### Status Transition Rules
- OrderStatus state machine with valid transitions
- PaymentStatus independent from OrderStatus
- Terminal states: DELIVERED, REJECTED, CANCELLED (OrderStatus)
- Terminal state: REFUNDED (PaymentStatus)

### Editing Rules
- Can edit: address, phone, observations, item quantities, payment method
- Cannot edit: customer name, add/remove products, prices
- Cannot edit if status is DELIVERED or REFUNDED
- Quantity changes require stock validation

### Anti-Abuse Rules
- Max N active orders per phone (default: 2, configurable 2-5)
- Active orders: NEW, CONFIRMED, IN_DELIVERY
- Rate limiting: 3 orders/hour per phone
- Phone number normalized before validation

## ✅ Criterios de Aceptación

### Funcionales
- [ ] OrderNumber generates unique sequential numbers
- [ ] CustomerData validates name, phone, consent
- [ ] AddressData validates required fields
- [ ] OrderStatus state machine enforced
- [ ] PaymentStatus independent transitions
- [ ] OrderTotals calculates correctly
- [ ] PhoneNumber normalized consistently
- [ ] All VOs validate in constructor
- [ ] All Casts bidirectional

### Técnicos
- [ ] All classes are `final`
- [ ] VOs are `readonly`
- [ ] Strong typing everywhere
- [ ] Unit tests for all VOs (100%)
- [ ] Tests for all Enums (state machine)
- [ ] Tests for all Casts
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Bloqueante para:** Cart checkout, WhatsApp notifications
