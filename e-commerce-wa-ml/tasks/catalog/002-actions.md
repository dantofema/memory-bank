---
task_id: "catalog-002-actions"
module: "Catalog"
agent: "Agente B - Actions y Tests Unitarios"
title: "Catalog - Actions de Lógica de Negocio"
priority: "CRITICAL"
estimated_time: "12 hours"
dependencies:
  - "catalog-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/catalog/agents_prompt.md"
phase: "Fase 1 - Fundamentos"
---

# Task 002: Catalog - Actions de Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Catalog: CRUD de productos/categorías/promociones, cálculo de promociones, validación de stock y disponibilidad.

## 📦 Artefactos a Crear

### Actions Commands (9)
1. **CreateProductAction** - Crear producto con categoría
2. **UpdateProductAction** - Actualizar producto
3. **DeleteProductAction** - Eliminar producto (cascade variants/promotions)
4. **CreateCategoryAction** - Crear categoría
5. **UpdateCategoryAction** - Actualizar categoría
6. **DeleteCategoryAction** - Eliminar categoría (validar sin productos)
7. **CreateVariantAction** - Crear variante de producto
8. **UpdateVariantAction** - Actualizar variante
9. **DeleteVariantAction** - Eliminar variante

### Actions Queries (8)
10. **GetProductAction** - Obtener producto por ID
11. **ListProductsAction** - Listar productos con filtros
12. **GetCategoryAction** - Obtener categoría por ID
13. **ListCategoriesAction** - Listar categorías activas
14. **SearchProductsAction** - Buscar por nombre/descripción
15. **GetProductVariantsAction** - Variantes de un producto
16. **CheckStockAvailabilityAction** - Validar stock disponible
17. **CalculateEffectivePriceAction** - Precio con promoción aplicada

### Actions Internal (3)
18. **ApplyBestPromotionAction** - Aplicar mejor promoción disponible
19. **ValidateStockLevelAction** - Validar nivel de stock
20. **EmitStockEventsAction** - Emitir eventos de stock bajo/agotado

### Excepciones de Dominio (10)
- ProductNotFoundException (404)
- CategoryNotFoundException (404)
- VariantNotFoundException (404)
- PromotionNotFoundException (404)
- DuplicateProductNameException (422)
- CategoryHasProductsException (422)
- InvalidPriceException (422)
- InvalidStockException (422)
- InactiveProductException (422)
- PromotionExpiredException (422)

## 🔑 Key Business Rules

### Product Management
- Unique name validation per merchant
- Must belong to one category
- Price > 0, Stock >= 0
- Cascade delete variants and promotions

### Variant Management
- Unique name within product
- Independent price and stock
- Overrides product price when selected

### Promotion Application
- Only one promotion per product
- Best discount wins (highest savings)
- Validity date validation
- Two types: percentage or fixed price

### Stock Validation
- Check availability before cart add
- Emit low stock event (< threshold)
- Emit out of stock event (= 0)
- Threshold configurable (default: 10)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] CRUD Actions for all entities
- [ ] Promotion calculation correct (percentage vs fixed)
- [ ] Stock validation prevents overselling
- [ ] Category delete blocked if has products
- [ ] Product delete cascades to variants
- [ ] Best promotion automatically selected
- [ ] Stock events emitted at thresholds

### Técnicos
- [ ] All Actions are `final`
- [ ] One public method per Action
- [ ] Dependency injection
- [ ] Unit tests with mocks
- [ ] 100% coverage critical paths
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001
