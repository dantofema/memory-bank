---
task_id: "agente-e-task-06"
module: "Orders"
agent: "Agente E"
title: "Listeners de Auditoría para Módulo Orders"
priority: "high"
estimated_time: "2.5h"
dependencies:
  - "agente-e-task-05: Eventos de Orders creados"
  - "Agente B: Actions de Orders implementadas"
  - "Agente C: Repositories de Orders disponibles"
status: "pending"
---

# Task: Listeners de Auditoría para Módulo Orders

## Contexto

Crear listeners que reaccionen a los eventos del módulo Orders para implementar **auditoría obligatoria** según `project_definition.md` sección 16. Los listeners NO contienen lógica de negocio, solo **reaccionan** a eventos y **delegan trabajo** a Repositories o Jobs.

**Casos de uso críticos**:
- Registrar cambios de estado de pedidos (OrderStatus y PaymentStatus)
- Auditar quién, cuándo y por qué cambió un estado
- Logging de creación de pedidos para análisis
- Sincronización de stock entre módulos

---

## Objetivos

1. ✅ Crear listener `LogOrderCreation`
2. ✅ Crear listener `AuditOrderStatusChange`
3. ✅ Crear listener `AuditPaymentStatusChange`
4. ✅ Crear listener `LogStockReservation`
5. ✅ Registrar listeners en `EventServiceProvider`
6. ✅ Tests de listeners con Event fake
7. ✅ Validar PHPStan level 6

---

## Listeners a Crear

### 1. Listener: LogOrderCreation

**Ubicación**: `Modules/Orders/app/Listeners/LogOrderCreation.php`

**Descripción**: Registra en logs cada pedido creado para análisis y debugging.

**Reacciona a**: `OrderCreated`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderCreated;

/**
 * Registra en logs la creación de pedidos.
 */
final readonly class LogOrderCreation
{
    public function handle(OrderCreated $event): void
    {
        Log::info('Order created successfully', [
            'order_id' => $event->orderId,
            'customer_name' => $event->customerName,
            'customer_phone' => $event->customerPhone,
            'total_amount_cents' => $event->totalAmountCents,
            'order_status' => $event->orderStatus,
            'payment_status' => $event->paymentStatus,
            'payment_method' => $event->paymentMethod,
            'items_count' => $event->itemsCount,
            'created_at' => $event->createdAt->toIso8601String(),
            'context' => 'order_tracking',
        ]);
    }
}
```

**Características**:
- ✅ `final readonly class`
- ✅ Un solo método público: `handle()`
- ✅ Tipado estricto del evento
- ✅ No contiene lógica de negocio
- ✅ Contexto: `order_tracking`

---

### 2. Listener: AuditOrderStatusChange

**Ubicación**: `Modules/Orders/app/Listeners/AuditOrderStatusChange.php`

**Descripción**: **Auditoría obligatoria** de cambios de estado de pedidos según `project_definition.md` sección 16. Registra en tabla `order_status_logs` y en logs de aplicación.

**Reacciona a**: `OrderStatusChanged`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderStatusChanged;
use Modules\Orders\Repositories\OrderStatusLogRepository;

/**
 * Audita cambios de estado de pedidos.
 * 
 * Requerido para cumplir con project_definition.md sección 16.
 */
final readonly class AuditOrderStatusChange
{
    public function __construct(
        private OrderStatusLogRepository $logRepository,
    ) {}

    public function handle(OrderStatusChanged $event): void
    {
        // Guardar en tabla de auditoría
        $this->logRepository->create([
            'order_id' => $event->orderId,
            'user_id' => $event->userId,
            'field' => 'order_status',
            'old_value' => $event->oldStatus,
            'new_value' => $event->newStatus,
            'reason' => $event->reason,
            'created_at' => $event->changedAt,
        ]);

        // Registrar en logs de aplicación
        Log::info('Order status changed', [
            'order_id' => $event->orderId,
            'user_id' => $event->userId,
            'old_status' => $event->oldStatus,
            'new_status' => $event->newStatus,
            'reason' => $event->reason,
            'changed_at' => $event->changedAt->toIso8601String(),
            'context' => 'order_audit',
        ]);
    }
}
```

**Características**:
- ✅ Delega a `OrderStatusLogRepository` para persistencia
- ✅ Registra en logs para debugging
- ✅ Contexto: `order_audit`
- ✅ Cumple requisitos de auditoría del proyecto

---

### 3. Listener: AuditPaymentStatusChange

**Ubicación**: `Modules/Orders/app/Listeners/AuditPaymentStatusChange.php`

**Descripción**: **Auditoría obligatoria** de cambios de estado de pago según `project_definition.md` sección 16. Registra en tabla `order_status_logs` y en logs de aplicación.

**Reacciona a**: `PaymentStatusChanged`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\PaymentStatusChanged;
use Modules\Orders\Repositories\OrderStatusLogRepository;

/**
 * Audita cambios de estado de pago de pedidos.
 * 
 * Requerido para cumplir con project_definition.md sección 16.
 */
final readonly class AuditPaymentStatusChange
{
    public function __construct(
        private OrderStatusLogRepository $logRepository,
    ) {}

    public function handle(PaymentStatusChanged $event): void
    {
        // Guardar en tabla de auditoría
        $this->logRepository->create([
            'order_id' => $event->orderId,
            'user_id' => $event->userId,
            'field' => 'payment_status',
            'old_value' => $event->oldStatus,
            'new_value' => $event->newStatus,
            'reason' => $event->source === 'webhook' 
                ? "Webhook - Transaction: {$event->transactionId}" 
                : 'Manual change',
            'created_at' => $event->changedAt,
        ]);

        // Registrar en logs de aplicación
        Log::info('Payment status changed', [
            'order_id' => $event->orderId,
            'user_id' => $event->userId,
            'old_status' => $event->oldStatus,
            'new_status' => $event->newStatus,
            'source' => $event->source,
            'transaction_id' => $event->transactionId,
            'changed_at' => $event->changedAt->toIso8601String(),
            'context' => 'payment_audit',
        ]);
    }
}
```

**Características**:
- ✅ Delega a `OrderStatusLogRepository` para persistencia
- ✅ Diferencia entre cambios manuales y automáticos (webhook)
- ✅ Contexto: `payment_audit`
- ✅ Cumple requisitos de auditoría del proyecto

---

### 4. Listener: LogStockReservation

**Ubicación**: `Modules/Orders/app/Listeners/LogStockReservation.php`

**Descripción**: Registra en logs las reservas de stock para debugging y análisis de inventario.

**Reacciona a**: `OrderItemStockReserved`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderItemStockReserved;

/**
 * Registra en logs las reservas de stock.
 */
final readonly class LogStockReservation
{
    public function handle(OrderItemStockReserved $event): void
    {
        Log::info('Stock reserved for order item', [
            'order_id' => $event->orderId,
            'order_item_id' => $event->orderItemId,
            'product_id' => $event->productId,
            'product_variant_id' => $event->productVariantId,
            'quantity_reserved' => $event->quantityReserved,
            'reserved_at' => $event->reservedAt->toIso8601String(),
            'context' => 'stock_management',
        ]);
    }
}
```

**Características**:
- ✅ Logging simple sin persistencia adicional
- ✅ Contexto: `stock_management`
- ✅ Facilita debugging de problemas de inventario

---

## Registro de Listeners

### EventServiceProvider del Módulo Orders

**Ubicación**: `Modules/Orders/app/Providers/EventServiceProvider.php`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Modules\Orders\Events\OrderCreated;
use Modules\Orders\Events\OrderItemStockReserved;
use Modules\Orders\Events\OrderStatusChanged;
use Modules\Orders\Events\PaymentStatusChanged;
use Modules\Orders\Listeners\AuditOrderStatusChange;
use Modules\Orders\Listeners\AuditPaymentStatusChange;
use Modules\Orders\Listeners\LogOrderCreation;
use Modules\Orders\Listeners\LogStockReservation;

final class EventServiceProvider extends ServiceProvider
{
    /**
     * @var array<string, array<int, string>>
     */
    protected $listen = [
        OrderCreated::class => [
            LogOrderCreation::class,
        ],
        OrderStatusChanged::class => [
            AuditOrderStatusChange::class,
        ],
        PaymentStatusChanged::class => [
            AuditPaymentStatusChange::class,
        ],
        OrderItemStockReserved::class => [
            LogStockReservation::class,
        ],
    ];

    /**
     * Register any events for your application.
     */
    public function boot(): void
    {
        parent::boot();
    }
}
```

**Registrar en `module.json`**:

Verificar que el `EventServiceProvider` está registrado en el archivo `module.json` del módulo:

```json
{
  "providers": [
    "Modules\\Orders\\Providers\\OrdersServiceProvider",
    "Modules\\Orders\\Providers\\EventServiceProvider"
  ]
}
```

---

## Repository: OrderStatusLogRepository

### Ubicación y Código

**Ubicación**: `Modules/Orders/app/Repositories/OrderStatusLogRepository.php`

**Descripción**: Repository para persistir logs de auditoría de cambios de estado.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Repositories;

use Modules\Orders\Models\OrderStatusLog;

/**
 * Repository para gestión de logs de auditoría de estados.
 */
final readonly class OrderStatusLogRepository
{
    /**
     * Crea un nuevo log de auditoría.
     * 
     * @param array{order_id: int, user_id: ?int, field: string, old_value: string, new_value: string, reason: ?string, created_at: \Carbon\Carbon} $data
     */
    public function create(array $data): OrderStatusLog
    {
        return OrderStatusLog::create($data);
    }

    /**
     * Obtiene logs de auditoría de un pedido.
     * 
     * @return \Illuminate\Support\Collection<int, OrderStatusLog>
     */
    public function getByOrderId(int $orderId): \Illuminate\Support\Collection
    {
        return OrderStatusLog::where('order_id', $orderId)
            ->orderBy('created_at', 'desc')
            ->get();
    }

    /**
     * Obtiene logs de un campo específico de un pedido.
     * 
     * @return \Illuminate\Support\Collection<int, OrderStatusLog>
     */
    public function getByOrderIdAndField(int $orderId, string $field): \Illuminate\Support\Collection
    {
        return OrderStatusLog::where('order_id', $orderId)
            ->where('field', $field)
            ->orderBy('created_at', 'desc')
            ->get();
    }
}
```

---

## Modelo: OrderStatusLog

### Ubicación y Código

**Ubicación**: `Modules/Orders/app/Models/OrderStatusLog.php`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Orders\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

/**
 * Modelo para logs de auditoría de cambios de estado.
 * 
 * @property int $id
 * @property int $order_id
 * @property int|null $user_id
 * @property string $field
 * @property string $old_value
 * @property string $new_value
 * @property string|null $reason
 * @property \Carbon\Carbon $created_at
 */
final class OrderStatusLog extends Model
{
    public const UPDATED_AT = null;

    protected $fillable = [
        'order_id',
        'user_id',
        'field',
        'old_value',
        'new_value',
        'reason',
        'created_at',
    ];

    protected $casts = [
        'created_at' => 'datetime',
    ];

    public function order(): BelongsTo
    {
        return $this->belongsTo(Order::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(\App\Models\User::class);
    }
}
```

---

## Migración: order_status_logs

### Ubicación y Código

**Ubicación**: `Modules/Orders/database/migrations/XXXX_XX_XX_create_order_status_logs_table.php`

**Código**:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('order_status_logs', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
            $table->string('field'); // 'order_status' | 'payment_status'
            $table->string('old_value');
            $table->string('new_value');
            $table->text('reason')->nullable();
            $table->timestamp('created_at');

            // Índices para consultas de auditoría
            $table->index(['order_id', 'created_at']);
            $table->index(['order_id', 'field', 'created_at']);
            $table->index('user_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('order_status_logs');
    }
};
```

---

## Tests de Listeners

### Estrategia de Testing

Los listeners se testean con `Event::fake()` y `Log::spy()` para verificar que:
1. Los listeners se ejecutan cuando se dispara el evento
2. Los listeners registran la información correcta en logs
3. Los listeners de auditoría persisten correctamente en BD

---

### Test: LogOrderCreationTest

**Ubicación**: `Modules/Orders/tests/Feature/Listeners/LogOrderCreationTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderCreated;
use Modules\Orders\Listeners\LogOrderCreation;

it('registra en logs cuando se crea un pedido', function () {
    Log::spy();

    $event = new OrderCreated(
        orderId: 1,
        customerName: 'Juan Pérez',
        customerPhone: '+541112345678',
        totalAmountCents: 150000,
        orderStatus: 'new',
        paymentStatus: 'pending',
        paymentMethod: 'mercado_pago',
        itemsCount: 3,
        createdAt: now(),
    );

    $listener = new LogOrderCreation();
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Order created successfully', \Mockery::on(function ($context) {
            return $context['order_id'] === 1
                && $context['customer_name'] === 'Juan Pérez'
                && $context['customer_phone'] === '+541112345678'
                && $context['total_amount_cents'] === 150000
                && $context['context'] === 'order_tracking';
        }));
});

it('ejecuta el listener cuando se dispara el evento OrderCreated', function () {
    Event::fake([OrderCreated::class]);

    event(new OrderCreated(
        orderId: 1,
        customerName: 'Test',
        customerPhone: '+541112345678',
        totalAmountCents: 10000,
        orderStatus: 'new',
        paymentStatus: 'pending',
        paymentMethod: 'cash',
        itemsCount: 1,
        createdAt: now(),
    ));

    Event::assertDispatched(OrderCreated::class);
});
```

---

### Test: AuditOrderStatusChangeTest

**Ubicación**: `Modules/Orders/tests/Feature/Listeners/AuditOrderStatusChangeTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderStatusChanged;
use Modules\Orders\Listeners\AuditOrderStatusChange;
use Modules\Orders\Models\OrderStatusLog;

it('audita cambio de estado de pedido en base de datos', function () {
    $event = new OrderStatusChanged(
        orderId: 5,
        userId: 10,
        oldStatus: 'new',
        newStatus: 'confirmed',
        reason: 'Cliente confirmó',
        changedAt: now(),
    );

    $listener = app(AuditOrderStatusChange::class);
    $listener->handle($event);

    // Verificar que se guardó en BD
    expect(OrderStatusLog::count())->toBe(1);

    $log = OrderStatusLog::first();
    expect($log->order_id)->toBe(5)
        ->and($log->user_id)->toBe(10)
        ->and($log->field)->toBe('order_status')
        ->and($log->old_value)->toBe('new')
        ->and($log->new_value)->toBe('confirmed')
        ->and($log->reason)->toBe('Cliente confirmó');
});

it('registra en logs cuando cambia estado de pedido', function () {
    Log::spy();

    $event = new OrderStatusChanged(
        orderId: 5,
        userId: 10,
        oldStatus: 'new',
        newStatus: 'confirmed',
        reason: null,
        changedAt: now(),
    );

    $listener = app(AuditOrderStatusChange::class);
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Order status changed', \Mockery::on(function ($context) {
            return $context['order_id'] === 5
                && $context['user_id'] === 10
                && $context['old_status'] === 'new'
                && $context['new_status'] === 'confirmed'
                && $context['context'] === 'order_audit';
        }));
});
```

---

### Test: AuditPaymentStatusChangeTest

**Ubicación**: `Modules/Orders/tests/Feature/Listeners/AuditPaymentStatusChangeTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\PaymentStatusChanged;
use Modules\Orders\Listeners\AuditPaymentStatusChange;
use Modules\Orders\Models\OrderStatusLog;

it('audita cambio manual de estado de pago', function () {
    $event = new PaymentStatusChanged(
        orderId: 8,
        userId: 15,
        oldStatus: 'pending',
        newStatus: 'paid',
        source: 'manual',
        transactionId: null,
        changedAt: now(),
    );

    $listener = app(AuditPaymentStatusChange::class);
    $listener->handle($event);

    $log = OrderStatusLog::first();
    expect($log->order_id)->toBe(8)
        ->and($log->user_id)->toBe(15)
        ->and($log->field)->toBe('payment_status')
        ->and($log->old_value)->toBe('pending')
        ->and($log->new_value)->toBe('paid')
        ->and($log->reason)->toBe('Manual change');
});

it('audita cambio automático de estado de pago vía webhook', function () {
    $event = new PaymentStatusChanged(
        orderId: 10,
        userId: null,
        oldStatus: 'pending',
        newStatus: 'paid',
        source: 'webhook',
        transactionId: 'MP-123456789',
        changedAt: now(),
    );

    $listener = app(AuditPaymentStatusChange::class);
    $listener->handle($event);

    $log = OrderStatusLog::first();
    expect($log->user_id)->toBeNull()
        ->and($log->reason)->toContain('Webhook')
        ->and($log->reason)->toContain('MP-123456789');
});

it('registra en logs cuando cambia estado de pago', function () {
    Log::spy();

    $event = new PaymentStatusChanged(
        orderId: 8,
        userId: 15,
        oldStatus: 'pending',
        newStatus: 'paid',
        source: 'manual',
        transactionId: null,
        changedAt: now(),
    );

    $listener = app(AuditPaymentStatusChange::class);
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Payment status changed', \Mockery::on(function ($context) {
            return $context['order_id'] === 8
                && $context['context'] === 'payment_audit';
        }));
});
```

---

### Test: LogStockReservationTest

**Ubicación**: `Modules/Orders/tests/Feature/Listeners/LogStockReservationTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Log;
use Modules\Orders\Events\OrderItemStockReserved;
use Modules\Orders\Listeners\LogStockReservation;

it('registra en logs cuando se reserva stock', function () {
    Log::spy();

    $event = new OrderItemStockReserved(
        orderId: 12,
        orderItemId: 45,
        productId: 7,
        productVariantId: 23,
        quantityReserved: 2,
        reservedAt: now(),
    );

    $listener = new LogStockReservation();
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Stock reserved for order item', \Mockery::on(function ($context) {
            return $context['order_id'] === 12
                && $context['product_id'] === 7
                && $context['quantity_reserved'] === 2
                && $context['context'] === 'stock_management';
        }));
});

it('maneja correctamente productos sin variantes', function () {
    Log::spy();

    $event = new OrderItemStockReserved(
        orderId: 1,
        orderItemId: 1,
        productId: 1,
        productVariantId: null,
        quantityReserved: 1,
        reservedAt: now(),
    );

    $listener = new LogStockReservation();
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Stock reserved for order item', \Mockery::on(function ($context) {
            return $context['product_variant_id'] === null;
        }));
});
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

- [ ] **Tests de listeners verdes**
  ```bash
  ./vendor/bin/sail test Modules/Orders/Tests/Feature/Listeners
  ```

- [ ] **Migración ejecutada correctamente**
  ```bash
  ./vendor/bin/sail artisan migrate
  ```

- [ ] **Listeners registrados correctamente**
  ```bash
  ./vendor/bin/sail artisan event:list | grep Orders
  ```

  **Salida esperada**:
  ```
  Modules\Orders\Events\OrderCreated
    Modules\Orders\Listeners\LogOrderCreation
  
  Modules\Orders\Events\OrderStatusChanged
    Modules\Orders\Listeners\AuditOrderStatusChange
  
  Modules\Orders\Events\PaymentStatusChanged
    Modules\Orders\Listeners\AuditPaymentStatusChange
  
  Modules\Orders\Events\OrderItemStockReserved
    Modules\Orders\Listeners\LogStockReservation
  ```

---

## Estructura de Archivos Final

```
Modules/Orders/app/
├── Events/
│   ├── OrderCreated.php
│   ├── OrderStatusChanged.php
│   ├── PaymentStatusChanged.php
│   └── OrderItemStockReserved.php
├── Listeners/
│   ├── LogOrderCreation.php ✅
│   ├── AuditOrderStatusChange.php ✅
│   ├── AuditPaymentStatusChange.php ✅
│   └── LogStockReservation.php ✅
├── Models/
│   └── OrderStatusLog.php ✅
├── Repositories/
│   └── OrderStatusLogRepository.php ✅
└── Providers/
    └── EventServiceProvider.php ✅

Modules/Orders/database/migrations/
└── XXXX_XX_XX_create_order_status_logs_table.php ✅

Modules/Orders/tests/Feature/Listeners/
├── LogOrderCreationTest.php ✅
├── AuditOrderStatusChangeTest.php ✅
├── AuditPaymentStatusChangeTest.php ✅
└── LogStockReservationTest.php ✅
```

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Los 4 listeners están creados con tipado fuerte
2. ✅ Todos son `final readonly class`
3. ✅ Solo tienen método `handle()`
4. ✅ Sin lógica de negocio
5. ✅ EventServiceProvider registrado correctamente
6. ✅ Tabla `order_status_logs` creada con migración
7. ✅ Repository y Modelo creados
8. ✅ Tests de listeners verdes (100% cobertura)
9. ✅ PHPStan level 6 sin errores
10. ✅ `event:list` muestra los listeners registrados

---

## Notas Importantes

### 🎯 Auditoría Obligatoria

Según `project_definition.md` sección 16, los listeners `AuditOrderStatusChange` y `AuditPaymentStatusChange` son **obligatorios** para cumplir con:
- Trazabilidad de cambios de estado
- Compliance y auditorías externas
- Debugging de problemas de pedidos

### 📊 Contextos de Logging

- `order_tracking`: logs de creación y tracking de pedidos
- `order_audit`: logs de auditoría de cambios de estado
- `payment_audit`: logs de auditoría de cambios de pago
- `stock_management`: logs de gestión de inventario

Estos contextos facilitan filtrado en sistemas de logs (ELK, CloudWatch, etc.).

### 🔄 Listeners Síncronos en MVP

En MVP, los listeners se ejecutan **síncronamente**. Si el volumen de pedidos crece, considerar convertir a Jobs asíncronos.

### 📝 Tabla de Auditoría

La tabla `order_status_logs` es append-only (solo inserts), nunca se actualiza ni elimina. Esto garantiza integridad de la auditoría.

---

## Referencias

- **agente-e-task-05**: Eventos del módulo Orders
- **Project Definition**: `e-commerce-wa-ml/project_definition.md` (sección 16)
- **Laravel Events**: https://laravel.com/docs/events
- **Laravel Logging**: https://laravel.com/docs/logging

---

**Última actualización**: 2025-12-19  
**Autor**: Alejandro Leone
