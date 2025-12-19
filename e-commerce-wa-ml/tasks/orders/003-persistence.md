---
task_id: "orders-003-persistence"
module: "Orders"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Orders - Modelos, Repositorios y Persistencia"
priority: "CRITICAL"
estimated_time: "12 hours"
dependencies:
  - "orders-001-contracts"
  - "catalog-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/orders/agents_prompt.md"
  - "@e-commerce-wa-ml/orders/domain_model.md"
phase: "Fase 2 - MVP Funcional"
---

# Task 003: Orders - Modelos, Repositorios y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Orders: modelos Eloquent con Casts para VOs, repositorios con soporte transaccional y locking, migraciones y factories.

## 📦 Artefactos a Crear

### Modelos Eloquent (4)

1. **Order** - Pedido principal (aggregate root)
   - Columns: id, order_number (unique), customer_name, customer_phone, customer_email, whatsapp_consent, order_status (enum), payment_status (enum), payment_method (enum), subtotal_cents, discount_cents, total_cents, notes, timestamps, soft_deletes
   - Relationships: hasMany(OrderItem), hasOne(Address), hasMany(OrderStatusLog)
   - Casts: OrderId, OrderNumber, CustomerData, PhoneNumber, OrderStatus, PaymentStatus, Money (totals)
   - Scopes: active(), byStatus(), byPhone()

2. **OrderItem** - Item de pedido (historical prices)
   - Columns: id, order_id (fk), product_id (fk), variant_id (fk nullable), name, price_cents, quantity, subtotal_cents, discount_cents, total_cents, applied_promotion_id (fk nullable), timestamps
   - Relationships: belongsTo(Order), belongsTo(Product), belongsTo(ProductVariant), belongsTo(Promotion)
   - Casts: OrderItemId, OrderId, Money (prices), Quantity
   - Note: Historical prices (never updated)

3. **Address** - Dirección de entrega
   - Columns: id, order_id (fk unique), street, number, apartment, city, state, postal_code, reference, timestamps
   - Relationships: belongsTo(Order)
   - Casts: AddressData (as single VO)

4. **OrderStatusLog** - Auditoría de cambios de estado
   - Columns: id, order_id (fk), user_id (fk nullable), field (order_status|payment_status), old_value, new_value, ip_address, user_agent, created_at
   - Relationships: belongsTo(Order), belongsTo(User)
   - Casts: none (keep as strings for audit)
   - Note: Immutable (no updates/deletes)

### Repositorios (4)

1. **OrderRepository** - CRUD con transacciones y locking
   - Methods: find, findByNumber, findByPhone, findActive, create, update, delete, withLock, countActiveByPhone
   - Transactions: createWithTransaction, updateWithTransaction
   - Locking: lockForUpdate (pessimistic)

2. **OrderItemRepository** - CRUD para items
   - Methods: findByOrder, create, update, delete, bulkCreate
   - Note: No independent access (always through Order)

3. **AddressRepository** - CRUD para direcciones
   - Methods: findByOrder, create, update
   - Note: 1:1 with Order

4. **OrderStatusLogRepository** - Solo escritura (append-only)
   - Methods: create, findByOrder, findByField
   - Note: Immutable audit trail

### Migraciones (4)

1. **create_orders_table**
   - Primary key, order_number unique index
   - Customer data columns (denormalized)
   - Status enums (stored as strings)
   - Money columns in cents (integer)
   - Timestamps, soft_deletes
   - Indexes: order_number, customer_phone, order_status, payment_status, created_at

2. **create_order_items_table**
   - Primary key, order_id FK (cascade delete)
   - Product/variant FKs (nullable variant_id)
   - Historical price columns in cents
   - Quantity integer
   - Indexes: order_id, product_id, variant_id

3. **create_addresses_table**
   - Primary key, order_id FK unique (1:1)
   - Address fields (street, number, city, state, etc.)
   - Index: order_id

4. **create_order_status_logs_table**
   - Primary key, order_id FK
   - Audit fields (field, old_value, new_value)
   - User tracking (user_id, ip_address, user_agent)
   - created_at only (no updates)
   - Indexes: order_id, field, created_at

### Factories (4)

1. **OrderFactory** - Generate test orders
   - States: new, confirmed, in_delivery, delivered, cancelled, with_items, with_payment
   - Methods: withItems($count), withStatus(OrderStatus), withPaymentStatus(PaymentStatus)

2. **OrderItemFactory** - Generate test items
   - States: with_product, with_variant, with_discount
   - Methods: forOrder(Order), withProduct(Product), withVariant(ProductVariant)

3. **AddressFactory** - Generate test addresses
   - States: complete, minimal
   - Methods: forOrder(Order)

4. **OrderStatusLogFactory** - Generate test logs
   - States: status_change, payment_change
   - Methods: forOrder(Order), byUser(User)

## 🔑 Database Rules

### Indexes Required
- orders.order_number (unique)
- orders.customer_phone
- orders.order_status
- orders.payment_status
- orders.created_at
- order_items.order_id
- order_items.product_id
- order_items.variant_id
- addresses.order_id (unique)
- order_status_logs.order_id
- order_status_logs.created_at

### Foreign Keys
- order_items.order_id → orders.id (cascade delete)
- order_items.product_id → products.id
- order_items.variant_id → product_variants.id (nullable)
- addresses.order_id → orders.id (cascade delete)
- order_status_logs.order_id → orders.id (cascade delete)
- order_status_logs.user_id → users.id (nullable)

### Constraints
- orders.order_number unique
- orders.subtotal_cents >= 0
- orders.total_cents > 0
- order_items.price_cents > 0
- order_items.quantity > 0
- addresses.order_id unique (1:1)

### Soft Deletes
- orders table uses soft deletes
- order_items hard delete (cascaded)
- addresses hard delete (cascaded)
- order_status_logs never delete (audit)

### Pessimistic Locking
- Orders table supports SELECT FOR UPDATE
- Used during stock reservation
- Lock timeout: 30 seconds
- Retry on deadlock (max 3 attempts)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Models use Casts for VOs
- [ ] Repositories support transactions
- [ ] Pessimistic locking works
- [ ] Cascade deletes work
- [ ] Unique constraints enforced
- [ ] Audit logs immutable
- [ ] Factories generate valid data
- [ ] Indexes improve performance

### Técnicos
- [ ] All models are `final`
- [ ] All repositories are `final`
- [ ] Transaction handling tested
- [ ] Locking tested (concurrent access)
- [ ] Deadlock retry tested
- [ ] Factories for all models with states
- [ ] Integration tests with database
- [ ] Migration rollback works
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 2 - MVP Funcional  
**Depende de:** Task 001, Catalog 003
