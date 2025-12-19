---
task_id: "cart-004-livewire-tests"
module: "Cart"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Cart - Componentes Livewire y Tests Feature"
priority: "HIGH"
estimated_time: "12 hours"
dependencies:
  - "cart-001-contracts"
  - "cart-002-actions"
  - "cart-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 004: Cart - Componentes Livewire y Tests Feature

## 🎯 Objetivo

Implementar los componentes Livewire para el frontend público (carrito y checkout) y tests feature completos de todos los flujos de usuario.

## 📦 Artefactos a Crear

### Livewire Components

1. **CartComponent** - Vista principal del carrito
   - Properties: `Cart $cart`
   - Actions: `addToCart()`, `removeFromCart()`, `updateQuantity()`, `clearCart()`
   - Events: `cart-updated`, `item-added`, `item-removed`
   - Real-time totals update

2. **CheckoutComponent** - Formulario de checkout
   - Properties: `CheckoutData $checkout`
   - Actions: `submitCheckout()`
   - Validación en tiempo real
   - Rate limiting
   - Captcha integration

3. **AddToCartButton** - Botón rápido desde catálogo
   - Properties: `Product $product`, `?ProductVariant $variant`
   - Action: `addToCart()`
   - Loading states

### Volt Pages
1. **cart.blade.php** - Página del carrito
2. **checkout.blade.php** - Página de checkout

### Routes
```php
// routes/web.php
Route::get('/cart', CartComponent::class)->name('cart');
Route::get('/checkout', CheckoutComponent::class)->name('checkout');
```

### Feature Tests

**CartManagementTest.php:**
- Add product to cart
- Add variant to cart
- Update quantity (increase/decrease)
- Remove item from cart
- Clear entire cart
- Cart persists across page refreshes
- Max items limit enforced

**CheckoutFlowTest.php:**
- Complete checkout flow (cart → checkout → order)
- Form validation (required fields)
- Phone normalization
- WhatsApp consent required
- Rate limiting enforced
- Captcha validation
- Max active orders per phone

**StockValidationTest.php:**
- Insufficient stock at add to cart
- Insufficient stock at checkout
- Concurrent purchases (pessimistic locking)
- Stock released on checkout failure

**PromotionApplicationTest.php:**
- Promotions applied automatically
- Best discount wins
- Expired promotions not applied

**Smoke Tests:**
- All cart pages load
- No N+1 queries

## ✅ Criterios de Aceptación

### Funcionales
- [ ] User can add items to cart
- [ ] Cart updates in real-time
- [ ] Checkout validates all fields
- [ ] Stock validated at every step
- [ ] Rate limiting blocks spam
- [ ] Order created successfully

### Técnicos
- [ ] Livewire components typed properties
- [ ] Real-time validation
- [ ] Feature tests cover all flows
- [ ] Smoke tests all pages
- [ ] PHPStan level 6+ sin errores

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, 002, 003
