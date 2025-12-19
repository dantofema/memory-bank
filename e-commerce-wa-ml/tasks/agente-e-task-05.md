---
task_id: "agente-e-task-05"
module: "Orders"
agent: "Agente E"
title: "Eventos de Dominio del Módulo Orders"
priority: "high"
estimated_time: "2h"
dependencies:
  - "Agente A: Data Objects del módulo Orders"
  - "Agente B: Actions de Orders implementadas"
  - "Agente C: Repositories de Orders disponibles"
status: "pending"
---

# Task: Eventos de Dominio del Módulo Orders

## Contexto

Crear los eventos de dominio que representan acciones críticas del módulo Orders. Estos eventos son esenciales para **auditoría, trazabilidad y cumplimiento** según la sección 16 del `project_definition.md`.

Los eventos son **inmutables**, **descriptivos** y **representan algo que ya ocurrió**. Se disparan desde las Actions del Agente B y permiten reaccionar con efectos secundarios (auditoría, notificaciones, reportes, sincronización).

**Nota**: El Agente E **NO dispara eventos**, solo los define. Los eventos se disparan desde las Actions del Agente B.

---

## Objetivos

1. ✅ Crear evento `OrderCreated`
2. ✅ Crear evento `OrderStatusChanged`
3. ✅ Crear evento `PaymentStatusChanged`
4. ✅ Crear evento `OrderItemStockReserved`
5. ✅ Tests unitarios de eventos
6. ✅ Validar PHPStan level 6

---

## Eventos a Crear

### 1. Evento: OrderCreated

**Ubicación**: `Modules/Orders/app/Events/OrderCreated.php`

**Descripción**: Se dispara cuando se crea exitosamente un pedido en el sistema.

**Cuándo se dispara**: Desde `CreateOrderAction` después de guardar el pedido en la base de datos.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Events;

use Carbon\Carbon;

/**
 * Evento disparado cuando se crea un pedido exitosamente.
 */
final readonly class OrderCreated
{
    public function __construct(
        public int $orderId,
        public string $customerName,
        public string $customerPhone,
        public int $totalAmountCents,
        public string $orderStatus,
        public string $paymentStatus,
        public string $paymentMethod,
        public int $itemsCount,
        public Carbon $createdAt,
    ) {}
}
```

**Propiedades**:
- `orderId`: ID del pedido creado
- `customerName`: Nombre del cliente
- `customerPhone`: Teléfono del cliente (normalizado)
- `totalAmountCents`: Total del pedido en centavos
- `orderStatus`: Estado inicial del pedido (ej: "new")
- `paymentStatus`: Estado inicial del pago (ej: "pending")
- `paymentMethod`: Método de pago seleccionado
- `itemsCount`: Cantidad de items en el pedido
- `createdAt`: Timestamp de creación

**Uso en Action** (referencia, NO implementar aquí):
```php
// En CreateOrderAction::execute()
event(new OrderCreated(
    orderId: $order->id,
    customerName: $order->customer_name,
    customerPhone: $order->customer_phone->toString(),
    totalAmountCents: $order->total_amount->cents,
    orderStatus: $order->order_status->value,
    paymentStatus: $order->payment_status->value,
    paymentMethod: $order->payment_method->value,
    itemsCount: $order->items->count(),
    createdAt: $order->created_at,
));
```

---

### 2. Evento: OrderStatusChanged

**Ubicación**: `Modules/Orders/app/Events/OrderStatusChanged.php`

**Descripción**: Se dispara cuando cambia el estado de un pedido (OrderStatus). **Crítico para auditoría** según project_definition.md sección 16.

**Cuándo se dispara**: Desde `UpdateOrderStatusAction` después de cambiar el estado.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Events;

use Carbon\Carbon;

/**
 * Evento disparado cuando cambia el estado de un pedido.
 * 
 * Requerido para auditoría según project_definition.md sección 16.
 */
final readonly class OrderStatusChanged
{
    public function __construct(
        public int $orderId,
        public int $userId,
        public string $oldStatus,
        public string $newStatus,
        public ?string $reason,
        public Carbon $changedAt,
    ) {}
}
```

**Propiedades**:
- `orderId`: ID del pedido
- `userId`: ID del merchant que realizó el cambio
- `oldStatus`: Estado anterior (ej: "new")
- `newStatus`: Nuevo estado (ej: "confirmed")
- `reason`: Razón del cambio (opcional, notas del merchant)
- `changedAt`: Timestamp del cambio

**Uso en Action** (referencia, NO implementar aquí):
```php
// En UpdateOrderStatusAction::execute()
event(new OrderStatusChanged(
    orderId: $order->id,
    userId: auth()->id(),
    oldStatus: $oldStatus->value,
    newStatus: $newStatus->value,
    reason: $data->reason,
    changedAt: now(),
));
```

---

### 3. Evento: PaymentStatusChanged

**Ubicación**: `Modules/Orders/app/Events/PaymentStatusChanged.php`

**Descripción**: Se dispara cuando cambia el estado de pago de un pedido. **Crítico para auditoría** según project_definition.md sección 16.

**Cuándo se dispara**: 
- Desde `UpdatePaymentStatusAction` (cambio manual)
- Desde `ProcessPaymentWebhookAction` (webhook de Mercado Pago)

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Events;

use Carbon\Carbon;

/**
 * Evento disparado cuando cambia el estado de pago de un pedido.
 * 
 * Requerido para auditoría según project_definition.md sección 16.
 */
final readonly class PaymentStatusChanged
{
    public function __construct(
        public int $orderId,
        public ?int $userId,
        public string $oldStatus,
        public string $newStatus,
        public string $source,
        public ?string $transactionId,
        public Carbon $changedAt,
    ) {}
}
```

**Propiedades**:
- `orderId`: ID del pedido
- `userId`: ID del merchant (null si cambio automático vía webhook)
- `oldStatus`: Estado anterior (ej: "pending")
- `newStatus`: Nuevo estado (ej: "paid")
- `source`: Origen del cambio ("manual" | "webhook")
- `transactionId`: ID de transacción de Mercado Pago (null si manual)
- `changedAt`: Timestamp del cambio

**Uso en Action** (referencia, NO implementar aquí):
```php
// Cambio manual
event(new PaymentStatusChanged(
    orderId: $order->id,
    userId: auth()->id(),
    oldStatus: $oldStatus->value,
    newStatus: $newStatus->value,
    source: 'manual',
    transactionId: null,
    changedAt: now(),
));

// Cambio vía webhook
event(new PaymentStatusChanged(
    orderId: $order->id,
    userId: null,
    oldStatus: $oldStatus->value,
    newStatus: $newStatus->value,
    source: 'webhook',
    transactionId: $webhookData->transaction_id,
    changedAt: now(),
));
```

---

### 4. Evento: OrderItemStockReserved

**Ubicación**: `Modules/Orders/app/Events/OrderItemStockReserved.php`

**Descripción**: Se dispara cuando se reserva stock para un item de pedido. Permite sincronización con módulo Catalog.

**Cuándo se dispara**: Desde `CreateOrderAction` después de descontar stock transaccionalmente.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Events;

use Carbon\Carbon;

/**
 * Evento disparado cuando se reserva stock para un item de pedido.
 * 
 * Permite sincronización entre Orders y Catalog modules.
 */
final readonly class OrderItemStockReserved
{
    public function __construct(
        public int $orderId,
        public int $orderItemId,
        public int $productId,
        public ?int $productVariantId,
        public int $quantityReserved,
        public Carbon $reservedAt,
    ) {}
}
```

**Propiedades**:
- `orderId`: ID del pedido
- `orderItemId`: ID del item del pedido
- `productId`: ID del producto
- `productVariantId`: ID de la variante (null si no tiene variantes)
- `quantityReserved`: Cantidad reservada
- `reservedAt`: Timestamp de la reserva

**Uso en Action** (referencia, NO implementar aquí):
```php
// En CreateOrderAction::execute()
// Después de crear cada OrderItem y descontar stock
foreach ($order->items as $item) {
    event(new OrderItemStockReserved(
        orderId: $order->id,
        orderItemId: $item->id,
        productId: $item->product_id,
        productVariantId: $item->product_variant_id,
        quantityReserved: $item->quantity,
        reservedAt: now(),
    ));
}
```

---

## Reglas de Implementación

### Características Obligatorias

✅ **Clase final y readonly**:
```php
final readonly class OrderCreated
```

✅ **Solo constructor con propiedades promovidas**:
```php
public function __construct(
    public int $orderId,
    public string $customerName,
) {}
```

✅ **Tipos primitivos o Carbon para fechas**:
```php
public int $orderId, // ✅ Primitivo
public Carbon $createdAt, // ✅ Carbon para fechas
```

**Nota importante**: En eventos **NO usar Value Objects complejos** porque:
- Los eventos se serializan para queues y logs
- Value Objects pueden complicar la serialización
- Los eventos deben ser lo más simples posible
- Se extraen valores primitivos de VOs antes de crear el evento

✅ **Sin lógica de negocio**:
```php
// ❌ NO hacer esto
public function shouldNotify(): bool { ... }

// ✅ Eventos solo datos
```

✅ **Docblock descriptivo**:
```php
/**
 * Evento disparado cuando se crea un pedido exitosamente.
 */
```

---

### ❌ Lo que NO debe tener un Evento

❌ **Métodos públicos** (excepto constructor)
❌ **Lógica de negocio**
❌ **Validaciones**
❌ **Acceso a base de datos**
❌ **Arrays públicos** (usar propiedades tipadas)
❌ **Propiedades mutables**
❌ **Value Objects complejos** (extraer valores primitivos)

---

## Tests Unitarios

### Ubicación de Tests

```
Modules/Orders/tests/Unit/Events/
├── OrderCreatedTest.php
├── OrderStatusChangedTest.php
├── PaymentStatusChangedTest.php
└── OrderItemStockReservedTest.php
```

---

### Test: OrderCreatedTest

**Ubicación**: `Modules/Orders/tests/Unit/Events/OrderCreatedTest.php`

```php
<?php

declare(strict_types=1);

use Carbon\Carbon;
use Modules\Orders\Events\OrderCreated;

it('crea el evento OrderCreated con datos correctos', function () {
    $orderId = 1;
    $customerName = 'Juan Pérez';
    $customerPhone = '+541112345678';
    $totalAmountCents = 150000;
    $orderStatus = 'new';
    $paymentStatus = 'pending';
    $paymentMethod = 'mercado_pago';
    $itemsCount = 3;
    $createdAt = Carbon::parse('2025-12-19 10:00:00');

    $event = new OrderCreated(
        orderId: $orderId,
        customerName: $customerName,
        customerPhone: $customerPhone,
        totalAmountCents: $totalAmountCents,
        orderStatus: $orderStatus,
        paymentStatus: $paymentStatus,
        paymentMethod: $paymentMethod,
        itemsCount: $itemsCount,
        createdAt: $createdAt,
    );

    expect($event->orderId)->toBe($orderId)
        ->and($event->customerName)->toBe($customerName)
        ->and($event->customerPhone)->toBe($customerPhone)
        ->and($event->totalAmountCents)->toBe($totalAmountCents)
        ->and($event->orderStatus)->toBe($orderStatus)
        ->and($event->paymentStatus)->toBe($paymentStatus)
        ->and($event->paymentMethod)->toBe($paymentMethod)
        ->and($event->itemsCount)->toBe($itemsCount)
        ->and($event->createdAt)->toEqual($createdAt);
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new OrderCreated(
        orderId: 1,
        customerName: 'Test',
        customerPhone: '+541112345678',
        totalAmountCents: 10000,
        orderStatus: 'new',
        paymentStatus: 'pending',
        paymentMethod: 'cash',
        itemsCount: 1,
        createdAt: now(),
    );

    $event->orderId = 999;
})->throws(Error::class);
```

---

### Test: OrderStatusChangedTest

**Ubicación**: `Modules/Orders/tests/Unit/Events/OrderStatusChangedTest.php`

```php
<?php

declare(strict_types=1);

use Carbon\Carbon;
use Modules\Orders\Events\OrderStatusChanged;

it('crea el evento OrderStatusChanged con datos correctos', function () {
    $orderId = 5;
    $userId = 10;
    $oldStatus = 'new';
    $newStatus = 'confirmed';
    $reason = 'Cliente confirmó por teléfono';
    $changedAt = Carbon::parse('2025-12-19 11:00:00');

    $event = new OrderStatusChanged(
        orderId: $orderId,
        userId: $userId,
        oldStatus: $oldStatus,
        newStatus: $newStatus,
        reason: $reason,
        changedAt: $changedAt,
    );

    expect($event->orderId)->toBe($orderId)
        ->and($event->userId)->toBe($userId)
        ->and($event->oldStatus)->toBe($oldStatus)
        ->and($event->newStatus)->toBe($newStatus)
        ->and($event->reason)->toBe($reason)
        ->and($event->changedAt)->toEqual($changedAt);
});

it('permite reason como null', function () {
    $event = new OrderStatusChanged(
        orderId: 1,
        userId: 1,
        oldStatus: 'new',
        newStatus: 'confirmed',
        reason: null,
        changedAt: now(),
    );

    expect($event->reason)->toBeNull();
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new OrderStatusChanged(
        orderId: 1,
        userId: 1,
        oldStatus: 'new',
        newStatus: 'confirmed',
        reason: null,
        changedAt: now(),
    );

    $event->oldStatus = 'cancelled';
})->throws(Error::class);
```

---

### Test: PaymentStatusChangedTest

**Ubicación**: `Modules/Orders/tests/Unit/Events/PaymentStatusChangedTest.php`

```php
<?php

declare(strict_types=1);

use Carbon\Carbon;
use Modules\Orders\Events\PaymentStatusChanged;

it('crea el evento PaymentStatusChanged con cambio manual', function () {
    $orderId = 8;
    $userId = 15;
    $oldStatus = 'pending';
    $newStatus = 'paid';
    $source = 'manual';
    $changedAt = Carbon::parse('2025-12-19 12:00:00');

    $event = new PaymentStatusChanged(
        orderId: $orderId,
        userId: $userId,
        oldStatus: $oldStatus,
        newStatus: $newStatus,
        source: $source,
        transactionId: null,
        changedAt: $changedAt,
    );

    expect($event->orderId)->toBe($orderId)
        ->and($event->userId)->toBe($userId)
        ->and($event->oldStatus)->toBe($oldStatus)
        ->and($event->newStatus)->toBe($newStatus)
        ->and($event->source)->toBe($source)
        ->and($event->transactionId)->toBeNull()
        ->and($event->changedAt)->toEqual($changedAt);
});

it('crea el evento PaymentStatusChanged con cambio vía webhook', function () {
    $event = new PaymentStatusChanged(
        orderId: 10,
        userId: null,
        oldStatus: 'pending',
        newStatus: 'paid',
        source: 'webhook',
        transactionId: 'MP-123456789',
        changedAt: now(),
    );

    expect($event->userId)->toBeNull()
        ->and($event->source)->toBe('webhook')
        ->and($event->transactionId)->toBe('MP-123456789');
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new PaymentStatusChanged(
        orderId: 1,
        userId: 1,
        oldStatus: 'pending',
        newStatus: 'paid',
        source: 'manual',
        transactionId: null,
        changedAt: now(),
    );

    $event->newStatus = 'refunded';
})->throws(Error::class);
```

---

### Test: OrderItemStockReservedTest

**Ubicación**: `Modules/Orders/tests/Unit/Events/OrderItemStockReservedTest.php`

```php
<?php

declare(strict_types=1);

use Carbon\Carbon;
use Modules\Orders\Events\OrderItemStockReserved;

it('crea el evento OrderItemStockReserved con datos correctos', function () {
    $orderId = 12;
    $orderItemId = 45;
    $productId = 7;
    $productVariantId = 23;
    $quantityReserved = 2;
    $reservedAt = Carbon::parse('2025-12-19 13:00:00');

    $event = new OrderItemStockReserved(
        orderId: $orderId,
        orderItemId: $orderItemId,
        productId: $productId,
        productVariantId: $productVariantId,
        quantityReserved: $quantityReserved,
        reservedAt: $reservedAt,
    );

    expect($event->orderId)->toBe($orderId)
        ->and($event->orderItemId)->toBe($orderItemId)
        ->and($event->productId)->toBe($productId)
        ->and($event->productVariantId)->toBe($productVariantId)
        ->and($event->quantityReserved)->toBe($quantityReserved)
        ->and($event->reservedAt)->toEqual($reservedAt);
});

it('permite productVariantId como null cuando no hay variantes', function () {
    $event = new OrderItemStockReserved(
        orderId: 1,
        orderItemId: 1,
        productId: 1,
        productVariantId: null,
        quantityReserved: 1,
        reservedAt: now(),
    );

    expect($event->productVariantId)->toBeNull();
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new OrderItemStockReserved(
        orderId: 1,
        orderItemId: 1,
        productId: 1,
        productVariantId: null,
        quantityReserved: 1,
        reservedAt: now(),
    );

    $event->quantityReserved = 999;
})->throws(Error::class);
```

---

## Validación de Calidad

### Checklist Obligatorio

- [ ] **PHPStan level 6** sin errores
  ```bash
  ./vendor/bin/sail composer run phpstan
  ```

- [ ] **Pint** ejecutado
  ```bash
  ./vendor/bin/sail bin pint --dirty
  ```

- [ ] **Tests unitarios verdes**
  ```bash
  ./vendor/bin/sail test Modules/Orders/Tests/Unit/Events
  ```

- [ ] **Eventos son `final readonly`**
- [ ] **Solo contienen constructor**
- [ ] **Usan tipos primitivos y Carbon**
- [ ] **Sin lógica de negocio**
- [ ] **Docblocks completos con referencia a auditoría**

---

## Estructura de Archivos Final

```
Modules/Orders/app/Events/
├── OrderCreated.php ✅
├── OrderStatusChanged.php ✅
├── PaymentStatusChanged.php ✅
└── OrderItemStockReserved.php ✅

Modules/Orders/tests/Unit/Events/
├── OrderCreatedTest.php ✅
├── OrderStatusChangedTest.php ✅
├── PaymentStatusChangedTest.php ✅
└── OrderItemStockReservedTest.php ✅
```

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Los 4 eventos están creados con tipado fuerte
2. ✅ Todos usan `final readonly class`
3. ✅ Usan tipos primitivos y Carbon (no Value Objects complejos)
4. ✅ Sin lógica de negocio
5. ✅ Tests unitarios verdes (100% cobertura)
6. ✅ PHPStan level 6 sin errores
7. ✅ Pint ejecutado sin cambios pendientes
8. ✅ Docblocks referencian auditoría donde aplique

---

## Notas Importantes

### ⚠️ El Agente E NO dispara eventos

Los eventos se disparan desde las **Actions del Agente B**:
- `OrderCreated` → desde `CreateOrderAction`
- `OrderStatusChanged` → desde `UpdateOrderStatusAction`
- `PaymentStatusChanged` → desde `UpdatePaymentStatusAction` o `ProcessPaymentWebhookAction`
- `OrderItemStockReserved` → desde `CreateOrderAction` (por cada item)

### 🎯 Los eventos solo contienen datos

- No contienen lógica
- No validan nada
- No deciden nada
- Solo transportan información

### 📝 Los eventos son inmutables

- `final readonly class`
- Sin setters
- Sin métodos públicos (excepto constructor)

### 🔒 Crítico para Auditoría

Según `project_definition.md` sección 16, los eventos `OrderStatusChanged` y `PaymentStatusChanged` son **obligatorios** para cumplir con requisitos de auditoría y trazabilidad.

### 🚫 No usar Value Objects en eventos

Los eventos usan **tipos primitivos** extraídos de Value Objects:
```php
// ❌ NO hacer esto
public Money $totalAmount, // Value Object complejo

// ✅ Hacer esto
public int $totalAmountCents, // Primitivo extraído del VO
```

**Razones**:
- Simplicidad en serialización
- Compatibilidad con queues y logs
- Eventos deben ser lo más ligeros posible
- Se extraen valores antes de crear el evento

---

## Referencias

- **Agente E (metodología)**: `laravel/agents/agente-e.md`
- **Project Definition**: `e-commerce-wa-ml/project_definition.md` (sección 16: Auditoría)
- **Orders Class Diagram**: `e-commerce-wa-ml/orders/orders-class-diagram.md`
- **Laravel Events**: https://laravel.com/docs/events

---

## Próximos Pasos

Después de completar esta tarea, crear:
- **agente-e-task-06**: Listeners para eventos de Orders (auditoría)
- **agente-e-task-07**: Integración de eventos en Actions de Orders

---

**Última actualización**: 2025-12-19  
**Autor**: Alejandro Leone
