---
task_id: "catalog-004-livewire-filament"
module: "Catalog"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Catalog - Livewire, Filament y Tests Feature"
priority: "HIGH"
estimated_time: "14 hours"
dependencies:
  - "catalog-001-contracts"
  - "catalog-002-actions"
  - "catalog-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/catalog/agents_prompt.md"
phase: "Fase 1 - Fundamentos"
---

# Task 004: Catalog - Livewire, Filament y Tests Feature

## 🎯 Objetivo

Implementar las interfaces de usuario para frontend público (Livewire/Volt) y backoffice (Filament), con tests feature completos.

## 📦 Artefactos a Crear

### Livewire Components (3)

1. **ProductListComponent** - Catálogo público con filtros
   - Properties: `$products`, `$selectedCategory`, `$searchTerm`
   - Actions: `filter()`, `search()`, `clearFilters()`
   - Real-time filtering and search
   - Pagination (20 items per page)

2. **ProductDetailComponent** - Detalle de producto
   - Properties: `Product $product`, `$selectedVariant`
   - Actions: `selectVariant()`
   - Show promotion badge if applicable
   - Display stock availability

3. **CategoryFilterComponent** - Filtro por categoría
   - Properties: `$categories`, `$selectedCategoryId`
   - Actions: `selectCategory()`
   - Emit event when category selected

### Volt Pages (2)

1. **products/index.blade.php** - Página principal del catálogo
2. **products/show.blade.php** - Página de detalle de producto

### Routes (3)

```php
Route::get('/products', ProductListComponent::class)->name('products.index');
Route::get('/products/{id}', ProductDetailComponent::class)->name('products.show');
Route::get('/categories/{id}/products', ProductListComponent::class)->name('categories.products');
```

### Filament Resources (3)

1. **ProductResource** - Gestión de productos
   - Form: name, category, description, price, stock, is_active
   - Relation Manager: Variants (inline CRUD)
   - Relation Manager: Promotions (attach/detach)
   - Table: name, category, price, stock, status, actions
   - Filters: category, active status, stock level
   - Actions: bulk activate/deactivate, duplicate
   - Widgets: Products stats (total, active, low stock, out of stock)

2. **CategoryResource** - Gestión de categorías
   - Form: name, description, is_active
   - Table: name, product count, is_active, actions
   - Filters: active status
   - Validation: prevent delete if has products

3. **PromotionResource** - Gestión de promociones
   - Form: product, type, discount_value, date_range, is_active
   - Table: product, type, discount, validity dates, status, actions
   - Filters: type, active status, validity (active/expired/future)
   - Actions: bulk activate/deactivate

### Filament Widgets (2)

1. **ProductStatsWidget** - Estadísticas de productos
   - Total products
   - Active products
   - Out of stock
   - Low stock (< threshold)

2. **CategoryStatsWidget** - Estadísticas de categorías
   - Total categories
   - Products per category chart

### Feature Tests (6)

**PublicCatalogTest.php:**
- Display active products only
- Filter by category
- Search by name
- Show product detail
- Display promotions correctly
- Show stock availability

**BackofficeCRUDTest.php:**
- Create/update/delete product
- Create/update/delete category
- Create/update/delete variant
- Create/update/delete promotion
- Prevent category delete with products
- Cascade delete variants/promotions

**PromotionApplicationTest.php:**
- Percentage discount calculation
- Fixed price application
- Best promotion wins
- Expired promotions not shown
- Promotion validity dates

**StockDisplayTest.php:**
- Out of stock label
- Low stock warning
- In stock display
- Variant stock override

**SearchAndFilterTest.php:**
- Search by product name
- Filter by category
- Combined filters
- Pagination works

**SmokeTest.php:**
- All public pages load
- All Filament pages load
- No N+1 queries

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Public catalog shows active products only
- [ ] Category filtering works
- [ ] Search functionality works
- [ ] Product detail shows variants
- [ ] Promotions displayed correctly
- [ ] Stock availability labeled correctly
- [ ] Filament CRUD fully functional
- [ ] Inline variant management works
- [ ] Promotion assignment works
- [ ] Stats widgets display correctly

### Técnicos
- [ ] Livewire components use typed properties
- [ ] Real-time filtering and search
- [ ] Pagination implemented
- [ ] Eager loading prevents N+1
- [ ] Feature tests cover all flows
- [ ] Smoke tests all pages
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001, 002, 003
