---
task_id: "reports-003-persistence"
module: "Reports"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Reports - Repositorios Read-Only (No Modelos)"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "reports-001-contracts"
  - "orders-003-persistence"
  - "payments-003-persistence"
  - "catalog-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/reports/agents_prompt.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
phase: "Fase 4 - Post-MVP"
---

# Task 003: Reports - Repositorios Read-Only (No Modelos Propios)

## 🎯 Objetivo

Implementar repositorios read-only que consultan datos de Orders, Payments y Catalog para generar reportes. **No hay modelos propios ni migraciones** (solo lectura).

## 📋 Contexto

Este módulo NO tiene modelos propios. Lee datos de otros módulos mediante repositorios especializados con queries optimizadas y caching.

## 📦 Artefactos a Crear

### Repositorios Read-Only (5)

1. **SalesReportRepository** - Consultas de ventas
   - Methods:
     - `getSalesByDateRange(DateRange $range, TimeAggregation $agg): array`
     - `getRevenueByPeriod(DateRange $range, AggregationPeriod $period): array`
     - `getAverageOrderValue(DateRange $range): Money`
     - `getOrderCountByPeriod(DateRange $range, AggregationPeriod $period): array`
   - Uses: Order, Payment models (read-only)
   - Optimizations: Indexed queries, eager loading, select only needed columns

2. **ProductReportRepository** - Consultas de productos
   - Methods:
     - `getTopProductsByRevenue(DateRange $range, int $limit): array`
     - `getTopProductsByQuantity(DateRange $range, int $limit): array`
     - `getProductSalesVelocity(int $productId, DateRange $range): float`
     - `getLowStockProducts(int $threshold): array`
   - Uses: Product, OrderItem models (read-only)
   - Optimizations: Aggregations, joins, indexes

3. **OrderReportRepository** - Consultas de órdenes
   - Methods:
     - `getOrderCountByStatus(DateRange $range): array`
     - `getOrderStatusDistribution(DateRange $range): array`
     - `getRevenueByStatus(DateRange $range): array`
     - `getPendingOrdersCount(): int`
   - Uses: Order model (read-only)
   - Optimizations: Group by status, cached counts

4. **PaymentReportRepository** - Consultas de pagos
   - Methods:
     - `getPaymentMethodDistribution(DateRange $range): array`
     - `getRevenueByPaymentMethod(DateRange $range): array`
     - `getPaidPaymentsRevenue(DateRange $range): Money`
     - `getRefundedAmount(DateRange $range): Money`
   - Uses: Payment model (read-only)
   - Optimizations: Payment status filters, method grouping

5. **DashboardRepository** - Métricas rápidas
   - Methods:
     - `getTodayRevenue(): Money`
     - `getWeekRevenue(): Money`
     - `getMonthRevenue(): Money`
     - `getTotalOrdersCount(): int`
     - `getPendingOrdersCount(): int`
     - `getLowStockProductsCount(int $threshold): int`
   - Uses: All models (read-only)
   - Optimizations: Heavy caching (1 minute TTL), simple counts

### Query Builders (3)

1. **SalesQueryBuilder** - Builder para queries de ventas
   - Fluent interface para construir queries complejas
   - Methods: dateRange(), aggregateBy(), withRevenue(), withOrderCount()

2. **ProductQueryBuilder** - Builder para queries de productos
   - Methods: dateRange(), topBy(), limit(), withDetails()

3. **ReportCacheManager** - Gestión de cache de reportes
   - Methods: get(), put(), forget(), flush()
   - Cache driver: Redis (recommended)
   - TTL: 5 minutes (reports), 1 minute (dashboard)

## 🔑 Database Optimization Rules

### Indexes Required (on other modules)
- orders.created_at (already exists)
- orders.order_status (already exists)
- payments.payment_status (already exists)
- payments.payment_method (already exists)
- order_items.product_id (already exists)

### Query Optimization
- Use `select()` to limit columns
- Use `with()` for eager loading
- Use `cache()` for frequent queries
- Use raw SQL for complex aggregations
- Use database views for complex reports (optional)

### No Migrations Needed
This module reads from existing tables. No new tables created.

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Repositories query existing models correctly
- [ ] Aggregations calculate accurately
- [ ] Caching reduces query load
- [ ] Read-only enforcement (no writes)
- [ ] Performance optimized (< 1s per report)

### Técnicos
- [ ] All repositories are `final`
- [ ] Repositories use query builders
- [ ] Cache manager handles TTL correctly
- [ ] Integration tests with database
- [ ] Performance tests (query time)
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Post-MVP  
**Depende de:** Task 001, Orders/Payments/Catalog 003
