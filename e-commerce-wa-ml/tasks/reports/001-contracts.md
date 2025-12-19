---
task_id: "reports-001-contracts"
module: "Reports"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Reports - Contracts, Value Objects, Enums y Data Objects"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "orders-001-contracts"
  - "payments-001-contracts"
  - "catalog-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/reports/agents_prompt.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 4 - Post-MVP"
---

# Task 001: Reports - Contracts, Value Objects, Enums y Data Objects

## 🎯 Objetivo

Implementar los contratos públicos del módulo Reports: Value Objects, Enums y DTOs para reportes de ventas, análisis de productos y métricas del dashboard. Módulo read-only y backoffice-only.

## 📋 Contexto

El módulo Reports es STANDARD y post-MVP. Consume datos de Orders, Payments y Catalog para generar reportes analíticos sin modificar datos fuente.

## 📦 Artefactos a Crear

### Value Objects (6)

1. **DateRange** - Rango de fechas (start/end, max 2 años, Wireable)
2. **TimeAggregation** - Agregación temporal (daily/weekly/monthly, Wireable)
3. **Revenue** - Revenue con breakdown (total, paid, pending, Wireable)
4. **Money** - Valor monetario en cents (shared, Wireable)
5. **Percentage** - Porcentaje con validación (0-100, Wireable)
6. **ReportMetric** - Métrica genérica (label, value, change, Wireable)

### Enums (3)

1. **ReportType** - SALES | PRODUCTS | ORDERS | PAYMENTS | DASHBOARD
   - Methods: label(), cacheDuration(), requiresDateRange()
   
2. **AggregationPeriod** - DAILY | WEEKLY | MONTHLY | YEARLY
   - Methods: label(), sqlGroupBy(), dateFormat()
   
3. **SortOrder** - ASC | DESC
   - Methods: label(), sqlDirection()

### Casts (4)

1. **DateRangeCast** - json ↔ DateRange
2. **TimeAggregationCast** - string ↔ TimeAggregation
3. **RevenueCast** - json ↔ Revenue
4. **MoneyCast** - cents ↔ Money (shared)

### Data Objects (8)

1. **SalesReportData** - Sales over time with aggregation
   - Fields: date_range, aggregation, revenue, order_count, avg_order_value, data_points
   
2. **ProductPerformanceData** - Top products by revenue/quantity
   - Fields: product_id, name, quantity_sold, revenue, order_count
   
3. **OrderStatusDistributionData** - Orders by status
   - Fields: status, count, percentage, revenue
   
4. **PaymentMethodDistributionData** - Payments by method
   - Fields: method, count, percentage, revenue
   
5. **DashboardMetricsData** - Quick metrics for dashboard
   - Fields: today_revenue, week_revenue, month_revenue, total_orders, pending_orders, low_stock_products
   
6. **ReportFilterData** - Common filters for reports
   - Fields: date_range, aggregation, limit, sort_by, sort_order
   
7. **DataPointData** - Single point in time series
   - Fields: date, label, value, count
   
8. **ComparisonMetricData** - Metric with period comparison
   - Fields: current_value, previous_value, change_percentage, trend (up/down/stable)

## 🔑 Business Rules

### Date Range Rules
- Max range: 2 years (730 days)
- Start date must be <= end date
- Dates in merchant timezone
- Future dates not allowed

### Aggregation Rules
- DAILY: max 90 days range
- WEEKLY: max 1 year range
- MONTHLY: max 2 years range
- YEARLY: no limit

### Caching Rules
- Reports: 5 minutes TTL
- Dashboard: 1 minute TTL
- Cache key includes: report_type, filters, merchant_id

### Performance Rules
- Max 1000 data points per report
- Queries limited to indexed columns only
- Heavy reports queued for background processing
- Rate limit: 60 requests/minute per merchant

## ✅ Criterios de Aceptación

### Funcionales
- [ ] DateRange validates max 2 years
- [ ] TimeAggregation maps to SQL GROUP BY
- [ ] Revenue calculates paid/pending breakdown
- [ ] ReportMetric supports trend indicators
- [ ] All VOs validate in constructor
- [ ] All Casts bidirectional

### Técnicos
- [ ] All classes are `final`
- [ ] VOs are `readonly`
- [ ] Strong typing everywhere
- [ ] Unit tests for all VOs (100%)
- [ ] Tests for Enums
- [ ] Tests for Casts
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Post-MVP  
**Bloqueante para:** Business analytics, Dashboard metrics
