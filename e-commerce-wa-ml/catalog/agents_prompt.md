---
title: "Catalog Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Catalog module following the 5-agent architecture"
module: "Catalog"
phase: "1 - Fundamentos"
module_type: "CORE"
dependencies: []
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
---

# Catalog Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Catalog module**, which is a **CORE module** in Phase 1 (Fundamentos) of the e-commerce platform. This module is the foundation of the entire system and must be completed first as other modules depend on it.

The Catalog module manages:

- **Products** with prices, stock, and active status
- **Categories** (one category per product)
- **Product Variants** (e.g., size, color) with independent pricing and stock
- **Promotions** (percentage discount or fixed price, with validity dates)
- Public catalog display (Livewire/Volt)
- Administrative management (Filament)

This module is **dependency-free** (no other modules required) and serves as the data foundation for Cart, Orders, and Reports modules.

---

## Architecture Overview

This system follows a **5-agent architecture** where each agent has specific responsibilities:

1. **Agent A**: Contracts, Data Objects, Value Objects, Enums
2. **Agent B**: Actions (Commands, Queries, Internal)
3. **Agent C**: Repositories, Models, Persistence, Migrations
4. **Agent D**: HTTP, Livewire/Volt, Filament, Tests Feature
5. **Agent E**: Events, Listeners, Jobs

You must implement the module following this structure, respecting Laravel 12 conventions and the project's strict quality standards.

---

## Project Context

**Project:** E-Commerce WhatsApp + Mercado Libre

**Architecture:**
- **Type:** Monolith with Laravel Modules
- **Frontend public:** Livewire v3 + Volt (anonymous users)
- **Backoffice:** Filament v4 (merchants only)
- **Backend:** Laravel 12 + PHP 8.5

**Quality Gates (NON-NEGOTIABLE):**
- ✅ PHPStan level 6+ (strict types everywhere)
- ✅ Pint (PSR-12 strict)
- ✅ Pest 4 (unit + feature tests, 100% coverage for critical paths)
- ✅ All classes `final`
- ✅ All Value Objects `final readonly` + `Wireable`
- ✅ No `protected` methods (composition over inheritance)
- ✅ No public arrays (use Value Objects or Data Objects)

**Key Constraints:**
- Single-tenant (one merchant per instance)
- One category per product (not multi-category)
- Promotions are not cumulative (best discount wins)
- Stock can be at product or variant level
- No authentication required for public catalog

---

## Domain Model Overview

### Key Entities

**Product:**
- Core entity with name, description, price, stock, active status
- Belongs to one Category
- May have multiple ProductVariants
- Can have applied Promotions
- Price and stock can be overridden by variant

**Category:**
- Simple classification entity
- One product belongs to one category
- Has name, description, active status
- Used for filtering and organization

**ProductVariant:**
- Variation of a product (e.g., size, color)
- Has independent price and stock
- Inherits product data but can override
- Must belong to a product

**Promotion:**
- Time-limited discount or special pricing
- Two types: percentage discount or fixed price
- Applies to specific products
- Has validity dates (start/end)
- Not cumulative (one per product)

### Key Value Objects

**ProductId** (int-based), **CategoryId** (int-based), **VariantId** (int-based), **PromotionId** (int-based), **Money** (cents-based, shared), **Stock** (int with availability logic), **Price** (Money with comparison), **Quantity** (validated int, shared), **PromotionDiscount** (percentage or fixed), **DateRange** (start/end dates)

### Key Services

**ProductService:**
- Product CRUD operations
- Stock management
- Availability checking

**PromotionService:**
- Promotion application logic
- Find applicable promotions
- Calculate discounts

**CategoryService:**
- Category CRUD operations
- Product listing by category

---

## Business Rules (CRITICAL)

### Product Management
1. Product must have unique name per merchant
2. Product must belong to exactly one category
3. Product price must be > 0
4. Product stock must be >= 0
5. Inactive products are hidden from public catalog
6. Inactive products cannot be added to cart
7. Product without variants uses its own price and stock
8. Product with variants uses variant price and stock

### Variant Management
9. Variant must belong to a product
10. Variant must have unique name within product
11. Variant price overrides product price
12. Variant stock is independent from product stock
13. Variant can be inactive (hidden)
14. Deleting product cascades to variants

### Category Management
15. Category must have unique name
16. Deleting category with products is not allowed (must reassign first)
17. Inactive categories hide their products from public catalog

### Stock Management
18. Stock is always an integer >= 0
19. Stock is checked but NOT decremented by Catalog module (Orders handles that)
20. Out of stock products (stock = 0) are visible but not purchasable
21. Low stock threshold configurable (default: 5)
22. Stock changes trigger events (ProductStockLowEvent, ProductOutOfStockEvent)

### Promotion Management
23. Promotion must have valid date range (start < end)
24. Promotion only applies if current date is within validity
25. Promotion types: PERCENTAGE (0-100%) or FIXED_PRICE (>0)
26. Only one promotion per product (best discount wins)
27. Promotions are not cumulative
28. Inactive promotions do not apply
29. Promotion cannot reduce price below 0

### Catalog Display
30. Public catalog shows only active products
31. Products from inactive categories are hidden
32. Products with stock = 0 show "Out of Stock" label
33. Prices displayed always include applicable promotion
34. Search/filter by name, category, price range
35. Results paginated (default: 20 per page)

---

## Module Structure

```
Modules/Catalog/
├── Contracts/                     # Agent A
│   ├── Commands/
│   │   ├── CreateProductInterface.php
│   │   ├── UpdateProductInterface.php
│   │   ├── DeleteProductInterface.php
│   │   ├── CreateCategoryInterface.php
│   │   └── CreatePromotionInterface.php
│   ├── Queries/
│   │   ├── GetProductInterface.php
│   │   ├── GetProductsByCategoryInterface.php
│   │   ├── CheckProductAvailabilityInterface.php
│   │   └── ApplyPromotionInterface.php
│   └── Data/
│       ├── ProductData.php
│       ├── CategoryData.php
│       ├── VariantData.php
│       ├── PromotionData.php
│       └── AvailabilityResult.php
├── ValueObjects/                  # Agent A
│   ├── Product/
│   │   ├── ProductId.php
│   │   ├── Price.php
│   │   └── Stock.php
│   ├── Category/
│   │   └── CategoryId.php
│   ├── Variant/
│   │   └── VariantId.php
│   └── Promotion/
│       ├── PromotionId.php
│       ├── PromotionDiscount.php
│       └── DateRange.php
├── Enums/                         # Agent A
│   └── PromotionType.php
├── Casts/                         # Agent A
│   ├── Product/
│   │   ├── ProductIdCast.php
│   │   ├── PriceCast.php
│   │   └── StockCast.php
│   ├── Category/
│   │   └── CategoryIdCast.php
│   └── Promotion/
│       ├── PromotionDiscountCast.php
│       └── DateRangeCast.php
├── Actions/                       # Agent B
│   ├── Commands/
│   │   ├── CreateProductAction.php
│   │   ├── UpdateProductAction.php
│   │   ├── DeleteProductAction.php
│   │   ├── CreateCategoryAction.php
│   │   ├── UpdateCategoryAction.php
│   │   ├── DeleteCategoryAction.php
│   │   ├── CreateProductVariantAction.php
│   │   ├── UpdateProductVariantAction.php
│   │   ├── CreatePromotionAction.php
│   │   └── UpdatePromotionAction.php
│   ├── Queries/
│   │   ├── GetProductAction.php
│   │   ├── GetProductsByCategoryAction.php
│   │   ├── GetActiveProductsAction.php
│   │   ├── SearchProductsAction.php
│   │   ├── CheckProductAvailabilityAction.php
│   │   └── GetApplicablePromotionsAction.php
│   └── Internal/
│       ├── ApplyPromotionToProductAction.php
│       ├── CalculateDiscountedPriceAction.php
│       └── ValidateStockAction.php
├── Exceptions/                    # Agent B
│   ├── ProductNotFoundException.php
│   ├── CategoryNotFoundException.php
│   ├── VariantNotFoundException.php
│   ├── PromotionNotFoundException.php
│   ├── DuplicateProductNameException.php
│   ├── InvalidPriceException.php
│   ├── InvalidStockException.php
│   ├── InvalidPromotionDateException.php
│   └── CategoryHasProductsException.php
├── Models/                        # Agent C
│   ├── Product.php
│   ├── Category.php
│   ├── ProductVariant.php
│   └── Promotion.php
├── Repositories/                  # Agent C
│   ├── ProductRepository.php
│   ├── CategoryRepository.php
│   ├── ProductVariantRepository.php
│   └── PromotionRepository.php
├── Database/                      # Agent C
│   ├── Factories/
│   │   ├── ProductFactory.php
│   │   ├── CategoryFactory.php
│   │   ├── ProductVariantFactory.php
│   │   └── PromotionFactory.php
│   ├── Migrations/
│   │   ├── xxxx_create_categories_table.php
│   │   ├── xxxx_create_products_table.php
│   │   ├── xxxx_create_product_variants_table.php
│   │   └── xxxx_create_promotions_table.php
│   └── Seeders/
│       ├── CategorySeeder.php
│       └── ProductSeeder.php
├── Livewire/                      # Agent D
│   ├── ProductList.php
│   ├── ProductDetail.php
│   ├── CategoryFilter.php
│   └── SearchProducts.php
├── Filament/                      # Agent D
│   └── Resources/
│       ├── ProductResource.php
│       ├── CategoryResource.php
│       └── PromotionResource.php
├── routes/                        # Agent D
│   └── web.php
├── Events/                        # Agent E
│   ├── ProductCreated.php
│   ├── ProductUpdated.php
│   ├── ProductDeleted.php
│   ├── ProductStockLowEvent.php
│   └── ProductOutOfStockEvent.php
├── Listeners/                     # Agent E
│   └── NotifyLowStockListener.php
└── Tests/
    ├── Unit/                      # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   └── Actions/
    └── Feature/                   # Agent D
        ├── ProductManagementTest.php
        ├── CategoryManagementTest.php
        ├── PromotionApplicationTest.php
        └── CatalogDisplayTest.php
```

---

## Interfaces Exposed (Communication with Other Modules)

This module exposes the following interfaces for other modules to consume:

### CheckProductAvailabilityInterface

```php
interface CheckProductAvailabilityInterface
{
    /**
     * Check if a product/variant has sufficient stock.
     * 
     * @throws ProductNotFoundException
     * @throws InvalidQuantityException
     */
    public function check(ProductId $productId, ?VariantId $variantId, Quantity $quantity): AvailabilityResult;
}
```

**Consumed by:** Cart, Orders

---

### ApplyPromotionInterface

```php
interface ApplyPromotionInterface
{
    /**
     * Apply applicable promotion to a product and return discounted price.
     * 
     * @throws ProductNotFoundException
     */
    public function apply(ProductId $productId, Money $basePrice): PriceResult;
}
```

**Consumed by:** Cart, Orders

---

### GetProductDetailsInterface

```php
interface GetProductDetailsInterface
{
    /**
     * Get complete product information including variants and promotions.
     * 
     * @throws ProductNotFoundException
     */
    public function get(ProductId $productId): ProductData;
}
```

**Consumed by:** Cart, Orders, Reports

---

## Events Emitted (Asynchronous Communication)

### ProductStockLowEvent

```php
final readonly class ProductStockLowEvent
{
    public function __construct(
        public ProductId $productId,
        public int $currentStock,
        public int $threshold
    ) {}
}
```

**Consumed by:** Reports (future: notifications to merchant)

---

### ProductOutOfStockEvent

```php
final readonly class ProductOutOfStockEvent
{
    public function __construct(
        public ProductId $productId
    ) {}
}
```

**Consumed by:** Reports (future: notifications to merchant)

---

### ProductCreated

```php
final readonly class ProductCreated
{
    public function __construct(
        public ProductId $productId,
        public string $name,
        public CategoryId $categoryId,
        public Carbon $createdAt
    ) {}
}
```

**Consumed by:** Reports (analytics)

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `ProductId` (int-based, Wireable, validation > 0)
   - `CategoryId` (int-based, Wireable, validation > 0)
   - `VariantId` (int-based, Wireable, validation > 0)
   - `PromotionId` (int-based, Wireable, validation > 0)
   - `Price` (wraps Money, adds comparison logic, Wireable)
   - `Stock` (int with availability logic, Wireable)
   - `PromotionDiscount` (value + type, Wireable)
   - `DateRange` (start + end validation, Wireable)

2. **Enums:**
   - `PromotionType` (PERCENTAGE, FIXED_PRICE, backed string enum)

3. **Casts:**
   - `ProductIdCast` (int ↔ ProductId)
   - `CategoryIdCast` (int ↔ CategoryId)
   - `VariantIdCast` (int ↔ VariantId)
   - `PromotionIdCast` (int ↔ PromotionId)
   - `PriceCast` (cents + currency ↔ Price)
   - `StockCast` (int ↔ Stock)
   - `PromotionDiscountCast` (value + type ↔ PromotionDiscount)
   - `DateRangeCast` (start_date + end_date ↔ DateRange)

4. **Data Objects (Spatie Laravel Data):**
   - `ProductData` (id, name, description, price, stock, category_id, variants, is_active)
   - `CategoryData` (id, name, description, is_active, products_count)
   - `VariantData` (id, product_id, name, price, stock, is_active)
   - `PromotionData` (id, product_id, type, discount, date_range, is_active)
   - `AvailabilityResult` (is_available, available_stock, requested_quantity)
   - `PriceResult` (original_price, discounted_price, discount_amount, promotion_id)

5. **Contracts (Commands):**
   - `CreateProductInterface`
   - `UpdateProductInterface`
   - `DeleteProductInterface`
   - `CreateCategoryInterface`
   - `UpdateCategoryInterface`
   - `DeleteCategoryInterface`
   - `CreateProductVariantInterface`
   - `UpdateProductVariantInterface`
   - `CreatePromotionInterface`
   - `UpdatePromotionInterface`

6. **Contracts (Queries):**
   - `GetProductInterface`
   - `GetProductsByCategoryInterface`
   - `GetActiveProductsInterface`
   - `SearchProductsInterface`
   - `CheckProductAvailabilityInterface`
   - `ApplyPromotionInterface`
   - `GetApplicablePromotionsInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] All VOs validate in constructor (throw exceptions)
- [ ] All Casts implement `CastsAttributes`
- [ ] All Data Objects use Spatie Laravel Data
- [ ] All IDs validate > 0
- [ ] Stock validates >= 0
- [ ] Price validates > 0
- [ ] DateRange validates start < end
- [ ] PromotionDiscount validates based on type (percentage: 0-100, fixed: > 0)
- [ ] Unit tests for all VOs (validation, behavior)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**

1. **Actions Commands:**
   - `CreateProductAction` (validate name unique, price > 0, stock >= 0, category exists)
   - `UpdateProductAction` (validate same as create)
   - `DeleteProductAction` (check no active orders reference it)
   - `CreateCategoryAction` (validate name unique)
   - `UpdateCategoryAction` (validate name unique if changed)
   - `DeleteCategoryAction` (prevent if has products)
   - `CreateProductVariantAction` (validate name unique within product, price > 0, stock >= 0)
   - `UpdateProductVariantAction` (validate same as create)
   - `CreatePromotionAction` (validate date range, discount based on type, product exists)
   - `UpdatePromotionAction` (validate same as create)

2. **Actions Queries:**
   - `GetProductAction` (get product by ID with variants and applicable promotion)
   - `GetProductsByCategoryAction` (paginated, filter active only, with promotions)
   - `GetActiveProductsAction` (paginated, filter is_active + category active)
   - `SearchProductsAction` (by name, category, price range, paginated)
   - `CheckProductAvailabilityAction` (check stock, return AvailabilityResult)
   - `GetApplicablePromotionsAction` (find valid promotions for product)

3. **Actions Internal:**
   - `ApplyPromotionToProductAction` (apply best promotion, calculate discounted price)
   - `CalculateDiscountedPriceAction` (based on promotion type)
   - `ValidateStockAction` (check availability without locking)

4. **Exceptions:**
   - `ProductNotFoundException` (404)
   - `CategoryNotFoundException` (404)
   - `VariantNotFoundException` (404)
   - `PromotionNotFoundException` (404)
   - `DuplicateProductNameException` (422)
   - `InvalidPriceException` (422)
   - `InvalidStockException` (422)
   - `InvalidPromotionDateException` (422)
   - `CategoryHasProductsException` (422)

**Unit Tests:**
- Mock repositories
- Test business logic in isolation
- Edge cases: duplicate names, invalid prices, date ranges, promotion calculations
- 100% coverage of critical paths

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method (`handle()` or `execute()`)
- [ ] All Actions use dependency injection
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Promotion calculation logic thoroughly tested (percentage and fixed)
- [ ] Best promotion selection logic tested
- [ ] Unit tests with mocks (no database)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories, Models, Persistence
**References:** `@laravel/agents/agent-c-persistence.md`

**Deliverables:**

1. **Models:**
   - `Product` (id, name, description, price_cents, price_currency, stock, is_active, category_id, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Category::class)`, `hasMany(ProductVariant::class)`, `hasMany(Promotion::class)`
     - Scopes: `active()`, `withVariants()`, `withPromotions()`
   - `Category` (id, name, description, is_active, timestamps)
     - Relationships: `hasMany(Product::class)`
     - Scopes: `active()`
   - `ProductVariant` (id, product_id, name, price_cents, price_currency, stock, is_active, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Product::class)`
     - Scopes: `active()`
   - `Promotion` (id, product_id, type, discount_value, start_date, end_date, is_active, timestamps)
     - Uses Casts for Value Objects
     - Relationships: `belongsTo(Product::class)`
     - Scopes: `active()`, `valid()`

2. **Repositories:**
   - `ProductRepository` (findById, findByName, findByCategory, findActive, save, delete)
   - `CategoryRepository` (findById, findByName, findActive, save, delete)
   - `ProductVariantRepository` (findById, findByProduct, save, delete)
   - `PromotionRepository` (findById, findByProduct, findValid, save, delete)

3. **Migrations:**
   - `create_categories_table` (id, name unique, description, is_active, timestamps, indexes)
   - `create_products_table` (id, name, description, price_cents, price_currency, stock, is_active, category_id foreign, timestamps, indexes)
   - `create_product_variants_table` (id, product_id foreign cascade, name, price_cents, price_currency, stock, is_active, timestamps, indexes)
   - `create_promotions_table` (id, product_id foreign cascade, type, discount_value, start_date, end_date, is_active, timestamps, indexes)

4. **Factories:**
   - `CategoryFactory` (active/inactive states)
   - `ProductFactory` (active/inactive, outOfStock, withVariants, withPromotion states)
   - `ProductVariantFactory` (active/inactive, outOfStock states)
   - `PromotionFactory` (percentage/fixed type, active/inactive, expired/valid states)

5. **Seeders:**
   - `CategorySeeder` (5-10 sample categories)
   - `ProductSeeder` (20-30 products across categories, some with variants, some with promotions)

**Integration Tests:**
- Database transactions
- Cascade deletes (product → variants, product → promotions)
- Unique constraints (product name, category name)
- Index performance
- Factory states

**Acceptance Criteria:**
- [ ] All models use Eloquent Casts for VOs
- [ ] All models are `final`
- [ ] All repositories implement contracts
- [ ] Factories for all models with relevant states
- [ ] Migrations with proper indexes and foreign keys
- [ ] Unique constraints enforced (product name, category name)
- [ ] Cascade deletes configured (product → variants/promotions)
- [ ] Prevent category delete if has products (database constraint)
- [ ] Integration tests with database
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Livewire, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **Livewire Components (Public Frontend):**
   - `ProductList` (product listing with filters, pagination, promotions display)
     - Public properties: `?CategoryId $categoryId`, `?string $search`, `?Money $minPrice`, `?Money $maxPrice`
     - Actions: `filterByCategory()`, `search()`, `filterByPrice()`
     - Events emitted: `product-list-updated`
   - `ProductDetail` (product detail view with variants selection, promotion display)
     - Public properties: `ProductId $productId`
     - Actions: `selectVariant(VariantId)`, `addToCart()`
     - Events emitted: `variant-selected`
   - `CategoryFilter` (category sidebar filter)
     - Public properties: `?CategoryId $selectedCategoryId`
     - Actions: `selectCategory(CategoryId)`
   - `SearchProducts` (search bar with autocomplete)
     - Public properties: `string $query`
     - Actions: `search()`

2. **Volt Pages:**
   - `products/index.blade.php` (main catalog page)
   - `products/show.blade.php` (product detail page)

3. **Routes (web.php):**
   - `GET /products` → ProductList
   - `GET /products/{id}` → ProductDetail
   - `GET /categories/{id}/products` → ProductList (filtered)

4. **Filament Resources (Backoffice):**
   - `ProductResource` (full CRUD, variants management inline, promotion assignment)
     - Form: name, description, price, stock, category, is_active
     - Table: name, category, price (with promotion), stock, is_active, actions
     - Filters: category, active status, stock level
     - Actions: bulk activate/deactivate, duplicate
     - Relation managers: variants, promotions
   - `CategoryResource` (full CRUD, product count display)
     - Form: name, description, is_active
     - Table: name, products count, is_active, actions
     - Filters: active status
   - `PromotionResource` (full CRUD, product selection)
     - Form: product, type, discount, date range, is_active
     - Table: product, type, discount, validity, is_active, actions
     - Filters: type, active status, validity

5. **Filament Widgets:**
   - `ProductStatsWidget` (total products, active, out of stock, low stock)
   - `CategoryStatsWidget` (total categories, products distribution)

**Feature Tests:**
- Public catalog display (active products only, promotions applied)
- Category filtering
- Search functionality
- Product detail view with variants
- Backoffice CRUD operations (products, categories, promotions)
- Variant management
- Promotion application
- Stock validation
- Smoke tests for all pages

**Acceptance Criteria:**
- [ ] All Livewire components use typed properties
- [ ] Real-time filtering and search
- [ ] Promotion prices displayed correctly
- [ ] Out of stock products labeled correctly
- [ ] Filament resources fully functional (CRUD + filters)
- [ ] Inline variant management in product form
- [ ] Feature tests cover all user journeys
- [ ] Smoke tests for all public and backoffice pages
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners, Jobs
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - `ProductCreated` (ProductId, name, CategoryId, Carbon)
   - `ProductUpdated` (ProductId, changed_fields, Carbon)
   - `ProductDeleted` (ProductId, Carbon)
   - `ProductStockLowEvent` (ProductId, current_stock, threshold)
   - `ProductOutOfStockEvent` (ProductId)

2. **Listeners:**
   - `NotifyLowStockListener` (listens to `ProductStockLowEvent`)
     - Future: send notification to merchant
     - MVP: log event for reporting

3. **Jobs:**
   - None for MVP (future: sync with external systems, ML recommendations)

**Event Tests:**
- Event dispatching on product creation/update/delete
- Stock events triggered correctly
- Listener execution
- Idempotency

**Acceptance Criteria:**
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Stock events trigger at correct thresholds
- [ ] Event tests verify dispatching and handling
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

## Quality Checklist (Before Completion)

### Code Quality
- [ ] PHPStan level 6+ passes (strict types everywhere)
- [ ] Pint executed (PSR-12 strict)
- [ ] All classes are `final`
- [ ] No `protected` methods
- [ ] All Value Objects are `final readonly` + `Wireable`
- [ ] No public arrays (use Value Objects or Data Objects)

### Testing
- [ ] Unit tests for all Value Objects (validation, behavior)
- [ ] Unit tests for all Actions (mocked dependencies)
- [ ] Feature tests for all user journeys (public catalog + backoffice)
- [ ] Integration tests for database operations
- [ ] Smoke tests for all pages
- [ ] 100% coverage for critical paths (promotion calculation, stock validation)

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex business rules have inline comments
- [ ] README.md in module root (overview, setup, usage)

### Database
- [ ] Migrations have proper indexes
- [ ] Foreign keys with cascade deletes where appropriate
- [ ] Unique constraints enforced
- [ ] Factories for all models with states

### Performance
- [ ] N+1 queries avoided (eager loading)
- [ ] Indexes on foreign keys and frequently queried columns
- [ ] Pagination implemented on listings

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Allow multiple categories per product (strict one-to-one)
- Allow cumulative promotions (only one per product)
- Modify stock directly from this module (Orders module handles stock decrement)
- Use float for prices (use integer cents)
- Skip validation in Value Objects
- Expose Eloquent models outside module (use Data Objects)
- Forget to check category active status when displaying products
- Allow negative stock or prices
- Create products without category
- Delete categories with products

✅ **DO:**
- Use Value Objects for all domain concepts
- One category per product (enforced by database)
- Apply best promotion if multiple are valid
- Use cents for prices (integer precision)
- Validate all inputs in constructors
- Use Data Objects for API responses
- Filter by category AND product active status
- Emit events for stock changes
- Cascade delete variants and promotions when product deleted
- Prevent category deletion if has products

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Merchant can create/edit/delete categories in Filament
   - Merchant can create/edit/delete products with pricing and stock
   - Merchant can create product variants with independent pricing/stock
   - Merchant can create promotions with date ranges
   - Anonymous users can browse active products
   - Users can filter by category
   - Users can search by name
   - Promotions automatically applied and displayed
   - Out of stock products labeled correctly
   - Low stock events emitted

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for critical paths
   - All acceptance criteria checked

3. **Performance:**
   - No N+1 queries
   - Proper indexes on all foreign keys
   - Catalog page loads in < 200ms
   - Product detail loads in < 100ms

4. **Integration:**
   - Interfaces exposed for Cart and Orders modules
   - Events emitted for Reports module
   - Seeders provide sample data for testing

---

## Configuration

### Environment Variables

```env
# Catalog Module
CATALOG_DEFAULT_CURRENCY=ARS
CATALOG_LOW_STOCK_THRESHOLD=5
CATALOG_PRODUCTS_PER_PAGE=20
```

### Config File: `config/catalog.php`

```php
return [
    'default_currency' => env('CATALOG_DEFAULT_CURRENCY', 'ARS'),
    'low_stock_threshold' => env('CATALOG_LOW_STOCK_THRESHOLD', 5),
    'products_per_page' => env('CATALOG_PRODUCTS_PER_PAGE', 20),
];
```

---

## Validation Commands

```bash
# Static analysis
./vendor/bin/sail composer run phpstan

# Code style
./vendor/bin/sail bin pint --dirty

# Refactoring
./vendor/bin/sail composer run rector

# Unit tests
./vendor/bin/sail test --filter=Unit

# Feature tests
./vendor/bin/sail test --filter=Feature

# All tests
./vendor/bin/sail test

# Specific module tests
./vendor/bin/sail test Modules/Catalog
```

---

## Dependencies and Integration

### Modules that Depend on Catalog

- **Cart**: Uses `CheckProductAvailabilityInterface`, `ApplyPromotionInterface`
- **Orders**: Uses `CheckProductAvailabilityInterface`, `GetProductDetailsInterface`
- **Reports**: Reads Product/Category data directly, listens to stock events

### External Dependencies

- None (this is a foundational module with no dependencies)

---

## Final Notes

- This module is **phase-critical** as it's the foundation for the entire system.
- Must be completed **first** (Phase 1) before Cart and Orders can be implemented.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.
- Focus on **simplicity**: one category per product, one promotion per product, clear stock rules.

**Expected Total Time:** 16-20 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
