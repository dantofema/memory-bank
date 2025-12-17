---
name: "Module Communication Patterns"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "Define communication patterns between Laravel Modules using Interfaces and Domain Events"
context:
  architecture: "nwidart/laravel-modules"
  pattern: "modular-monolith"
  coupling: "loose-coupling via contracts"
tags: ["architecture", "modules", "events", "interfaces", "ddd"]
---

# Patrones de Comunicación entre Módulos

## Resumen

Define dos mecanismos complementarios para comunicación entre módulos en arquitectura modular: **Interfaces** (para comunicación síncrona con respuesta) y **Eventos de Dominio** (para notificaciones asíncronas sin respuesta).

**Principio fundamental**: Las decisiones de negocio se toman mediante interfaces; los eventos solo notifican hechos pasados.

---

## Mecanismos de Comunicación

### 1. Interfaces (Commands / Queries)

**Propósito**: Comunicación síncrona cuando se necesita una **respuesta inmediata y determinista**.

#### Características

- ✅ Representan **capacidades**, **validaciones** o **decisiones** del negocio
- ✅ Son **sincrónicas**, **fuertemente tipadas** y **explícitas**
- ✅ **No exponen modelos internos** ni detalles de persistencia
- ✅ Fallan de forma inmediata y propagan el error al llamador
- ✅ Retornan siempre **Data Objects** o **Value Objects**, nunca modelos Eloquent

#### Cuándo usar

- Verificar disponibilidad de stock antes de confirmar una orden
- Validar si un usuario tiene permisos para una acción
- Obtener el precio actualizado de un producto
- Aplicar reglas de negocio que afectan el flujo de ejecución

#### Ejemplo: Verificar Stock

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Contracts;

use App\ValueObjects\ProductId;
use App\ValueObjects\Quantity;
use Modules\Catalog\Data\AvailabilityResultData;

interface CheckProductAvailabilityInterface
{
    /**
     * Verifica si hay stock disponible para un producto.
     *
     * @throws ProductNotFoundException si el producto no existe
     * @throws StockNotAvailableException si no hay stock suficiente
     */
    public function check(ProductId $productId, Quantity $quantity): AvailabilityResultData;
}
```

```php
<?php

declare(strict_types=1);

namespace Modules\Order\Services;

use Modules\Catalog\Contracts\CheckProductAvailabilityInterface;
use Modules\Order\Data\CreateOrderData;
use Modules\Order\Exceptions\InsufficientStockException;

final readonly class CreateOrderService
{
    public function __construct(
        private CheckProductAvailabilityInterface $checkAvailability,
    ) {}

    public function execute(CreateOrderData $data): OrderData
    {
        // Decisión de negocio: verificar stock antes de crear orden
        $availability = $this->checkAvailability->check(
            $data->productId,
            $data->quantity
        );

        if (! $availability->isAvailable) {
            throw new InsufficientStockException();
        }

        // Continúa con la creación de la orden...
    }
}
```

---

### 2. Eventos de Dominio

**Propósito**: Notificar que **un hecho relevante del negocio ya ocurrió**.

#### Características

- ✅ Son señales **inmutables**, **unidireccionales** y **sin valor de retorno**
- ✅ **No influyen en decisiones de negocio** ni en el control del flujo
- ✅ Habilitan **efectos secundarios**: notificaciones, integraciones, reportes, auditoría
- ✅ Los consumidores deben estar **desacoplados**, ser **idempotentes** y **tolerar duplicados**
- ✅ **No incluyen datos sensibles** (passwords, tokens, tarjetas de crédito)

#### Cuándo usar

- Notificar que una orden fue confirmada (para enviar email de confirmación)
- Registrar auditoría cuando un precio cambió
- Actualizar estadísticas después de completar una venta
- Sincronizar datos con sistemas externos

#### Ejemplo: Orden Confirmada

```php
<?php

declare(strict_types=1);

namespace Modules\Order\Events;

use App\ValueObjects\OrderId;
use Carbon\Carbon;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

final readonly class OrderConfirmedEvent
{
    use Dispatchable;
    use SerializesModels;

    public function __construct(
        public OrderId $orderId,
        public Carbon $confirmedAt,
    ) {}
}
```

```php
<?php

declare(strict_types=1);

namespace Modules\Notification\Listeners;

use Modules\Order\Events\OrderConfirmedEvent;
use Modules\Notification\Services\SendOrderConfirmationEmailService;

final readonly class SendOrderConfirmationEmailListener
{
    public function __construct(
        private SendOrderConfirmationEmailService $sendEmail,
    ) {}

    /**
     * @param OrderConfirmedEvent $event
     */
    public function handle(object $event): void
    {
        // Efecto secundario: enviar email
        // No afecta el flujo de negocio de Order
        $this->sendEmail->execute($event->orderId);
    }
}
```

---

## Tabla Comparativa

| Aspecto | Interfaces (Commands/Queries) | Eventos de Dominio |
|---------|-------------------------------|-------------------|
| **Propósito** | Tomar decisiones de negocio | Notificar hechos pasados |
| **Sincronía** | Síncrona (bloquea hasta obtener respuesta) | Asíncrona (fire-and-forget) |
| **Retorno** | Sí (Data/Value Objects) | No (void) |
| **Dirección** | Bidireccional (request/response) | Unidireccional (solo envío) |
| **Fallo** | Propaga error al llamador | Consumidor maneja error aisladamente |
| **Acoplamiento** | Acoplamiento explícito (dependencia) | Desacoplado (mediado por event bus) |
| **Ejemplo** | `CheckProductAvailability` | `OrderConfirmedEvent` |
| **Uso en flujo** | Afecta decisiones (if/else, validaciones) | Solo efectos secundarios |

---

## Reglas Obligatorias

1. ✅ **El estado del negocio se decide mediante interfaces, nunca mediante eventos**
   - ❌ Incorrecto: Emitir `StockVerifiedEvent` y esperar que un listener decida si crear la orden
   - ✅ Correcto: Llamar a `CheckProductAvailabilityInterface` y decidir basado en el resultado

2. ✅ **Los eventos comunican hechos del pasado; las interfaces gobiernan el presente**
   - Eventos: `OrderConfirmed`, `PaymentProcessed`, `StockDecremented` (pasado)
   - Interfaces: `CheckAvailability`, `ProcessPayment`, `ReserveStock` (presente/imperativo)

3. ✅ **Los módulos core emiten eventos; los módulos periféricos los consumen**
   - Core: `Order`, `Catalog`, `Payment` (emiten eventos)
   - Periféricos: `Notification`, `Analytics`, `Reporting` (consumen eventos)

4. ✅ **Los eventos no deben incluir datos sensibles**
   - ❌ Incluir: passwords, tokens de API, datos completos de tarjetas de crédito
   - ✅ Incluir: IDs, timestamps, estados, cantidades, montos

5. ✅ **Interfaces nunca devuelven modelos Eloquent**
   - Usar Spatie Laravel Data o Value Objects
   - Mantener la encapsulación del módulo

---

## Ejemplo Completo: Flujo de Crear Orden

### Paso 1: Verificar Stock (Interface - Síncrona)

```php
// Módulo Order llama a Catalog para tomar decisión
$availability = $this->checkAvailability->check($productId, $quantity);

if (! $availability->isAvailable) {
    throw new InsufficientStockException();
}
```

### Paso 2: Crear Orden

```php
$order = Order::create([...]);
```

### Paso 3: Emitir Evento (Notificación - Asíncrona)

```php
// Notificar que la orden fue creada
OrderConfirmedEvent::dispatch($order->id, now());
```

### Paso 4: Consumir Evento (Efectos Secundarios)

```php
// Listener 1: Enviar email de confirmación
class SendConfirmationEmailListener
{
    public function handle(OrderConfirmedEvent $event): void
    {
        Mail::to($order->customer)->send(new OrderConfirmationMail($event->orderId));
    }
}

// Listener 2: Actualizar analytics
class UpdateAnalyticsListener
{
    public function handle(OrderConfirmedEvent $event): void
    {
        Analytics::recordOrderConfirmed($event->orderId);
    }
}
```

---

## Organización de Archivos

```
Modules/
├── Catalog/
│   ├── Contracts/
│   │   └── CheckProductAvailabilityInterface.php
│   ├── Services/
│   │   └── CheckProductAvailabilityService.php  # Implementación
│   └── Providers/
│       └── CatalogServiceProvider.php  # Bind interface → service
├── Order/
│   ├── Events/
│   │   └── OrderConfirmedEvent.php
│   ├── Services/
│   │   └── CreateOrderService.php  # Usa interface de Catalog
│   └── Listeners/
└── Notification/
    ├── Listeners/
    │   └── SendOrderConfirmationEmailListener.php  # Consume evento
    └── Services/
        └── SendOrderConfirmationEmailService.php
```

---

## Antipatrones a Evitar

### ❌ Antipatrón 1: Tomar Decisiones Basadas en Eventos

```php
// ❌ INCORRECTO: Listener intenta afectar el flujo de negocio
class ValidateStockListener
{
    public function handle(OrderCreatingEvent $event): void
    {
        // No se puede detener la creación de la orden desde aquí
        if (! $this->hasStock($event->productId)) {
            throw new Exception('No hay stock'); // Esto no funciona bien
        }
    }
}
```

✅ **Correcto**: Usar interface antes de crear la orden.

### ❌ Antipatrón 2: Exponer Modelos Eloquent en Interfaces

```php
// ❌ INCORRECTO
interface GetProductInterface
{
    public function get(int $id): Product; // Modelo Eloquent expuesto
}
```

✅ **Correcto**: Retornar Data Object.

```php
// ✅ CORRECTO
interface GetProductInterface
{
    public function get(ProductId $id): ProductData; // Data Object
}
```

### ❌ Antipatrón 3: Incluir Datos Sensibles en Eventos

```php
// ❌ INCORRECTO
class PaymentProcessedEvent
{
    public function __construct(
        public string $creditCardNumber, // Sensible!
        public string $cvv, // Sensible!
    ) {}
}
```

✅ **Correcto**: Solo incluir IDs y metadata no sensible.

```php
// ✅ CORRECTO
class PaymentProcessedEvent
{
    public function __construct(
        public PaymentId $paymentId,
        public OrderId $orderId,
        public Carbon $processedAt,
    ) {}
}
```

---

## Testing

### Interfaces: Mockear en Tests

```php
it('lanza excepción si no hay stock disponible', function () {
    $checkAvailability = Mockery::mock(CheckProductAvailabilityInterface::class);
    $checkAvailability->shouldReceive('check')
        ->once()
        ->andReturn(new AvailabilityResultData(isAvailable: false));

    $service = new CreateOrderService($checkAvailability);

    expect(fn () => $service->execute($orderData))
        ->toThrow(InsufficientStockException::class);
});
```

### Eventos: Verificar Dispatching

```php
it('emite OrderConfirmedEvent después de crear orden', function () {
    Event::fake();

    $service->execute($orderData);

    Event::assertDispatched(OrderConfirmedEvent::class);
});
```

---

## Referencias

- **Laravel Events**: https://laravel.com/docs/events
- **nwidart/laravel-modules**: https://nwidartmodules.com/
