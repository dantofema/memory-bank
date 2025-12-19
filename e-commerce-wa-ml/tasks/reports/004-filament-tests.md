---
task_id: "reports-004-filament-tests"
module: "Reports"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Reports - Filament Widgets y Tests Feature"
priority: "MEDIUM"
estimated_time: "8 hours"
dependencies:
  - "reports-001-contracts"
  - "reports-002-actions"
  - "reports-003-persistence"
  - "auth-004-http-filament"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/reports/agents_prompt.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
phase: "Fase 4 - Post-MVP"
---

# Task 004: Reports - Filament Widgets y Tests Feature

## 🎯 Objetivo

Implementar Filament widgets y pages para reportes analíticos en el backoffice. NO hay UI pública (módulo exclusivo de merchant). Tests feature de todos los reportes.

## 📦 Artefactos a Crear

### Filament Pages (2)

1. **ReportsPage** - Página principal de reportes
   - Tabs: Sales, Products, Orders, Payments
   - Date range selector (with presets: today, week, month, custom)
   - Aggregation selector (daily, weekly, monthly)
   - Export button (CSV, PDF)
   - Charts with Chart.js/ApexCharts

2. **DashboardPage** - Dashboard con métricas rápidas
   - Revenue cards (today, week, month)
   - Order status chart
   - Top products table
   - Recent orders list
   - Auto-refresh every minute

### Filament Widgets (8)

1. **RevenueTrendWidget** - Gráfico de tendencia de revenue
   - Line chart con time series
   - Period selector
   - Comparison with previous period

2. **OrderStatusChartWidget** - Gráfico de distribución de órdenes
   - Pie/donut chart
   - Count + percentage
   - Filtro por fecha

3. **TopProductsTableWidget** - Tabla de top productos
   - Sortable columns
   - Revenue + quantity
   - Limit selector (10/20/50)

4. **PaymentMethodChartWidget** - Distribución de métodos de pago
   - Bar chart
   - Revenue por método
   - Percentage

5. **RevenueStatsWidget** - Cards de revenue
   - Today, week, month
   - Change percentage vs previous period
   - Trend indicator (↑/↓)

6. **OrderStatsWidget** - Cards de órdenes
   - Total orders
   - Pending orders
   - Average order value

7. **LowStockAlertsWidget** - Alertas de stock bajo
   - List de productos
   - Current stock
   - Sales velocity
   - Link to Catalog

8. **RecentOrdersWidget** - Órdenes recientes
   - Last 5-10 orders
   - Quick view
   - Link to Orders module

### No Public UI
**Note:** Reports module is **backoffice only**. All UI requires merchant authentication.

### Feature Tests (5)

**SalesReportTest.php:**
- Generate sales report with date range
- Aggregation by period works
- Revenue calculates correctly (paid only)
- Caching reduces query time
- Export to CSV works

**ProductPerformanceTest.php:**
- Top products by revenue
- Top products by quantity
- Low stock products listed
- Sorting works correctly

**OrderStatusReportTest.php:**
- Distribution by status
- Revenue by status
- Percentage calculation correct

**PaymentMethodReportTest.php:**
- Distribution by method
- Revenue by method
- Refunds excluded from revenue

**DashboardTest.php:**
- Dashboard metrics load quickly (< 1s)
- Cache reduces database queries
- Auto-refresh works
- All widgets display correctly

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Merchant can view all reports
- [ ] Date range selector works
- [ ] Charts display correctly
- [ ] Export to CSV/PDF works
- [ ] Dashboard loads fast (< 1s)
- [ ] Cache improves performance
- [ ] Widgets update on refresh
- [ ] No customer PII exposed

### Técnicos
- [ ] Filament pages fully functional
- [ ] Widgets use cached data
- [ ] Charts render correctly
- [ ] Feature tests cover all reports
- [ ] Performance tests (query count)
- [ ] Rate limiting tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Post-MVP  
**Depende de:** Task 001, 002, 003, Auth 004
