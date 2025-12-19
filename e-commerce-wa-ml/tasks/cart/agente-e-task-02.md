---
task_id: "agente-e-task-02"
module: "Auth"
agent: "Agente E"
title: "Listeners de Auditoría y Seguridad"
priority: "high"
estimated_time: "2h"
dependencies:
  - "agente-e-task-01: Eventos creados"
  - "Agente B: Actions implementadas"
  - "Agente C: Repositories disponibles"
status: "pending"
---

# Task: Listeners de Auditoría y Seguridad

## Contexto

Crear listeners que reaccionen a los eventos del módulo Auth para implementar **auditoría de seguridad** y **logging**. Los listeners NO contienen lógica de negocio, solo **reaccionan** a eventos y **delegan trabajo** a Actions o Jobs.

**Casos de uso**:
- Registrar intentos de autenticación (éxito/fallo)
- Registrar solicitudes de password reset
- Registrar cambios de contraseña
- Notificar actividades sospechosas

---

## Objetivos

1. ✅ Crear listener `LogAuthenticationAttempt`
2. ✅ Crear listener `LogPasswordResetRequest`
3. ✅ Crear listener `LogPasswordResetCompleted`
4. ✅ Registrar listeners en `EventServiceProvider`
5. ✅ Tests de listeners con Event fake
6. ✅ Validar PHPStan level 6

---

## Listeners a Crear

### 1. Listener: LogAuthenticationAttempt

**Ubicación**: `Modules/Auth/app/Listeners/LogAuthenticationAttempt.php`

**Descripción**: Registra en logs cada intento de autenticación exitoso para auditoría de seguridad.

**Reacciona a**: `MerchantAuthenticated`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\MerchantAuthenticated;

/**
 * Registra en logs los intentos de autenticación exitosos.
 */
final readonly class LogAuthenticationAttempt
{
    public function handle(MerchantAuthenticated $event): void
    {
        Log::info('Merchant authenticated successfully', [
            'user_id' => $event->userId,
            'email' => $event->email->toString(),
            'ip_address' => $event->ipAddress,
            'authenticated_at' => $event->authenticatedAt->toIso8601String(),
            'context' => 'security_audit',
        ]);
    }
}
```

**Características**:
- ✅ `final readonly class`
- ✅ Un solo método público: `handle()`
- ✅ Tipado estricto del evento
- ✅ No contiene lógica de negocio
- ✅ Solo registra información

---

### 2. Listener: LogPasswordResetRequest

**Ubicación**: `Modules/Auth/app/Listeners/LogPasswordResetRequest.php`

**Descripción**: Registra en logs cada solicitud de password reset para detectar patrones sospechosos.

**Reacciona a**: `PasswordResetRequested`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\PasswordResetRequested;

/**
 * Registra en logs las solicitudes de password reset.
 */
final readonly class LogPasswordResetRequest
{
    public function handle(PasswordResetRequested $event): void
    {
        Log::info('Password reset requested', [
            'email' => $event->email->toString(),
            'ip_address' => $event->ipAddress,
            'requested_at' => $event->requestedAt->toIso8601String(),
            'context' => 'security_audit',
        ]);
    }
}
```

**Características**:
- ✅ Mismo patrón que `LogAuthenticationAttempt`
- ✅ Registra información relevante para seguridad
- ✅ Sin lógica de negocio

---

### 3. Listener: LogPasswordResetCompleted

**Ubicación**: `Modules/Auth/app/Listeners/LogPasswordResetCompleted.php`

**Descripción**: Registra cuando un usuario completa exitosamente un password reset.

**Reacciona a**: `PasswordResetCompleted`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Listeners;

use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\PasswordResetCompleted;

/**
 * Registra en logs cuando se completa un password reset.
 */
final readonly class LogPasswordResetCompleted
{
    public function handle(PasswordResetCompleted $event): void
    {
        Log::info('Password reset completed successfully', [
            'user_id' => $event->userId,
            'email' => $event->email->toString(),
            'ip_address' => $event->ipAddress,
            'completed_at' => $event->completedAt->toIso8601String(),
            'context' => 'security_audit',
        ]);
    }
}
```

**Características**:
- ✅ Registra información de auditoría
- ✅ Contexto de seguridad
- ✅ Sin lógica de negocio

---

## Registro de Listeners

### EventServiceProvider del Módulo Auth

**Ubicación**: `Modules/Auth/app/Providers/EventServiceProvider.php`

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Modules\Auth\Events\MerchantAuthenticated;
use Modules\Auth\Events\PasswordResetCompleted;
use Modules\Auth\Events\PasswordResetRequested;
use Modules\Auth\Listeners\LogAuthenticationAttempt;
use Modules\Auth\Listeners\LogPasswordResetCompleted;
use Modules\Auth\Listeners\LogPasswordResetRequest;

final class EventServiceProvider extends ServiceProvider
{
    /**
     * @var array<string, array<int, string>>
     */
    protected $listen = [
        MerchantAuthenticated::class => [
            LogAuthenticationAttempt::class,
        ],
        PasswordResetRequested::class => [
            LogPasswordResetRequest::class,
        ],
        PasswordResetCompleted::class => [
            LogPasswordResetCompleted::class,
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
    "Modules\\Auth\\Providers\\AuthServiceProvider",
    "Modules\\Auth\\Providers\\EventServiceProvider"
  ]
}
```

---

## Tests de Listeners

### Estrategia de Testing

Los listeners se testean con `Event::fake()` para verificar que:
1. Los listeners se ejecutan cuando se dispara el evento
2. Los listeners registran la información correcta en logs

---

### Test: LogAuthenticationAttemptTest

**Ubicación**: `Modules/Auth/tests/Feature/Listeners/LogAuthenticationAttemptTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\MerchantAuthenticated;
use Modules\Auth\Listeners\LogAuthenticationAttempt;

it('registra en logs cuando un merchant se autentica', function () {
    // Fake de logs
    Log::spy();

    // Crear evento
    $event = new MerchantAuthenticated(
        userId: 1,
        email: new EmailValueObject('test@example.com'),
        ipAddress: '192.168.1.1',
        authenticatedAt: now(),
    );

    // Ejecutar listener
    $listener = new LogAuthenticationAttempt();
    $listener->handle($event);

    // Verificar que se registró el log
    Log::shouldHaveReceived('info')
        ->once()
        ->with('Merchant authenticated successfully', \Mockery::on(function ($context) {
            return $context['user_id'] === 1
                && $context['email'] === 'test@example.com'
                && $context['ip_address'] === '192.168.1.1'
                && $context['context'] === 'security_audit';
        }));
});

it('ejecuta el listener cuando se dispara el evento MerchantAuthenticated', function () {
    Event::fake([MerchantAuthenticated::class]);

    // Disparar evento
    event(new MerchantAuthenticated(
        userId: 1,
        email: new EmailValueObject('test@example.com'),
        ipAddress: '192.168.1.1',
        authenticatedAt: now(),
    ));

    // Verificar que el evento se disparó
    Event::assertDispatched(MerchantAuthenticated::class);
});
```

---

### Test: LogPasswordResetRequestTest

**Ubicación**: `Modules/Auth/tests/Feature/Listeners/LogPasswordResetRequestTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\PasswordResetRequested;
use Modules\Auth\Listeners\LogPasswordResetRequest;

it('registra en logs cuando se solicita password reset', function () {
    Log::spy();

    $event = new PasswordResetRequested(
        email: new EmailValueObject('reset@example.com'),
        ipAddress: '192.168.1.2',
        requestedAt: now(),
    );

    $listener = new LogPasswordResetRequest();
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Password reset requested', \Mockery::on(function ($context) {
            return $context['email'] === 'reset@example.com'
                && $context['ip_address'] === '192.168.1.2'
                && $context['context'] === 'security_audit';
        }));
});

it('ejecuta el listener cuando se dispara el evento PasswordResetRequested', function () {
    Event::fake([PasswordResetRequested::class]);

    event(new PasswordResetRequested(
        email: new EmailValueObject('reset@example.com'),
        ipAddress: '192.168.1.2',
        requestedAt: now(),
    ));

    Event::assertDispatched(PasswordResetRequested::class);
});
```

---

### Test: LogPasswordResetCompletedTest

**Ubicación**: `Modules/Auth/tests/Feature/Listeners/LogPasswordResetCompletedTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Events\PasswordResetCompleted;
use Modules\Auth\Listeners\LogPasswordResetCompleted;

it('registra en logs cuando se completa password reset', function () {
    Log::spy();

    $event = new PasswordResetCompleted(
        userId: 5,
        email: new EmailValueObject('completed@example.com'),
        ipAddress: '192.168.1.3',
        completedAt: now(),
    );

    $listener = new LogPasswordResetCompleted();
    $listener->handle($event);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Password reset completed successfully', \Mockery::on(function ($context) {
            return $context['user_id'] === 5
                && $context['email'] === 'completed@example.com'
                && $context['ip_address'] === '192.168.1.3'
                && $context['context'] === 'security_audit';
        }));
});

it('ejecuta el listener cuando se dispara el evento PasswordResetCompleted', function () {
    Event::fake([PasswordResetCompleted::class]);

    event(new PasswordResetCompleted(
        userId: 5,
        email: new EmailValueObject('completed@example.com'),
        ipAddress: '192.168.1.3',
        completedAt: now(),
    ));

    Event::assertDispatched(PasswordResetCompleted::class);
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
  ./vendor/bin/sail test Modules/Auth/Tests/Feature/Listeners
  ```

- [ ] **Listeners registrados correctamente**
  ```bash
  ./vendor/bin/sail artisan event:list | grep Auth
  ```

  **Salida esperada**:
  ```
  Modules\Auth\Events\MerchantAuthenticated
    Modules\Auth\Listeners\LogAuthenticationAttempt
  
  Modules\Auth\Events\PasswordResetRequested
    Modules\Auth\Listeners\LogPasswordResetRequest
  
  Modules\Auth\Events\PasswordResetCompleted
    Modules\Auth\Listeners\LogPasswordResetCompleted
  ```

---

## Estructura de Archivos Final

```
Modules/Auth/app/
├── Events/
│   ├── MerchantAuthenticated.php
│   ├── PasswordResetRequested.php
│   └── PasswordResetCompleted.php
├── Listeners/
│   ├── LogAuthenticationAttempt.php ✅
│   ├── LogPasswordResetRequest.php ✅
│   └── LogPasswordResetCompleted.php ✅
└── Providers/
    └── EventServiceProvider.php ✅

Modules/Auth/tests/Feature/Listeners/
├── LogAuthenticationAttemptTest.php ✅
├── LogPasswordResetRequestTest.php ✅
└── LogPasswordResetCompletedTest.php ✅
```

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Los 3 listeners están creados con tipado fuerte
2. ✅ Todos son `final readonly class`
3. ✅ Solo tienen método `handle()`
4. ✅ Sin lógica de negocio
5. ✅ EventServiceProvider registrado correctamente
6. ✅ Tests de listeners verdes (100% cobertura)
7. ✅ PHPStan level 6 sin errores
8. ✅ `event:list` muestra los listeners registrados

---

## Notas Importantes

### 🎯 Listeners solo reaccionan

- No contienen lógica de negocio
- No toman decisiones
- Solo registran información o delegan trabajo

### 📝 Contexto de seguridad

Todos los logs incluyen `'context' => 'security_audit'` para:
- Facilitar filtrado en sistemas de logs
- Identificar eventos de seguridad
- Auditoría y compliance

### ⚡ Listeners síncronos en MVP

En MVP, los listeners se ejecutan **síncronamente**. En futuro se pueden convertir a Jobs asíncronos si el volumen de logs afecta performance.

---

## Referencias

- **Agente E (metodología)**: `laravel/agents/agente-e.md`
- **agente-e-task-01**: Eventos del módulo Auth
- **Laravel Events**: https://laravel.com/docs/events
- **Laravel Logging**: https://laravel.com/docs/logging

---

**Última actualización**: 2025-12-18  
**Autor**: Alejandro Leone
