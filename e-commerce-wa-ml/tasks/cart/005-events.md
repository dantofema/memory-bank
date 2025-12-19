---
task_id: "cart-005-events"
module: "Cart"
agent: "Agente E - Events, Listeners y Jobs"
title: "Cart - Eventos de Dominio y Listeners"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "cart-001-contracts"
  - "cart-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 005: Cart - Eventos de Dominio y Listeners

## 🎯 Objetivo

Implementar eventos de dominio para auditoría y coordinación, y listeners para efectos secundarios (limpiar cart después de checkout, liberar stock en fallo).

## 📦 Artefactos a Crear

### Domain Events

1. **CartCreated** - Cart was created for session
   - Payload: Cart

2. **ItemAddedToCart** - Item added to cart
   - Payload: Cart, CartItem

3. **ItemRemovedFromCart** - Item removed from cart
   - Payload: Cart, CartItemId

4. **ItemQuantityUpdated** - Quantity changed
   - Payload: Cart, CartItem, old_quantity, new_quantity

5. **CheckoutStarted** - Checkout process initiated
   - Payload: Cart, CheckoutData

6. **CheckoutCompleted** - Order created successfully
   - Payload: Order, Cart

7. **CheckoutFailed** - Checkout failed validation/stock
   - Payload: Cart, CheckoutData, ValidationResult

### Event Listeners

1. **ClearCartAfterCheckout** - Listener for CheckoutCompleted
   - Clears cart items
   - Marks cart as completed

2. **ReleaseStockOnFailure** - Listener for CheckoutFailed
   - Releases reserved stock
   - Logs failure reason

### Jobs
None (WhatsApp notification handled by WhatsApp module)

## 🔑 Event Flow

```
Add Item → ItemAddedToCart
Remove Item → ItemRemovedFromCart
Update Qty → ItemQuantityUpdated

Checkout Start → CheckoutStarted
  ↓ Success → CheckoutCompleted → ClearCartAfterCheckout
  ↓ Failure → CheckoutFailed → ReleaseStockOnFailure
```

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Eventos dispatched en momentos correctos
- [ ] Listeners ejecutan acciones correctas
- [ ] Stock liberado en checkout fallido
- [ ] Cart limpiado después de checkout exitoso

### Técnicos
- [ ] Todos los events son `final readonly`
- [ ] Todos los listeners son `final`
- [ ] Listeners implementan ShouldQueue
- [ ] Tests verifican dispatch y handling
- [ ] Idempotency en listeners
- [ ] PHPStan level 6+ sin errores

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, 003
