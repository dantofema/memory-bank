---
task_id: "orders-002-actions"
module: "Orders"
agent: "Agente B - Actions y Tests Unitarios"
title: "Orders - Actions de Lógica de Negocio"
priority: "CRITICAL"
estimated_time: "16 hours"
dependencies:
  - "orders-001-contracts"
  - "catalog-002-actions"
  - "catalog-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/orders/agents_prompt.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 002: Orders - Actions de Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Orders: creación transaccional con stock, gestión de estados, edición limitada, auditoría y liberación de stock. Este es el núcleo más complejo del sistema.

## 📦 Artefactos a Crear

### Actions Commands (8)

1. **CreateOrderAction** - Crear pedido desde carrito
   - Validate stock transactionally (pessimistic lock)
   - Create order with historical prices
   - Create order items
   - Create address
   - Reserve stock atomically
   - Emit OrderCreated event
   
2. **UpdateOrderAction** - Actualizar pedido limitadamente
   - Validate editable status
   - Validate allowed field changes
   - Validate stock for quantity changes
   - Update order
   - Emit OrderUpdated event

3. **ChangeOrderStatusAction** - Cambiar OrderStatus
   - Validate state transition
   - Update status
   - Create audit log entry
   - Release stock if CANCELLED/REJECTED
   - Emit OrderStatusChanged event

4. **ChangePaymentStatusAction** - Cambiar PaymentStatus
   - Validate state transition (independent)
   - Update status
   - Create audit log entry
   - Emit PaymentStatusChanged event

5. **CancelOrderAction** - Cancelar pedido
   - Validate cancellable status
   - Change status to CANCELLED
   - Release stock transactionally
   - Create audit log
   - Emit OrderCancelled event

6. **UpdateOrderItemQuantityAction** - Actualizar cantidad de item
   - Validate order editable
   - Validate stock available
   - Update quantity
   - Recalculate totals
   - Emit ItemQuantityUpdated event

7. **AddNoteToOrderAction** - Agregar nota interna
   - Add note with timestamp and user
   - Emit NoteAdded event

8. **ReleaseOrderStockAction** - Liberar stock reservado
   - Release stock for all order items
   - Transactional operation
   - Idempotent (can retry)

### Actions Queries (6)

9. **GetOrderAction** - Obtener pedido por ID/número
10. **ListOrdersAction** - Listar pedidos con filtros
11. **GetOrdersByPhoneAction** - Pedidos por teléfono
12. **CountActiveOrdersByPhoneAction** - Contar pedidos activos
13. **ValidateOrderEditableAction** - Validar si es editable
14. **CalculateOrderTotalsAction** - Recalcular totales

### Actions Internal (5)

15. **ValidateStockForOrderAction** - Validar stock disponible
16. **ReserveStockForOrderAction** - Reservar stock con lock
17. **ValidateOrderStatusTransitionAction** - Validar transición válida
18. **ValidateActiveOrdersLimitAction** - Validar límite por teléfono
19. **CreateOrderAuditLogAction** - Crear entrada de auditoría

### Excepciones de Dominio (12)

- OrderNotFoundException (404)
- OrderItemNotFoundException (404)
- OrderNotEditableException (422)
- InvalidOrderStatusTransitionException (422)
- InvalidPaymentStatusTransitionException (422)
- InsufficientStockException (422)
- StockReservationFailedException (409)
- ActiveOrdersLimitExceededException (422)
- CannotModifyHistoricalPriceException (422)
- CannotAddRemoveProductsException (422)
- OrderAlreadyCancelledException (422)
- InvalidOrderDataException (422)

## 🔑 Key Business Rules

### Order Creation (Rules 1-10)
1. Stock validated transactionally (pessimistic lock: SELECT FOR UPDATE)
2. Stock reserved atomically with order creation
3. Historical prices saved (never change)
4. Order number generated: ORD-YYYYMMDD-{seq}
5. Max N active orders per phone enforced
6. Rate limiting: 3 orders/hour per phone
7. All operations in database transaction
8. Stock lock timeout: 30 seconds
9. Stock released on any failure
10. Order created in NEW status, payment PENDING

### Order Editing (Rules 11-18)
11. Cannot edit if status DELIVERED or REFUNDED
12. Can edit: address, phone, observations, item quantities, payment method
13. Cannot edit: customer name, add/remove products, prices
14. Quantity changes require stock validation
15. Totals recalculated automatically
16. Audit log created for every edit
17. Status changes require valid transitions
18. Stock released if order CANCELLED or REJECTED

### Status Transitions (Rules 19-26)
19. OrderStatus state machine enforced
20. PaymentStatus independent from OrderStatus
21. Valid transitions defined in state machine
22. Cannot transition from terminal states
23. Audit log for every status change
24. Stock released on CANCELLED/REJECTED
25. Events emitted on every transition
26. Terminal states: DELIVERED, REJECTED, CANCELLED, REFUNDED

### Anti-Abuse (Rules 27-31)
27. Active orders = NEW | CONFIRMED | IN_DELIVERY
28. Max active orders per phone configurable (default: 2)
29. Phone normalized before validation
30. Rate limiting by phone number
31. Captcha validation on order creation

## ✅ Criterios de Aceptación

### Funcionales
- [ ] CreateOrder validates and reserves stock atomically
- [ ] UpdateOrder enforces editing rules
- [ ] Status transitions follow state machine
- [ ] Stock released on cancellation
- [ ] Active orders limit enforced
- [ ] Audit logs created for all changes
- [ ] Totals recalculated correctly
- [ ] Pessimistic locking prevents overselling

### Técnicos
- [ ] All Actions are `final`
- [ ] One public method per Action
- [ ] Dependency injection
- [ ] Unit tests with mocks
- [ ] 100% coverage critical paths (stock, transitions)
- [ ] Transaction handling tested
- [ ] Concurrent access tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, Catalog 002-003
