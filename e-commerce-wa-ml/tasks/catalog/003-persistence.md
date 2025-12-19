---
task_id: "catalog-003-persistence"
module: "Catalog"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Catalog - Modelos, Repositorios y Persistencia"
priority: "CRITICAL"
estimated_time: "10 hours"
dependencies:
  - "catalog-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/catalog/agents_prompt.md"
phase: "Fase 1 - Fundamentos"
---

# Task 003: Catalog - Modelos, Repositorios y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Catalog: modelos Eloquent con Casts para VOs, repositorios, migraciones y factories.

## 📦 Artefactos a Crear

### Modelos Eloquent (4)

1. **Product** - Producto principal
   - Columns: id, category_id (fk), name (unique), description, price_cents, stock, is_active, timestamps, soft_deletes
   - Relationships: belongsTo(Category), hasMany(ProductVariant), hasMany(Promotion)
   - Casts: ProductId, CategoryId, Money (price), Stock

2. **Category** - Categoría de productos
   - Columns: id, name (unique), description, is_active, timestamps
   - Relationships: hasMany(Product)
   - Casts: CategoryId

3. **ProductVariant** - Variante de producto
   - Columns: id, product_id (fk), name, price_cents, stock, is_active, timestamps
   - Relationships: belongsTo(Product)
   - Casts: VariantId, ProductId, Money (price), Stock

4. **Promotion** - Promoción de producto
   - Columns: id, product_id (fk), type (enum), discount_value, start_date, end_date, is_active, timestamps
   - Relationships: belongsTo(Product)
   - Casts: PromotionId, ProductId, PromotionType (enum), PromotionDiscount, DateRange

### Repositorios (4)

1. **ProductRepository** - CRUD productos
   - Methods: find, findActive, create, update, delete, search, filterByCategory

2. **CategoryRepository** - CRUD categorías
   - Methods: find, findActive, create, update, delete, hasProducts

3. **VariantRepository** - CRUD variantes
   - Methods: find, findByProduct, create, update, delete

4. **PromotionRepository** - CRUD promociones
   - Methods: find, findActiveByProduct, create, update, delete, findExpired

### Migraciones (4)

1. **create_categories_table**
   - Primary key, name unique, is_active boolean, timestamps

2. **create_products_table**
   - Primary key, category_id FK, name unique, price_cents integer, stock integer, is_active boolean, timestamps, soft_deletes
   - Indexes: category_id, name, is_active

3. **create_product_variants_table**
   - Primary key, product_id FK (cascade delete), name, price_cents, stock, is_active, timestamps
   - Unique: product_id + name
   - Indexes: product_id

4. **create_promotions_table**
   - Primary key, product_id FK (cascade delete), type enum, discount_value, dates, is_active, timestamps
   - Indexes: product_id, start_date, end_date

### Factories (4)

1. **CategoryFactory** - Generate test categories
   - States: active, inactive

2. **ProductFactory** - Generate test products
   - States: active, inactive, out_of_stock, with_variants, with_promotion

3. **VariantFactory** - Generate test variants
   - States: active, inactive, out_of_stock

4. **PromotionFactory** - Generate test promotions
   - States: active, expired, percentage, fixed_price

## 🔑 Database Rules

### Indexes Required
- categories.name (unique)
- products.category_id
- products.name (unique)
- products.is_active
- product_variants.product_id
- product_variants(product_id, name) (unique composite)
- promotions.product_id
- promotions.start_date, end_date

### Foreign Keys
- products.category_id → categories.id
- product_variants.product_id → products.id (cascade delete)
- promotions.product_id → products.id (cascade delete)

### Constraints
- products.price_cents > 0
- products.stock >= 0
- product_variants.price_cents > 0
- product_variants.stock >= 0
- promotions.discount_value > 0
- promotions.start_date < end_date

### Soft Deletes
- products table uses soft deletes
- Variants and promotions hard delete (cascaded)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Models use Casts for VOs
- [ ] Repositories implement contracts
- [ ] Cascade deletes work (product → variants/promotions)
- [ ] Unique constraints enforced
- [ ] Factories generate valid data
- [ ] Indexes improve query performance

### Técnicos
- [ ] All models are `final`
- [ ] All repositories are `final`
- [ ] Factories for all models with states
- [ ] Integration tests with database
- [ ] Migration rollback works
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001
