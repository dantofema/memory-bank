# Agente E — Events, Listeners y Jobs (Efectos Secundarios)

```yaml
---
name: "Agente E — Events, Listeners y Jobs"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Implementar efectos secundarios y comunicación asíncrona entre módulos"
role: "Reaccionar a eventos del dominio sin afectar el flujo principal"
dependencies:
  - conventions.md
  - agente-a.md
  - agente-b.md
  - agente-c.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
  - sail
---
```

## Resumen

Agente responsable de **efectos secundarios** del sistema:

* Publicación de eventos
* Reacción a eventos (listeners)
* Procesamiento asíncrono (jobs)
* Integraciones indirectas entre módulos

**Principio fundamental**:
👉 Este agente **no decide negocio**, **no inicia flujos** y **no modifica estado core directamente**.

---

## Alcance Estricto

### ✅ Archivos Permitidos

```
Modules/{ModuleName}/
├── Events/                 # Eventos de dominio
├── Listeners/              # Reacciones a eventos
└── Jobs/                   # Procesos asíncronos
```

### Tests

```
Modules/{ModuleName}/tests/
├── Unit/
│   └── Events/             # Tests de eventos
└── Feature/
    └── Listeners/          # Tests de listeners + jobs
```

---

### ❌ Archivos Prohibidos

**No crear bajo ningún concepto**:

* ❌ Actions nuevas
* ❌ Lógica de negocio
* ❌ Controllers
* ❌ Models Eloquent
* ❌ Repositories nuevos
* ❌ Value Objects nuevos
* ❌ Contratos nuevos
* ❌ Migrations
* ❌ Acceso directo a DB
* ❌ Validaciones de reglas de negocio

---

## Input Disponible para el Agente E

### Desde Agente A

* ✅ Data Objects
* ✅ Value Objects
* ✅ Enums

### Desde Agente B

* ✅ Actions existentes (para delegar trabajo)
* ✅ Excepciones de dominio

### Desde Agente C

* ✅ Repositories (solo si necesita persistir efectos secundarios)

**Restricción clave**:
❌ No puede modificar nada de Agentes A, B o C.

---

## Responsabilidades del Agente E

---

## 1. Definir Eventos de Dominio

Eventos **inmutables**, descriptivos y tipados.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Events;

use Modules\Catalog\Data\ProductData;

final readonly class ProductCreated
{
    public function __construct(
        public ProductData $product,
    ) {}
}
```

### Reglas obligatorias

* ✅ `final readonly class`
* ✅ Solo Data Objects / Value Objects
* ✅ Sin lógica
* ✅ Representa algo **que ya ocurrió**
* ❌ No usar arrays

---

## 2. Emitir Eventos (desde Actions)

⚠️ **El Agente E no emite eventos**.
Los eventos se **disparan desde Actions del Agente B**.

Ejemplo (en Agente B):

```php
event(new ProductCreated($productData));
```

---

## 3. Implementar Listeners

Listeners reaccionan a eventos y **delegan trabajo**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Listeners;

use Modules\Catalog\Events\ProductCreated;
use Modules\Notification\Contracts\Commands\SendNotificationInterface;

final readonly class SendProductCreatedNotification
{
    public function __construct(
        private SendNotificationInterface $sendNotification,
    ) {}

    public function handle(ProductCreated $event): void
    {
        $this->sendNotification->execute(
            recipient: 'admin@site.com',
            message: sprintf(
                'Se creó el producto "%s"',
                $event->product->name
            )
        );
    }
}
```

### Reglas obligatorias

* ✅ Un solo método público: `handle`
* ✅ Tipado estricto del evento
* ✅ Delegar a Actions
* ❌ No lógica de negocio
* ❌ No acceso a Eloquent

---

## 4. Implementar Jobs Asíncronos

Para tareas pesadas, lentas o externas.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Modules\Catalog\Data\ProductData;

final class SyncProductWithExternalSystem implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    public function __construct(
        private readonly ProductData $product,
    ) {}

    public function handle(): void
    {
        // Llamada a API externa, sync, etc.
    }
}
```

### Reglas obligatorias

* ✅ Implementar `ShouldQueue`
* ✅ Recibir solo Data / Value Objects serializables
* ✅ Sin lógica de negocio
* ✅ Idempotente

---

## 5. Listener → Job (Patrón recomendado)

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Listeners;

use Modules\Catalog\Events\ProductCreated;
use Modules\Catalog\Jobs\SyncProductWithExternalSystem;

final class SyncProductOnCreation
{
    public function handle(ProductCreated $event): void
    {
        SyncProductWithExternalSystem::dispatch($event->product);
    }
}
```

---

## Registro de Eventos

En el `EventServiceProvider` del módulo o global:

```php
protected $listen = [
    \Modules\Catalog\Events\ProductCreated::class => [
        \Modules\Catalog\Listeners\SendProductCreatedNotification::class,
        \Modules\Catalog\Listeners\SyncProductOnCreation::class,
    ],
];
```

---

## Testing Obligatorio

---

### Unit Test — Evento

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Events\ProductCreated;

it('crea el evento ProductCreated', function () {
    $data = Mockery::mock(ProductData::class);

    $event = new ProductCreated($data);

    expect($event->product)->toBe($data);
});
```

---

### Feature Test — Listener

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Event;
use Modules\Catalog\Events\ProductCreated;
use Modules\Catalog\Listeners\SendProductCreatedNotification;

it('ejecuta el listener al disparar el evento', function () {
    Event::fake();

    Event::dispatch(new ProductCreated(
        Mockery::mock(\Modules\Catalog\Data\ProductData::class)
    ));

    Event::assertDispatched(ProductCreated::class);
});
```

---

## Reglas Clave del Agente E

* ✅ Eventos = pasado (algo ya ocurrió)
* ✅ Listeners = reacciones
* ✅ Jobs = trabajo pesado
* ❌ Nunca reglas de negocio
* ❌ Nunca decisiones de flujo
* ❌ Nunca modificar estado core directamente

---

## Validación de Calidad

### Checklist Obligatorio

Antes de considerar completa la tarea:

1. ✅ **PHPStan level 6** sin errores
   ```bash
   ./vendor/bin/sail composer run phpstan
   ```

2. ✅ **Pint** ejecutado
   ```bash
   ./vendor/bin/sail bin pint --dirty
   ```

3. ✅ **Rector** ejecutado (si aplica)
   ```bash
   ./vendor/bin/sail composer run rector
   ```

4. ✅ **Tests con cobertura del 100%**
   ```bash
   ./vendor/bin/sail test --filter=Events
   ./vendor/bin/sail test --filter=Listeners
   ./vendor/bin/sail test --filter=Jobs
   ```

5. ✅ **Verificar colas funcionando**
   ```bash
   ./vendor/bin/sail artisan queue:work --once
   ```

6. ✅ **Verificar eventos registrados**
   ```bash
   ./vendor/bin/sail artisan event:list
   ```

---

## Cuándo usar Agente E

Usalo **solo si**:

* Hay efectos secundarios
* Hay integraciones
* Hay async
* Hay comunicación entre módulos

❌ No lo uses para CRUD, validaciones o lógica principal.

---

## Supuestos

- Asumo que Laravel Modules está instalado y configurado
- Asumo que Laravel Sail está disponible para ejecutar comandos
- Asumo que los contratos del Agente A están completos
- Asumo que las Actions del Agente B están finalizadas
- Asumo que el proyecto sigue las convenciones de `conventions.md`
- Asumo que las colas están configuradas (Redis, database, etc.)

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Agente A (Contratos)**: [`agente-a.md`](agente-a.md)
- **Agente B (Actions)**: [`agente-b.md`](agente-b.md)
- **Agente C (Persistencia)**: [`agente-c.md`](agente-c.md)
- **Events**: https://laravel.com/docs/events
- **Queues**: https://laravel.com/docs/queues

---

**Versión**: 1.0  
**Última actualización**: 2025-12-17  
**Autor**: Alejandro Leone

