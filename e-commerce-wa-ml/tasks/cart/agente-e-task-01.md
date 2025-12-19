---
task_id: "agente-e-task-01"
module: "Auth"
agent: "Agente E"
title: "Eventos de Dominio del Módulo Auth"
priority: "high"
estimated_time: "1h"
dependencies:
  - "Agente A: Data Objects"
  - "Agente B: Actions implementadas"
status: "pending"
---

# Task: Eventos de Dominio del Módulo Auth

## Contexto

Crear los eventos de dominio que representan acciones importantes del módulo Auth. Los eventos son **inmutables**, **descriptivos** y **representan algo que ya ocurrió**. Se disparan desde las Actions del Agente B y permiten reaccionar con efectos secundarios (notificaciones, logs, auditoría).

**Nota**: El Agente E **NO dispara eventos**, solo los define. Los eventos se disparan desde las Actions del Agente B.

---

## Objetivos

1. ✅ Crear evento `MerchantAuthenticated`
2. ✅ Crear evento `PasswordResetRequested`
3. ✅ Crear evento `PasswordResetCompleted`
4. ✅ Tests unitarios de eventos
5. ✅ Validar PHPStan level 6

---

## Eventos a Crear

### 1. Evento: MerchantAuthenticated

**Ubicación**: `Modules/Auth/app/Events/MerchantAuthenticated.php`

**Descripción**: Se dispara cuando un merchant se autentica exitosamente en el sistema.

**Cuándo se dispara**: Desde `AuthenticateMerchantAction` después de `Auth::attempt()` exitoso.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Events;

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;

/**
 * Evento disparado cuando un merchant se autentica exitosamente.
 */
final readonly class MerchantAuthenticated
{
    public function __construct(
        public int $userId,
        public EmailValueObject $email,
        public string $ipAddress,
        public Carbon $authenticatedAt,
    ) {}
}
```

**Propiedades**:
- `userId`: ID del usuario autenticado
- `email`: Email del merchant (Value Object)
- `ipAddress`: IP desde donde se autenticó
- `authenticatedAt`: Timestamp de la autenticación

**Uso en Action** (referencia, NO implementar aquí):
```php
// En AuthenticateMerchantAction::execute()
event(new MerchantAuthenticated(
    userId: $user->id,
    email: $user->email,
    ipAddress: request()->ip(),
    authenticatedAt: now(),
));
```

---

### 2. Evento: PasswordResetRequested

**Ubicación**: `Modules/Auth/app/Events/PasswordResetRequested.php`

**Descripción**: Se dispara cuando un usuario solicita recuperar su contraseña.

**Cuándo se dispara**: Desde `RequestPasswordResetAction` después de crear el token.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Events;

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;

/**
 * Evento disparado cuando se solicita un reset de contraseña.
 */
final readonly class PasswordResetRequested
{
    public function __construct(
        public EmailValueObject $email,
        public string $ipAddress,
        public Carbon $requestedAt,
    ) {}
}
```

**Propiedades**:
- `email`: Email del solicitante (Value Object)
- `ipAddress`: IP desde donde se solicitó
- `requestedAt`: Timestamp de la solicitud

**Uso en Action** (referencia, NO implementar aquí):
```php
// En RequestPasswordResetAction::execute()
event(new PasswordResetRequested(
    email: $data->email,
    ipAddress: request()->ip(),
    requestedAt: now(),
));
```

---

### 3. Evento: PasswordResetCompleted

**Ubicación**: `Modules/Auth/app/Events/PasswordResetCompleted.php`

**Descripción**: Se dispara cuando un usuario completa exitosamente el reset de contraseña.

**Cuándo se dispara**: Desde `ResetPasswordAction` después de actualizar la contraseña.

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Events;

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;

/**
 * Evento disparado cuando se completa un reset de contraseña.
 */
final readonly class PasswordResetCompleted
{
    public function __construct(
        public int $userId,
        public EmailValueObject $email,
        public string $ipAddress,
        public Carbon $completedAt,
    ) {}
}
```

**Propiedades**:
- `userId`: ID del usuario que reseteó su contraseña
- `email`: Email del usuario (Value Object)
- `ipAddress`: IP desde donde se completó el reset
- `completedAt`: Timestamp del reset

**Uso en Action** (referencia, NO implementar aquí):
```php
// En ResetPasswordAction::execute()
event(new PasswordResetCompleted(
    userId: $user->id,
    email: $data->email,
    ipAddress: request()->ip(),
    completedAt: now(),
));
```

---

## Reglas de Implementación

### Características Obligatorias

✅ **Clase final y readonly**:
```php
final readonly class MerchantAuthenticated
```

✅ **Solo constructor con propiedades promovidas**:
```php
public function __construct(
    public int $userId,
    public EmailValueObject $email,
) {}
```

✅ **Usar Value Objects cuando aplique**:
```php
public EmailValueObject $email, // ✅ Value Object
public Carbon $authenticatedAt, // ✅ Carbon para fechas
```

✅ **Sin lógica de negocio**:
```php
// ❌ NO hacer esto
public function shouldNotify(): bool { ... }

// ✅ Eventos solo datos
```

✅ **Docblock descriptivo**:
```php
/**
 * Evento disparado cuando un merchant se autentica exitosamente.
 */
```

---

### ❌ Lo que NO debe tener un Evento

❌ **Métodos públicos** (excepto constructor)
❌ **Lógica de negocio**
❌ **Validaciones**
❌ **Acceso a base de datos**
❌ **Arrays públicos** (usar Data Objects o Value Objects)
❌ **Propiedades mutables**

---

## Tests Unitarios

### Ubicación de Tests

```
Modules/Auth/tests/Unit/Events/
├── MerchantAuthenticatedTest.php
├── PasswordResetRequestedTest.php
└── PasswordResetCompletedTest.php
```

---

### Test: MerchantAuthenticatedTest

**Ubicación**: `Modules/Auth/tests/Unit/Events/MerchantAuthenticatedTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;
use Modules\Auth\Events\MerchantAuthenticated;

it('crea el evento MerchantAuthenticated con datos correctos', function () {
    $userId = 1;
    $email = new EmailValueObject('test@example.com');
    $ipAddress = '192.168.1.1';
    $authenticatedAt = Carbon::parse('2025-12-18 10:00:00');

    $event = new MerchantAuthenticated(
        userId: $userId,
        email: $email,
        ipAddress: $ipAddress,
        authenticatedAt: $authenticatedAt,
    );

    expect($event->userId)->toBe($userId)
        ->and($event->email)->toBe($email)
        ->and($event->email->toString())->toBe('test@example.com')
        ->and($event->ipAddress)->toBe($ipAddress)
        ->and($event->authenticatedAt)->toEqual($authenticatedAt);
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new MerchantAuthenticated(
        userId: 1,
        email: new EmailValueObject('test@example.com'),
        ipAddress: '192.168.1.1',
        authenticatedAt: now(),
    );

    // Intentar modificar debe fallar
    $event->userId = 2;
})->throws(Error::class);
```

---

### Test: PasswordResetRequestedTest

**Ubicación**: `Modules/Auth/tests/Unit/Events/PasswordResetRequestedTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;
use Modules\Auth\Events\PasswordResetRequested;

it('crea el evento PasswordResetRequested con datos correctos', function () {
    $email = new EmailValueObject('reset@example.com');
    $ipAddress = '192.168.1.2';
    $requestedAt = Carbon::parse('2025-12-18 11:00:00');

    $event = new PasswordResetRequested(
        email: $email,
        ipAddress: $ipAddress,
        requestedAt: $requestedAt,
    );

    expect($event->email)->toBe($email)
        ->and($event->email->toString())->toBe('reset@example.com')
        ->and($event->ipAddress)->toBe($ipAddress)
        ->and($event->requestedAt)->toEqual($requestedAt);
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new PasswordResetRequested(
        email: new EmailValueObject('test@example.com'),
        ipAddress: '192.168.1.1',
        requestedAt: now(),
    );

    $event->ipAddress = '10.0.0.1';
})->throws(Error::class);
```

---

### Test: PasswordResetCompletedTest

**Ubicación**: `Modules/Auth/tests/Unit/Events/PasswordResetCompletedTest.php`

```php
<?php

declare(strict_types=1);

use App\ValueObjects\EmailValueObject;
use Carbon\Carbon;
use Modules\Auth\Events\PasswordResetCompleted;

it('crea el evento PasswordResetCompleted con datos correctos', function () {
    $userId = 5;
    $email = new EmailValueObject('completed@example.com');
    $ipAddress = '192.168.1.3';
    $completedAt = Carbon::parse('2025-12-18 12:00:00');

    $event = new PasswordResetCompleted(
        userId: $userId,
        email: $email,
        ipAddress: $ipAddress,
        completedAt: $completedAt,
    );

    expect($event->userId)->toBe($userId)
        ->and($event->email)->toBe($email)
        ->and($event->email->toString())->toBe('completed@example.com')
        ->and($event->ipAddress)->toBe($ipAddress)
        ->and($event->completedAt)->toEqual($completedAt);
});

it('es readonly y no permite modificar propiedades', function () {
    $event = new PasswordResetCompleted(
        userId: 1,
        email: new EmailValueObject('test@example.com'),
        ipAddress: '192.168.1.1',
        completedAt: now(),
    );

    $event->userId = 999;
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
  ./vendor/bin/sail test Modules/Auth/Tests/Unit/Events
  ```

- [ ] **Eventos son `final readonly`**
- [ ] **Solo contienen constructor**
- [ ] **Usan Value Objects cuando aplica**
- [ ] **Sin lógica de negocio**

---

## Estructura de Archivos Final

```
Modules/Auth/app/Events/
├── MerchantAuthenticated.php ✅
├── PasswordResetRequested.php ✅
└── PasswordResetCompleted.php ✅

Modules/Auth/tests/Unit/Events/
├── MerchantAuthenticatedTest.php ✅
├── PasswordResetRequestedTest.php ✅
└── PasswordResetCompletedTest.php ✅
```

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Los 3 eventos están creados con tipado fuerte
2. ✅ Todos usan `final readonly class`
3. ✅ Usan Value Objects (`EmailValueObject`)
4. ✅ Sin lógica de negocio
5. ✅ Tests unitarios verdes (100% cobertura)
6. ✅ PHPStan level 6 sin errores
7. ✅ Pint ejecutado sin cambios pendientes

---

## Notas Importantes

### ⚠️ El Agente E NO dispara eventos

Los eventos se disparan desde las **Actions del Agente B**:
- `MerchantAuthenticated` → desde `AuthenticateMerchantAction`
- `PasswordResetRequested` → desde `RequestPasswordResetAction`
- `PasswordResetCompleted` → desde `ResetPasswordAction`

### 🎯 Los eventos solo contienen datos

- No contienen lógica
- No validan nada
- No deciden nada
- Solo transportan información

### 📝 Los eventos son inmutables

- `final readonly class`
- Sin setters
- Sin métodos públicos (excepto constructor)

---

## Referencias

- **Agente E (metodología)**: `laravel/agents/agente-e.md`
- **Auth Class Diagram**: `e-commerce-wa-ml/auth/auth-class-diagram.md`
- **Laravel Events**: https://laravel.com/docs/events

---

**Última actualización**: 2025-12-18  
**Autor**: Alejandro Leone
