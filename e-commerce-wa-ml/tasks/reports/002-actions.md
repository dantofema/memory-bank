---
task_id: "reports-002-actions"
module: "Reports"
agent: "Agente B - Actions y Tests Unitarios"
title: "Reports - Actions de Lógica de Negocio"
priority: "MEDIUM"
estimated_time: "10 hours"
dependencies:
  - "reports-001-contracts"
  - "orders-002-actions"
  - "payments-002-actions"
  - "catalog-002-actions"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/reports/agents_prompt.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
phase: "Fase 4 - Post-MVP"
---

# Task 002: Reports - Actions de Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Reports: generación de reportes, cálculos analíticos, agregaciones temporales y métricas del dashboard. Todo read-only con caching agresivo.

## 📦 Artefactos a Crear

### Actions Queries (12) - NO Commands (read-only module)

1. **GenerateSalesReportAction** - Reporte de ventas
   - Date range + aggregation
   - Revenue calculation (paid only)
   - Order count
   - Average order value
   - Time series data points
   - Cache 5 minutes

2. **GenerateProductPerformanceReportAction** - Top productos
   - Date range filter
   - Sort by revenue or quantity
   - Limit results (default 20)
   - Include product details
   - Cache 5 minutes

3. **GenerateOrderStatusReportAction** - Distribución por estado
   - Date range filter
   - Count by status
   - Percentage calculation
   - Revenue by status
   - Cache 5 minutes

4. **GeneratePaymentMethodReportAction** - Distribución por método
   - Date range filter
   - Count by method
   - Percentage calculation
   - Revenue by method
   - Cache 5 minutes

5. **GetDashboardMetricsAction** - Métricas rápidas
   - Today/week/month revenue
   - Total orders
   - Pending orders count
   - Low stock products count
   - Cache 1 minute

6. **ComparePeriodsAction** - Comparar períodos
   - Two date ranges
   - Calculate change percentage
   - Trend indicator (up/down/stable)
   - All metrics comparison

7. **GetRevenueBreakdownAction** - Desglose de revenue
   - By payment status (paid/pending/refunded)
   - By payment method
   - Percentages

8. **GetTopCustomersAction** - Top clientes (by phone)
   - Order count per phone
   - Total spent
   - Average order value
   - Privacy: phone masked

9. **GetLowStockProductsAction** - Productos con stock bajo
   - Threshold configurable
   - Product details
   - Stock level
   - Sales velocity

10. **ExportReportAction** - Exportar a CSV/PDF
    - Any report type
    - Format selection
    - Background job for large reports

11. **CalculateGrowthRateAction** - Calcular crecimiento
    - Period over period
    - Percentage calculation
    - Trend analysis

12. **GetReportCacheStatusAction** - Estado del cache
    - Check if cached
    - TTL remaining
    - Cache hit/miss stats

### Actions Internal (4)

13. **ApplyDateRangeFilterAction** - Aplicar filtro de fechas
14. **AggregateByPeriodAction** - Agrupar por período
15. **CalculatePercentagesAction** - Calcular porcentajes
16. **BuildCacheKeyAction** - Generar clave de cache

### Excepciones de Dominio (5)

- InvalidDateRangeException (422)
- DateRangeTooLargeException (422)
- ReportNotFoundException (404)
- RateLimitExceededException (429)
- ReportGenerationFailedException (500)

## 🔑 Key Business Rules

### Date Range (Rules 1-4)
1. Max 2 years (730 days)
2. Start <= end date
3. No future dates
4. Merchant timezone

### Aggregation Limits (Rules 5-8)
5. DAILY: max 90 days
6. WEEKLY: max 1 year
7. MONTHLY: max 2 years
8. YEARLY: unlimited

### Revenue Calculation (Rules 9-12)
9. Only PAID payments count
10. Refunds subtract from revenue
11. Pending payments excluded
12. All amounts in merchant currency

### Performance (Rules 13-16)
13. Cache reports 5 minutes
14. Cache dashboard 1 minute
15. Max 1000 data points per report
16. Heavy reports queued

### Privacy (Rules 17-18)
17. No customer PII exposed
18. Phone numbers masked in reports

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Sales report generates time series
- [ ] Product performance calculates correctly
- [ ] Dashboard metrics update every minute
- [ ] Date range validation enforced
- [ ] Revenue only counts paid payments
- [ ] Caching works correctly
- [ ] Export to CSV/PDF works

### Técnicos
- [ ] All Actions are `final`
- [ ] One public method per Action
- [ ] Dependency injection
- [ ] Unit tests with mocks
- [ ] 100% coverage critical paths
- [ ] Cache tested
- [ ] Rate limiting tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Post-MVP  
**Depende de:** Task 001, Orders/Payments/Catalog 002
