---
task_id: "catalog-005-events"
module: "Catalog"
agent: "Agente E - Events, Listeners y Jobs"
title: "Catalog - Eventos de Dominio y Listeners"
priority: "MEDIUM"
estimated_time: "6 hours"
dependencies:
  - "catalog-001-contracts"
  - "catalog-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/catalog/agents_prompt.md"
phase: "Fase 1 - Fundamentos"
---

# Task 005: Catalog - Eventos de Dominio y Listeners

## 🎯 Objetivo

Implementar eventos de dominio para auditoría y notificaciones, especialmente eventos de stock bajo y agotado para alertas futuras.

## 📦 Artefactos a Crear

### Domain Events (5)

1. **ProductCreated** - Producto creado
   - Payload: ProductId, name, CategoryId, created_at

2. **ProductUpdated** - Producto actualizado
   - Payload: ProductId, changed_fields, updated_at

3. **ProductDeleted** - Producto eliminado
   - Payload: ProductId, deleted_at

4. **ProductStockLowEvent** - Stock bajo
   - Payload: ProductId, current_stock, threshold, product_name

5. **ProductOutOfStockEvent** - Stock agotado
   - Payload: ProductId, product_name, out_of_stock_at

### Event Listeners (1)

1. **NotifyLowStockListener** - Listener para ProductStockLowEvent
   - MVP: Log event for reporting
   - Future: Send notification to merchant (email/WhatsApp)
   - Implements ShouldQueue

### Jobs
None for MVP (future: sync with external systems, ML recommendations)

## 🔑 Event Flow

```
Product Created → ProductCreated
Product Updated → ProductUpdated
  ↓ (if stock changed)
  ↓ Stock < threshold → ProductStockLowEvent → NotifyLowStockListener
  ↓ Stock = 0 → ProductOutOfStockEvent

Product Deleted → ProductDeleted
```

## 🔑 Business Rules

### Stock Events
- Low stock event when stock < threshold (default: 10)
- Out of stock event when stock = 0
- Threshold configurable via env: `STOCK_LOW_THRESHOLD=10`
- Events only dispatched on stock decrease
- Events include product name for easy identification

### Event Timing
- ProductCreated: after product saved
- ProductUpdated: after product saved (with diff)
- ProductDeleted: before actual deletion
- Stock events: after stock update if threshold crossed

### Listener Behavior
- NotifyLowStockListener is queued
- Idempotent (can be retried safely)
- Logs event for reporting dashboard
- Future: Send merchant notification

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Events dispatched at correct moments
- [ ] Stock events triggered at thresholds
- [ ] Listener logs low stock events
- [ ] No duplicate events for same state
- [ ] ProductUpdated includes changed fields
- [ ] Events include all necessary data

### Técnicos
- [ ] All events are `final readonly`
- [ ] All listeners are `final`
- [ ] Listener implements ShouldQueue
- [ ] Event tests verify dispatching
- [ ] Listener tests verify execution
- [ ] Idempotency tested
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

## 🧪 Tests Requeridos

```php
describe('Catalog Events', function () {
    it('dispatches ProductCreated on product creation');
    it('dispatches ProductUpdated on product update');
    it('dispatches ProductDeleted before deletion');
    it('dispatches ProductStockLowEvent when stock < threshold');
    it('dispatches ProductOutOfStockEvent when stock = 0');
    it('does not dispatch low stock event if already low');
});

describe('NotifyLowStockListener', function () {
    it('logs low stock event');
    it('is idempotent');
    it('handles queue failures gracefully');
});
```

## 📚 Configuration

Add to `.env`:
```env
STOCK_LOW_THRESHOLD=10
```

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001, 003
