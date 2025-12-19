---
task_id: "catalog-001-contracts"
module: "Catalog"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Catalog - Contracts, Value Objects, Enums y Data Objects"
priority: "CRITICAL"
estimated_time: "10 hours"
dependencies: []
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/catalog/agents_prompt.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 1 - Fundamentos"
---

# Task 001: Catalog - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Catalog: Value Objects, Enums, Eloquent Casts y Data Transfer Objects. Este módulo es la base del sistema (no tiene dependencias).

## 📋 Contexto

El módulo Catalog es CORE y fundacional. Gestiona productos, categorías, variantes y promociones. Es prerequisito para Cart, Orders y Reports.

## 📦 Artefactos a Crear

### Value Objects (8)

1. **ProductId** - ID de producto (int-based, Wireable)
2. **CategoryId** - ID de categoría (int-based, Wireable)
3. **VariantId** - ID de variante (int-based, Wireable)
4. **PromotionId** - ID de promoción (int-based, Wireable)
5. **Money** - Valor monetario en cents (shared with Cart/Orders)
6. **Stock** - Stock con lógica de disponibilidad (Wireable)
7. **Price** - Price wrapper sobre Money con comparación (Wireable)
8. **PromotionDiscount** - Descuento porcentual o fijo (Wireable)
9. **DateRange** - Rango de fechas (start/end) (Wireable)

### Enums (2)

1. **PromotionType** - PERCENTAGE_DISCOUNT | FIXED_PRICE
2. **StockLevel** - OUT_OF_STOCK | LOW_STOCK | IN_STOCK

### Casts (6)

1. **ProductIdCast** - int ↔ ProductId
2. **CategoryIdCast** - int ↔ CategoryId
3. **VariantIdCast** - int ↔ VariantId
4. **PromotionIdCast** - int ↔ PromotionId
5. **MoneyCast** - cents ↔ Money (shared)
6. **StockCast** - int ↔ Stock

### Data Objects (5)

1. **ProductData** - Product with category, price, stock, active
2. **CategoryData** - Category with name, description, active
3. **VariantData** - Variant with name, price, stock
4. **PromotionData** - Promotion with type, discount, dates
5. **ProductListData** - Product summary for listings

## 🔑 Business Rules

### Product Rules
- Product must have unique name per merchant
- Product must belong to exactly one category
- Product price must be > 0
- Product stock must be >= 0
- Inactive products hidden from public catalog

### Variant Rules
- Variant must belong to a product
- Variant must have unique name within product
- Variant price overrides product price
- Variant stock is independent

### Promotion Rules
- Two types: percentage discount or fixed price
- Has validity dates (start/end)
- Not cumulative (one per product)
- Best discount wins if multiple valid

### Stock Rules
- Stock levels: OUT_OF_STOCK (0), LOW_STOCK (< threshold), IN_STOCK
- Low stock threshold configurable (default: 10)
- Negative stock not allowed

## ✅ Criterios de Aceptación

### Funcionales
- [ ] All IDs are int-based VOs with Wireable
- [ ] Money uses cents (integer precision)
- [ ] Stock calculates availability levels
- [ ] PromotionDiscount handles percentage and fixed types
- [ ] DateRange validates start < end
- [ ] All VOs validate in constructor
- [ ] All Casts are bidirectional
- [ ] All Data Objects use Spatie Laravel Data

### Técnicos
- [ ] All classes are `final`
- [ ] VOs are `readonly`
- [ ] Strong typing everywhere
- [ ] Unit tests for all VOs (100% coverage)
- [ ] Tests for all Enums
- [ ] Tests for all Casts
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Bloqueante para:** Cart, Orders, Reports modules
