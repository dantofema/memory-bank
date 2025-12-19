---
task_id: "cart-002-actions"
module: "Cart"
agent: "Agente B - Actions y Tests Unitarios"
title: "Cart - Actions de Lógica de Negocio"
priority: "HIGH"
estimated_time: "10 hours"
dependencies:
  - "cart-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 002: Cart - Actions y Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Cart que encapsulan toda la lógica de negocio: gestión de items, validación de stock, aplicación de promociones y proceso de checkout.

## 📦 Artefactos a Crear

### Actions Commands
1. **AddItemToCartAction** - Agregar producto/variante al carrito
2. **RemoveItemFromCartAction** - Remover item del carrito
3. **UpdateCartItemQuantityAction** - Actualizar cantidad de item
4. **ClearCartAction** - Limpiar carrito completo
5. **ProcessCheckoutAction** - Procesar checkout completo

### Actions Queries
6. **GetCartAction** - Obtener o crear carrito por session
7. **CalculateCartTotalsAction** - Calcular subtotal/discount/total
8. **ValidateCartStockAction** - Validar stock de todos los items

### Actions Internal
9. **ApplyPromotionsAction** - Aplicar promociones a items
10. **ReserveStockAction** - Reservar stock con pessimistic lock
11. **ReleaseStockAction** - Liberar stock reservado

### Excepciones de Dominio
- CartNotFoundException (404)
- CartItemNotFoundException (404)
- InsufficientStockException (422)
- MaxCartItemsExceededException (422)
- InvalidQuantityException (422)
- InvalidCheckoutDataException (422)
- StockReservationFailedException (409)
- CartExpiredException (410)

## 🔑 Business Rules

### Cart Management
- Max 50 items per cart (configurable)
- Max 9999 quantity per item
- Cart expires after 7 days inactivity
- Totals recalculated on every change

### Stock Validation
- Check at: add, update, checkout, order creation
- Pessimistic locking during checkout
- Release stock on any failure

### Promotions
- Only one promotion per item
- Best discount wins
- Applied automatically on total calculation

## ✅ Criterios de Aceptación

### Funcionales
- [ ] AddItem valida stock antes de agregar
- [ ] UpdateQuantity valida stock disponible
- [ ] ProcessCheckout reserva stock atomically
- [ ] ApplyPromotions encuentra mejor descuento
- [ ] ReleaseStock libera en caso de fallo
- [ ] Excepciones con mensajes descriptivos

### Técnicos
- [ ] Todas las Actions son `final`
- [ ] Un solo método público por Action
- [ ] Dependency injection correcta
- [ ] Tests unitarios con mocks
- [ ] Cobertura 100% lógica crítica
- [ ] PHPStan level 6+ sin errores

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001
