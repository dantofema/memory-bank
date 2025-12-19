---
task_id: "cart-003-persistence"
module: "Cart"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Cart - Modelos, Repositorios y Persistencia"
priority: "HIGH"
estimated_time: "8 hours"
dependencies:
  - "cart-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/cart/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 003: Cart - Modelos, Repositorios y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Cart: modelos Eloquent con Casts para VOs, repositorios, migraciones y factories.

## 📦 Artefactos a Crear

### Modelos Eloquent
1. **Cart** - Carrito de compras session-based
   - Columns: id (uuid), session_id (unique), subtotal_cents, discount_cents, total_cents, timestamps
   - Relationships: hasMany(CartItem)
   - Casts: CartId, Money VOs

2. **CartItem** - Item individual en el carrito
   - Columns: id (uuid), cart_id (fk), product_id (fk), variant_id (fk nullable), name, price_cents, quantity, subtotal_cents, applied_promotion_id (fk nullable), discount_cents, total_cents, is_available, timestamps
   - Relationships: belongsTo(Cart), belongsTo(Product), belongsTo(ProductVariant), belongsTo(Promotion)
   - Casts: CartItemId, Money, Quantity VOs

### Repositorios
1. **CartRepository** - CRUD para Cart
   - Methods: findBySessionId, findById, save, delete, deleteExpired

2. **CartItemRepository** - CRUD para CartItem
   - Methods: findByCartId, save, delete, deleteByCartId

### Migraciones
1. **create_carts_table**
   - UUID primary key
   - session_id unique index
   - Cents columns (integer)
   - Timestamps
   - Index on updated_at for expiration cleanup

2. **create_cart_items_table**
   - UUID primary key
   - Foreign keys with cascade delete
   - Indexes on cart_id, product_id, variant_id
   - Cents columns

### Factories
1. **CartFactory** - Generate test carts
   - States: empty, with_items, with_promotions

2. **CartItemFactory** - Generate test items
   - States: with_product, with_variant, with_discount

## 🔑 Database Rules

### Indexes Required
- carts.session_id (unique)
- carts.updated_at (for expiration cleanup)
- cart_items.cart_id
- cart_items.product_id
- cart_items.variant_id

### Foreign Keys
- cart_items.cart_id → carts.id (cascade delete)
- cart_items.product_id → products.id
- cart_items.variant_id → product_variants.id (nullable)
- cart_items.applied_promotion_id → promotions.id (nullable)

### Cascade Deletes
- Deleting Cart deletes all CartItems
- Cart expiration cleanup deletes Cart + Items

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Models usan Casts para VOs
- [ ] Repositories implementan contratos
- [ ] Cascade deletes funcionan
- [ ] Factories generan datos válidos
- [ ] Indexes mejoran performance

### Técnicos
- [ ] Todos los models son `final`
- [ ] Repositories son `final`
- [ ] Factories son `final`
- [ ] Tests de integración con DB
- [ ] PHPStan level 6+ sin errores

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001
