---
task_id: "reports-005-events"
module: "Reports"
agent: "Agente E - Events, Listeners y Jobs"
title: "Reports - Eventos y Jobs (Cache Warming)"
priority: "LOW"
estimated_time: "4 hours"
dependencies:
  - "reports-001-contracts"
  - "reports-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/reports/agents_prompt.md"
  - "@e-commerce-wa-ml/reports/domain_model.md"
phase: "Fase 4 - Post-MVP"
---

# Task 005: Reports - Eventos y Jobs (Cache Warming)

## 🎯 Objetivo

Implementar jobs programados para pre-calentar cache de reportes frecuentes y listeners para invalidar cache cuando cambian datos fuente.

## 📦 Artefactos a Crear

### Domain Events (2) - Minimal

1. **ReportGenerated** - Reporte generado
   - Payload: report_type, filters, generation_time, cached

2. **ReportExported** - Reporte exportado
   - Payload: report_type, format (CSV/PDF), exported_at

### Event Listeners (3)

1. **InvalidateReportCacheOnOrderCreated** - Listens to OrderCreated (from Orders module)
   - Invalidates sales report cache
   - Invalidates dashboard cache
   - Non-blocking (queued)

2. **InvalidateReportCacheOnPaymentStatusChanged** - Listens to PaymentStatusChanged (from Payments module)
   - Invalidates payment reports cache
   - Invalidates revenue cache
   - Non-blocking (queued)

3. **InvalidateReportCacheOnProductUpdated** - Listens to ProductUpdated (from Catalog module)
   - Invalidates product reports cache
   - Non-blocking (queued)

### Scheduled Jobs (3)

1. **WarmDashboardCacheJob** - Pre-calienta dashboard
   - Runs every 5 minutes
   - Generates dashboard metrics
   - Stores in cache
   - Priority: high

2. **WarmSalesReportCacheJob** - Pre-calienta reportes comunes
   - Runs every hour
   - Generates: today, week, month sales reports
   - Stores in cache
   - Priority: medium

3. **CleanExpiredReportCacheJob** - Limpia cache expirado
   - Runs daily at 3 AM
   - Removes old cached reports
   - Frees memory
   - Priority: low

### No Commands
This is a read-only module. No commands, only cache management.

## 🔑 Event Flow

```
Order Created → OrderCreated
  → InvalidateReportCacheOnOrderCreated
    → Flush sales cache
    → Flush dashboard cache

Payment Status Changed → PaymentStatusChanged
  → InvalidateReportCacheOnPaymentStatusChanged
    → Flush payment cache
    → Flush revenue cache

Product Updated → ProductUpdated
  → InvalidateReportCacheOnProductUpdated
    → Flush product cache

Scheduled Jobs:
  Every 5 min → WarmDashboardCacheJob
  Every 1 hour → WarmSalesReportCacheJob
  Daily 3 AM → CleanExpiredReportCacheJob
```

## 🔑 Business Rules

### Cache Invalidation
- Invalidate on source data change
- Selective invalidation (only affected reports)
- Non-blocking (queued)
- Graceful degradation if invalidation fails

### Cache Warming
- Pre-generate common reports
- Run during low-traffic hours
- Priority: dashboard > sales > products
- Max 5 minutes per job

### Cache Cleanup
- Remove expired entries daily
- Keep max 1000 cached reports
- Remove oldest first (LRU)
- Log cleanup stats

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Cache invalidated on data changes
- [ ] Dashboard pre-warmed every 5 minutes
- [ ] Common reports pre-generated
- [ ] Expired cache cleaned daily
- [ ] No performance impact on source modules

### Técnicos
- [ ] All listeners are `final`
- [ ] All jobs are `final`
- [ ] Listeners queued (non-blocking)
- [ ] Jobs scheduled correctly
- [ ] Event tests verify dispatching
- [ ] Job tests verify execution
- [ ] Cache warming tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## 🧪 Tests Requeridos

```php
describe('Report Cache Invalidation', function () {
    it('invalidates sales cache on order created');
    it('invalidates payment cache on status change');
    it('invalidates product cache on update');
    it('handles invalidation failures gracefully');
});

describe('Cache Warming Jobs', function () {
    it('warms dashboard cache every 5 minutes');
    it('warms sales reports every hour');
    it('cleans expired cache daily');
    it('handles job failures gracefully');
});
```

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Post-MVP  
**Depende de:** Task 001, 003
