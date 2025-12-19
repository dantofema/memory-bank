---
title: "Reports Module - Agents Prompt"
version: "1.0"
author: "Alejandro Leone"
date: "2025-12-19"
purpose: "Professional prompt for AI agents to implement the Reports module following the 5-agent architecture"
module: "Reports"
phase: "4 - Post-MVP"
module_type: "STANDARD"
dependencies:
  - "Orders"
  - "Payments"
  - "Catalog"
  - "Auth"
references:
  - "@laravel/AGENTS_ARCHITECTURE.md"
  - "@laravel/conventions/value-objects.md"
  - "@e-commerce-wa-ml/project_definition.md"
  - "@e-commerce-wa-ml/modular-architecture.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
---

# Reports Module - Professional Agents Prompt

## Context

You are tasked with implementing the **Reports module**, a **STANDARD module** in Phase 4 (Post-MVP) of the e-commerce platform. This is a **read-only analytics module** that provides:

- **Sales reports** by date range with time aggregation (daily, weekly, monthly)
- **Product performance** tracking (top selling products by quantity and revenue)
- **Order status distribution** analysis
- **Payment status monitoring**
- **Revenue calculations** with payment filtering
- **Dashboard metrics** for quick business insights

This module **consumes data** from Orders, Payments, and Catalog modules without modifying them. All reports are **merchant-only** (authentication required) and heavily **cached** for performance.

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
- **Frontend public:** None (this module is backoffice only)
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
- **Read-only module** (no writes to Orders/Payments/Catalog)
- **Merchant authenticated only** (via Auth module)
- **Heavily cached** (5 minutes for reports, 1 minute for dashboard)
- **Maximum date range**: 2 years
- **Rate limited**: 60 requests/minute per merchant
- No customer PII exposed (aggregate data only)

---

## Domain Model Overview

### Key Services

**SalesReportService:**
- Generate comprehensive sales reports
- Aggregate by period (daily, weekly, monthly, custom)
- Calculate totals, AOV, product counts
- Apply status/payment filters

**ProductPerformanceService:**
- Analyze product-level performance
- Top selling products by revenue/quantity
- Product metrics (revenue, quantity, order count, avg price)

**OrderStatusReportService:**
- Order status distribution analysis
- Count and percentage by status
- Total value by status

**PaymentStatusReportService:**
- Payment status distribution analysis
- Count and percentage by payment status
- Total value by payment status

**RevenueReportService:**
- Detailed revenue reports
- Breakdown by payment status (paid, pending, refunded)
- Daily revenue time series

**DashboardMetricsService:**
- Quick metrics for dashboard
- Today's orders, week's orders, month's revenue
- Top products

### Key Value Objects

**ReportRequest** (startDate, endDate, aggregationPeriod, limit, statusFilter, paymentFilter), **Money** (shared, cents-based)

### Key Data Objects

**SalesReportData**, **ProductPerformanceData**, **OrderStatusDistributionData**, **PaymentStatusDistributionData**, **RevenueReportData**, **DashboardMetrics**, **PeriodSalesData**, **StatusCountData**, **PaymentCountData**, **DailyRevenueData**

### Key Enums

**AggregationPeriod** (DAILY, WEEKLY, MONTHLY, CUSTOM)
**OrderStatusFilter** (ALL, NEW, CONFIRMED, IN_DELIVERY, DELIVERED, REJECTED, CANCELLED)
**PaymentStatusFilter** (ALL, PENDING, PAID, REFUNDED)

---

## Business Rules (CRITICAL)

### Date Ranges (Rules 1-4)
1. Default date range: Last 30 days
2. Maximum date range: **2 years** (throw exception if exceeded)
3. Future dates **not allowed**
4. startDate must be <= endDate

### Aggregation (Rules 5-8)
5. DAILY: One data point per calendar day
6. WEEKLY: Monday to Sunday grouping
7. MONTHLY: Calendar month grouping
8. CUSTOM: No sub-aggregation, single total

### Revenue Calculation (Rules 9-12)
9. Order total = SUM(orderItems.quantity × orderItems.unit_price)
10. Paid revenue = orders where payment_status = 'paid'
11. Pending revenue = orders where payment_status = 'pending'
12. **Refunded orders excluded** from revenue metrics

### Product Performance (Rules 13-16)
13. Only products with at least **1 sale** in period
14. Sorted by **totalRevenue DESC** by default
15. Variant-level products aggregated to parent product
16. Archived products **included** if sold during period

### Order Status Filtering (Rules 17-19)
17. Filter applies **before** aggregation
18. Multiple statuses not supported (use ALL or specific)
19. Cancelled and rejected orders included in counts but flagged

### Caching Strategy (Rules 20-23)
20. Reports cached for **5 minutes**
21. Cache key includes all request parameters
22. Cache invalidated on new order creation (listen to OrderCreated event)
23. Dashboard metrics cached for **1 minute**

### Security & Privacy (Rules 24-26)
24. All endpoints protected by **Auth middleware** (merchant only)
25. Reports contain **aggregate data only** (no customer PII)
26. Rate limiting: **60 requests/minute** per merchant

---

## Module Structure

```
Modules/Reports/
├── Contracts/                             # Agent A
│   ├── Queries/
│   │   ├── GenerateSalesReportInterface.php
│   │   ├── GenerateProductPerformanceInterface.php
│   │   ├── GenerateOrderStatusReportInterface.php
│   │   ├── GeneratePaymentStatusReportInterface.php
│   │   ├── GenerateRevenueReportInterface.php
│   │   └── GetDashboardMetricsInterface.php
│   └── Data/
│       ├── SalesReportData.php
│       ├── ProductPerformanceData.php
│       ├── OrderStatusDistributionData.php
│       ├── PaymentStatusDistributionData.php
│       ├── RevenueReportData.php
│       ├── DashboardMetrics.php
│       ├── PeriodSalesData.php
│       ├── StatusCountData.php
│       ├── PaymentCountData.php
│       └── DailyRevenueData.php
├── ValueObjects/                          # Agent A
│   └── ReportRequest.php
├── Enums/                                 # Agent A
│   ├── AggregationPeriod.php
│   ├── OrderStatusFilter.php
│   └── PaymentStatusFilter.php
├── Casts/                                 # Agent A
│   └── (None - uses shared casts)
├── Actions/                               # Agent B
│   ├── Queries/
│   │   ├── GenerateSalesReportAction.php
│   │   ├── GenerateProductPerformanceAction.php
│   │   ├── GenerateOrderStatusReportAction.php
│   │   ├── GeneratePaymentStatusReportAction.php
│   │   ├── GenerateRevenueReportAction.php
│   │   └── GetDashboardMetricsAction.php
│   └── Internal/
│       ├── AggregateByPeriodAction.php
│       ├── CalculateTotalsAction.php
│       ├── CalculateAOVAction.php
│       ├── GroupByProductAction.php
│       ├── CalculatePercentagesAction.php
│       └── ValidateReportRequestAction.php
├── Exceptions/                            # Agent B
│   ├── InvalidReportRequestException.php
│   ├── DateRangeExceededException.php
│   └── FutureDateNotAllowedException.php
├── Services/                              # Agent B
│   ├── SalesReportService.php
│   ├── ProductPerformanceService.php
│   ├── OrderStatusReportService.php
│   ├── PaymentStatusReportService.php
│   ├── RevenueReportService.php
│   └── DashboardMetricsService.php
├── Repositories/                          # Agent C
│   ├── ReportOrderRepository.php
│   └── ReportOrderItemRepository.php
├── Http/                                  # Agent D
│   └── Controllers/
│       └── ReportController.php
├── Filament/                              # Agent D
│   ├── Pages/
│   │   ├── SalesReport.php
│   │   ├── ProductPerformance.php
│   │   ├── OrderStatusReport.php
│   │   ├── PaymentStatusReport.php
│   │   └── RevenueReport.php
│   └── Widgets/
│       ├── DashboardStats.php
│       ├── TopProducts.php
│       ├── RecentOrders.php
│       └── RevenueChart.php
├── routes/                                # Agent D
│   └── api.php
├── Listeners/                             # Agent E
│   ├── InvalidateSalesCacheOnOrderCreated.php
│   ├── InvalidateOrderStatusCacheOnStatusChange.php
│   └── InvalidatePaymentCacheOnPaymentChange.php
└── Tests/
    ├── Unit/                              # Agent A + Agent B
    │   ├── ValueObjects/
    │   ├── Enums/
    │   ├── Actions/
    │   └── Services/
    └── Feature/                           # Agent D
        ├── SalesReportTest.php
        ├── ProductPerformanceTest.php
        ├── OrderStatusReportTest.php
        ├── PaymentStatusReportTest.php
        ├── RevenueReportTest.php
        ├── DashboardMetricsTest.php
        └── CachingBehaviorTest.php
```

---

## Interfaces Exposed (Communication with Other Modules)

### GenerateSalesReportInterface

```php
interface GenerateSalesReportInterface
{
    /**
     * Generate sales report for date range.
     * 
     * @throws InvalidReportRequestException
     * @throws DateRangeExceededException
     */
    public function generate(ReportRequest $request): SalesReportData;
}
```

**Consumed by:** Backoffice (Filament pages)

---

### GetDashboardMetricsInterface

```php
interface GetDashboardMetricsInterface
{
    /**
     * Get quick metrics for dashboard.
     */
    public function get(): DashboardMetrics;
}
```

**Consumed by:** Backoffice (Dashboard widgets)

---

## Events Consumed (Cache Invalidation)

### From Orders Module

- `OrderCreated` - Invalidate sales and revenue caches
- `OrderStatusChanged` - Invalidate order status report caches
- `PaymentStatusChanged` - Invalidate payment and revenue caches

---

## Implementation Tasks by Agent

### Task 001: Agent A - Contracts, Data, VOs, Enums
**References:** `@laravel/agents/agent-a-contracts.md`, `@laravel/conventions/value-objects.md`

**Deliverables:**

1. **Value Objects:**
   - `ReportRequest` (startDate, endDate, period, limit, statusFilter, paymentFilter, Wireable)
     - Properties: `DateTime $startDate`, `DateTime $endDate`, `AggregationPeriod $period`, `int $limit` (default: 10), `OrderStatusFilter $statusFilter`, `PaymentStatusFilter $paymentFilter`
     - Methods: `fromArray(array)`, `validate()`, `toArray()`
     - Validation:
       - startDate <= endDate
       - No future dates
       - Date range <= 2 years
       - limit between 1 and 100

2. **Enums:**
   - `AggregationPeriod` (DAILY, WEEKLY, MONTHLY, CUSTOM)
     - Methods: `label()`, `isDaily()`, `isWeekly()`, `isMonthly()`, `isCustom()`
   - `OrderStatusFilter` (ALL, NEW, CONFIRMED, IN_DELIVERY, DELIVERED, REJECTED, CANCELLED)
     - Methods: `label()`, `toOrderStatus()`, `isAll()`
   - `PaymentStatusFilter` (ALL, PENDING, PAID, REFUNDED)
     - Methods: `label()`, `toPaymentStatus()`, `isAll()`

3. **Casts:**
   - None (uses shared Money casts from Orders/Payments modules)

4. **Data Objects (Spatie Laravel Data):**
   - `SalesReportData` (periodStart, periodEnd, totalOrders, totalRevenue, paidRevenue, pendingRevenue, averageOrderValue, totalProductsSold, uniqueProducts, periodBreakdown[])
   - `ProductPerformanceData` (productId, productName, totalQuantitySold, totalRevenue, orderCount, averagePrice, firstOrderDate, lastOrderDate)
   - `OrderStatusDistributionData` (periodStart, periodEnd, totalOrders, statusCounts[], statusPercentages[])
   - `PaymentStatusDistributionData` (periodStart, periodEnd, totalAmount, paymentCounts[], paymentPercentages[])
   - `RevenueReportData` (periodStart, periodEnd, totalRevenue, paidRevenue, pendingRevenue, refundedRevenue, totalOrders, paidOrders, pendingOrders, averageOrderValue, dailyRevenue[])
   - `DashboardMetrics` (todayOrders, weekOrders, monthRevenue, monthPaidRevenue, topProducts[], recentOrderStatuses[])
   - `PeriodSalesData` (periodLabel, periodStart, periodEnd, orderCount, revenue, paidRevenue, productCount)
   - `StatusCountData` (status, count, percentage, totalValue)
   - `PaymentCountData` (status, count, percentage, totalValue)
   - `DailyRevenueData` (date, revenue, paidRevenue, orderCount)

5. **Contracts (Queries):**
   - `GenerateSalesReportInterface`
   - `GenerateProductPerformanceInterface`
   - `GenerateOrderStatusReportInterface`
   - `GeneratePaymentStatusReportInterface`
   - `GenerateRevenueReportInterface`
   - `GetDashboardMetricsInterface`

**Acceptance Criteria:**
- [ ] All Value Objects are `final readonly` with `Wireable`
- [ ] ReportRequest validates all business rules (date range, future dates, 2 year limit)
- [ ] All Enums have label() methods for display
- [ ] OrderStatusFilter/PaymentStatusFilter can convert to actual status enums
- [ ] All Data Objects use Spatie Laravel Data
- [ ] All Data Objects include Money objects (not raw cents)
- [ ] Unit tests for ReportRequest validation (all edge cases)
- [ ] Unit tests for Enum methods
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 002: Agent B - Actions, Services and Tests
**References:** `@laravel/agents/agent-b-actions.md`

**Deliverables:**

1. **Actions Queries:**
   - `GenerateSalesReportAction`
     - Validate ReportRequest
     - Call SalesReportService
     - Return SalesReportData
   - `GenerateProductPerformanceAction`
     - Validate ReportRequest
     - Call ProductPerformanceService
     - Return ProductPerformanceData[]
   - `GenerateOrderStatusReportAction`
     - Validate ReportRequest
     - Call OrderStatusReportService
     - Return OrderStatusDistributionData
   - `GeneratePaymentStatusReportAction`
     - Validate ReportRequest
     - Call PaymentStatusReportService
     - Return PaymentStatusDistributionData
   - `GenerateRevenueReportAction`
     - Validate ReportRequest
     - Call RevenueReportService
     - Return RevenueReportData
   - `GetDashboardMetricsAction`
     - Call DashboardMetricsService
     - Return DashboardMetrics

2. **Actions Internal:**
   - `AggregateByPeriodAction` (group orders by day/week/month)
   - `CalculateTotalsAction` (sum revenues, counts)
   - `CalculateAOVAction` (average order value)
   - `GroupByProductAction` (group order items by product_id)
   - `CalculatePercentagesAction` (calculate distribution percentages)
   - `ValidateReportRequestAction` (validate all business rules)

3. **Services:**
   - `SalesReportService`
     - Dependencies: ReportOrderRepository, ReportOrderItemRepository
     - Methods: `generate(ReportRequest): SalesReportData`
     - Private: `aggregateByPeriod()`, `calculateTotals()`, `calculateAOV()`
   - `ProductPerformanceService`
     - Dependencies: ReportOrderItemRepository, ProductRepository (from Catalog)
     - Methods: `generate(ReportRequest): ProductPerformanceData[]`
     - Private: `groupByProduct()`, `calculateProductMetrics()`, `sortByRevenue()`
   - `OrderStatusReportService`
     - Dependencies: ReportOrderRepository
     - Methods: `generate(ReportRequest): OrderStatusDistributionData`
     - Private: `countByStatus()`, `calculatePercentages()`, `calculateValueByStatus()`
   - `PaymentStatusReportService`
     - Dependencies: ReportOrderRepository
     - Methods: `generate(ReportRequest): PaymentStatusDistributionData`
     - Private: `countByPaymentStatus()`, `calculatePercentages()`
   - `RevenueReportService`
     - Dependencies: ReportOrderRepository
     - Methods: `generate(ReportRequest): RevenueReportData`
     - Private: `calculateTotalRevenue()`, `filterByPaymentStatus()`, `aggregateByDay()`
   - `DashboardMetricsService`
     - Dependencies: ReportOrderRepository, ProductPerformanceService
     - Methods: `getMetrics(): DashboardMetrics`
     - Private: `getTodayOrders()`, `getWeekOrders()`, `getMonthRevenue()`, `getTopProducts()`

4. **Exceptions:**
   - `InvalidReportRequestException` (422, "Invalid report request: {details}")
   - `DateRangeExceededException` (400, "Date range exceeds maximum of 2 years")
   - `FutureDateNotAllowedException` (400, "Future dates are not allowed")

**Unit Tests:**
- Mock all repositories
- Test SalesReportService calculations (totals, AOV, aggregations)
- Test ProductPerformanceService grouping and sorting
- Test OrderStatusReportService percentages calculation
- Test PaymentStatusReportService filtering
- Test RevenueReportService breakdown by payment status
- Test DashboardMetricsService quick metrics
- Test all validation rules (date range, future dates, limits)
- 100% coverage of calculation logic

**Acceptance Criteria:**
- [ ] All Actions are `final` with single public method (`execute()`)
- [ ] All Services are `final` with clear responsibilities
- [ ] All calculations use Money objects (not raw integers)
- [ ] AOV calculation handles division by zero gracefully
- [ ] Percentage calculations return 0-100 with 2 decimal precision
- [ ] Services use database aggregations (COUNT, SUM) not collection methods
- [ ] All exceptions extend `DomainException` with HTTP status
- [ ] Unit tests with mocks (no database)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 003: Agent C - Repositories (No Models/Migrations)
**References:** `@laravel/agents/agent-c-persistence.md`

**Note:** This module does **not create models or migrations** as it only reads from existing tables.

**Deliverables:**

1. **Repositories:**
   - `ReportOrderRepository` (queries Order model from Orders module)
     - Methods:
       - `findByDateRange(DateTime $start, DateTime $end): Collection`
       - `findByDateRangeAndStatus(DateTime $start, DateTime $end, string $status): Collection`
       - `findByDateRangeAndPaymentStatus(DateTime $start, DateTime $end, string $status): Collection`
       - `countByStatus(DateTime $start, DateTime $end): array`
       - `countByPaymentStatus(DateTime $start, DateTime $end): array`
     - Uses: Eloquent query builder with eager loading (orderItems)
     - Caching: 5 minutes TTL, cache key includes all parameters
   - `ReportOrderItemRepository` (queries OrderItem model from Orders module)
     - Methods:
       - `findByDateRange(DateTime $start, DateTime $end): Collection`
       - `groupByProduct(DateTime $start, DateTime $end): Collection`
       - `sumQuantitiesByProduct(DateTime $start, DateTime $end): array`
       - `sumRevenueByProduct(DateTime $start, DateTime $end): array`
     - Uses: DB aggregations (SUM, COUNT, GROUP BY)
     - Joins: with orders table for date filtering
     - Caching: 5 minutes TTL

**Integration Tests:**
- Seed Orders with various statuses and dates
- Test date range filtering
- Test status filtering
- Test payment status filtering
- Test aggregations (COUNT, SUM)
- Test eager loading (no N+1)
- Test caching behavior

**Acceptance Criteria:**
- [ ] All repositories are `final`
- [ ] All repositories query existing models (no new models created)
- [ ] All repositories use query builder for aggregations (not collections)
- [ ] All repositories eager load relationships to avoid N+1
- [ ] All repositories implement caching (5 minutes)
- [ ] Cache keys include all query parameters
- [ ] Integration tests with seeded data
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 004: Agent D - HTTP, Filament, Tests Feature
**References:** `@laravel/agents/agent-d-http.md`

**Deliverables:**

1. **HTTP Controllers:**
   - `ReportController` (API endpoints for reports)
     - `GET /backoffice/reports/sales` → sales report
     - `GET /backoffice/reports/products` → product performance
     - `GET /backoffice/reports/orders/status` → order status distribution
     - `GET /backoffice/reports/payments/status` → payment status distribution
     - `GET /backoffice/reports/revenue` → revenue report
     - All protected by Auth middleware
     - All rate limited (60/min per merchant)
     - All responses cached (5 minutes)

2. **API Routes (routes/api.php):**
   - All routes under `/backoffice/reports/` prefix
   - All require authentication
   - All return JSON (Data Objects)

3. **Filament Pages (Backoffice):**
   - `SalesReport` (form with date range, period, filters; displays SalesReportData)
   - `ProductPerformance` (form with date range, limit, sort; displays table of products)
   - `OrderStatusReport` (form with date range; displays pie chart and table)
   - `PaymentStatusReport` (form with date range; displays pie chart and table)
   - `RevenueReport` (form with date range; displays line chart and breakdown)

4. **Filament Widgets (Dashboard):**
   - `DashboardStats` (today orders, week orders, month revenue cards)
   - `TopProducts` (top 5 products this month)
   - `RecentOrders` (last 10 orders with status badges)
   - `RevenueChart` (last 30 days revenue line chart)

**Feature Tests:**
- Sales report generation (various date ranges, periods, filters)
- Product performance report (top N products)
- Order status distribution
- Payment status distribution
- Revenue report with payment breakdown
- Dashboard metrics retrieval
- Caching behavior (cache hit on second request)
- Cache invalidation on OrderCreated event
- Rate limiting enforcement
- Authentication requirement
- Validation errors (invalid date range, future dates, exceeds 2 years)
- Smoke tests for all Filament pages

**Acceptance Criteria:**
- [ ] All API endpoints protected by Auth middleware
- [ ] All API endpoints rate limited (60/min)
- [ ] All responses cached (5 minutes)
- [ ] All responses return Data Objects (JSON serialized)
- [ ] Filament pages have form validation
- [ ] Filament widgets cached (1 minute)
- [ ] Charts use appropriate libraries (Chart.js or Filament charts)
- [ ] Feature tests cover all endpoints
- [ ] Feature tests verify caching behavior
- [ ] Smoke tests for all Filament pages
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

---

### Task 005: Agent E - Events, Listeners
**References:** `@laravel/agents/agent-e-events.md`

**Deliverables:**

1. **Events:**
   - None (this module only consumes events, does not emit)

2. **Listeners:**
   - `InvalidateSalesCacheOnOrderCreated` (listens to `OrderCreated` from Orders)
     - Flush all sales report caches
     - Flush dashboard metrics cache
   - `InvalidateOrderStatusCacheOnStatusChange` (listens to `OrderStatusChanged` from Orders)
     - Flush order status report caches
     - Flush dashboard metrics cache
   - `InvalidatePaymentCacheOnPaymentChange` (listens to `PaymentStatusChanged` from Payments)
     - Flush payment status report caches
     - Flush revenue report caches
     - Flush dashboard metrics cache

3. **Jobs:**
   - None (cache invalidation is fast, no queue needed)

**Event Tests:**
- Event dispatching triggers cache invalidation
- Listener execution (verify cache cleared)
- Multiple cache tags cleared correctly

**Acceptance Criteria:**
- [ ] All listeners are `final`
- [ ] Listeners clear specific cache tags/keys
- [ ] Listeners do not throw exceptions (graceful failure)
- [ ] Event tests verify cache invalidation
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
- [ ] Unit tests for all Value Objects (validation)
- [ ] Unit tests for all Services (calculation logic)
- [ ] Feature tests for all API endpoints
- [ ] Feature tests for caching behavior
- [ ] Integration tests for repositories (aggregations)
- [ ] Smoke tests for all Filament pages
- [ ] 100% coverage for calculation logic

### Documentation
- [ ] All public methods have PHPDoc
- [ ] Complex calculations have inline comments
- [ ] README.md in module root (overview, usage)

### Database
- [ ] No new models or migrations (read-only module)
- [ ] Repositories use existing models from other modules
- [ ] All queries use database aggregations (not collection methods)
- [ ] All queries eager load to avoid N+1

### Performance
- [ ] All reports cached (5 minutes)
- [ ] Dashboard metrics cached (1 minute)
- [ ] Cache keys include all parameters
- [ ] Database aggregations used (COUNT, SUM, GROUP BY)
- [ ] Indexes on foreign keys verified (Orders.created_at, etc.)

### Security
- [ ] All endpoints protected by Auth middleware
- [ ] Rate limiting enforced (60/min)
- [ ] No customer PII exposed (aggregate only)
- [ ] No writes to other modules

---

## Anti-Patterns to Avoid

❌ **DO NOT:**
- Write to Orders, Payments, or Catalog tables (read-only)
- Use collection methods for aggregations (use database queries)
- Return raw database results (use Data Objects)
- Skip caching (performance killer)
- Expose customer PII in reports
- Allow unauthenticated access
- Use raw cents instead of Money objects
- Calculate percentages in controllers (use services)
- Forget to invalidate caches on data changes

✅ **DO:**
- Use database aggregations (COUNT, SUM, GROUP BY)
- Return Data Objects with Money objects
- Cache all report results (5 minutes)
- Listen to events for cache invalidation
- Eager load relationships (avoid N+1)
- Validate date ranges and parameters
- Use Enums for filters
- Calculate AOV safely (handle division by zero)
- Apply rate limiting

---

## Success Criteria

This module is considered **complete** when:

1. **Functional:**
   - Merchants can generate sales reports with various date ranges and filters
   - Merchants can view top selling products
   - Merchants can analyze order status distribution
   - Merchants can monitor payment status
   - Merchants can view revenue breakdown
   - Dashboard shows quick metrics
   - All reports properly cached
   - Caches invalidate on data changes

2. **Quality:**
   - PHPStan level 6+ passes
   - Pint executed (no formatting issues)
   - 100% test coverage for calculation logic
   - All acceptance criteria checked

3. **Performance:**
   - No N+1 queries
   - Database aggregations used (not collection methods)
   - All reports cached (5 minutes)
   - Dashboard cached (1 minute)
   - Report generation completes in < 1s (even for large date ranges)

4. **Security:**
   - All endpoints authenticated (merchant only)
   - Rate limiting enforced (60/min)
   - No customer PII exposed
   - No writes to other modules

---

## Configuration

### Environment Variables

```env
# Reports Module
REPORTS_CACHE_TTL=300
DASHBOARD_CACHE_TTL=60
REPORTS_RATE_LIMIT=60
REPORTS_MAX_DATE_RANGE_YEARS=2
REPORTS_DEFAULT_DATE_RANGE_DAYS=30
```

### Config File: `config/reports.php`

```php
return [
    'cache_ttl' => env('REPORTS_CACHE_TTL', 300), // 5 minutes
    'dashboard_cache_ttl' => env('DASHBOARD_CACHE_TTL', 60), // 1 minute
    'rate_limit' => env('REPORTS_RATE_LIMIT', 60),
    'max_date_range_years' => env('REPORTS_MAX_DATE_RANGE_YEARS', 2),
    'default_date_range_days' => env('REPORTS_DEFAULT_DATE_RANGE_DAYS', 30),
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
./vendor/bin/sail test Modules/Reports

# Test with coverage
./vendor/bin/sail test --coverage
```

---

## Dependencies and Integration

### Modules that Depend on Reports

- None (this is a terminal module)

### External Dependencies

- **Orders**: Read Order and OrderItem models
- **Payments**: Read payment status
- **Catalog**: Read Product model
- **Auth**: Validate merchant authentication

### Events Consumed

- `OrderCreated` (Orders) - Invalidate sales caches
- `OrderStatusChanged` (Orders) - Invalidate order status caches
- `PaymentStatusChanged` (Payments) - Invalidate payment and revenue caches

---

## Final Notes

- This module is **Phase 4 (Post-MVP)** - implement after Orders, Payments, Cart are complete.
- This is a **read-only module** - no writes to other modules.
- **Performance is critical** - use database aggregations and caching aggressively.
- **Security is important** - merchant authentication required, no PII exposed.
- Follow the 5-agent architecture strictly.
- All code must pass PHPStan level 6+ and Pint before commit.

**Expected Total Time:** 12-16 hours (distributed across 5 agents)

---

**This prompt is optimized for AI agent consumption and human developer guidance.**
