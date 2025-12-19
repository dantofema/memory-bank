---
task_id: "agente-e-task-04"
module: "Auth"
agent: "Agente E"
title: "Integración de Eventos en Actions y Validación Final"
priority: "high"
estimated_time: "1.5h"
dependencies:
  - "agente-e-task-01: Eventos creados"
  - "agente-e-task-02: Listeners creados"
  - "agente-e-task-03: Job creado"
  - "Agente B: Actions implementadas"
status: "pending"
---

# Task: Integración de Eventos en Actions y Validación Final

## Contexto

Integrar los eventos creados en las Actions del Agente B y validar que todo el sistema de eventos, listeners y jobs funciona correctamente de punta a punta. Esta tarea completa el trabajo del Agente E asegurando que los efectos secundarios se ejecutan correctamente.

**Objetivo**: Asegurar que los eventos se disparan en los momentos correctos y que los listeners reaccionan apropiadamente.

---

## Objetivos

1. ✅ Modificar Actions del Agente B para disparar eventos
2. ✅ Validar integración Events + Listeners + Jobs
3. ✅ Tests de integración end-to-end
4. ✅ Verificar configuración completa
5. ✅ Documentar sistema de eventos

---

## Modificaciones en Actions (Agente B)

### ⚠️ Nota Importante

Técnicamente, **modificar Actions es responsabilidad del Agente B**, no del Agente E. Sin embargo, para completar la integración, el Agente E debe indicar **dónde y cómo** disparar los eventos.

**Estrategia**: Crear tests de integración que validen que las Actions disparan eventos, sin modificar las Actions directamente. Si las Actions no disparan eventos, los tests fallarán y será responsabilidad del Agente B corregirlo.

---

## Especificación de Integración

### 1. AuthenticateMerchantAction

**Ubicación**: `Modules/Auth/app/Actions/AuthenticateMerchantAction.php`

**Evento a disparar**: `MerchantAuthenticated`

**Momento**: Después de `Auth::attempt()` exitoso, antes de retornar `AuthenticationResult`.

**Código esperado** (dentro de `execute()`):

```php
// Después de Auth::attempt() exitoso
if ($authenticated) {
    event(new MerchantAuthenticated(
        userId: $user->id,
        email: $user->email,
        ipAddress: request()->ip() ?? '127.0.0.1',
        authenticatedAt: now(),
    ));

    return new AuthenticationResult(
        success: true,
        user: $user,
    );
}
```

---

### 2. RequestPasswordResetAction

**Ubicación**: `Modules/Auth/app/Actions/RequestPasswordResetAction.php`

**Evento a disparar**: `PasswordResetRequested`

**Momento**: Después de crear el token, antes de enviar el email.

**Código esperado** (dentro de `execute()`):

```php
// Después de crear token
$token = $this->repository->create($data->email);

event(new PasswordResetRequested(
    email: $data->email,
    ipAddress: request()->ip() ?? '127.0.0.1',
    requestedAt: now(),
));

// Enviar email...
```

---

### 3. ResetPasswordAction

**Ubicación**: `Modules/Auth/app/Actions/ResetPasswordAction.php`

**Evento a disparar**: `PasswordResetCompleted`

**Momento**: Después de actualizar la contraseña y antes de retornar.

**Código esperado** (dentro de `execute()`):

```php
// Después de actualizar password
$user->password = PasswordValueObject::fromPlain($data->password);
$user->save();

// Eliminar token
$this->repository->delete($data->email);

event(new PasswordResetCompleted(
    userId: $user->id,
    email: $data->email,
    ipAddress: request()->ip() ?? '127.0.0.1',
    completedAt: now(),
));
```

---

## Tests de Integración End-to-End

### Objetivo

Validar que el flujo completo funciona:
1. Action ejecuta su lógica
2. Action dispara evento
3. Listener reacciona al evento
4. Listener registra en logs

---

### Test: Flujo Completo de Autenticación

**Ubicación**: `Modules/Auth/tests/Feature/Integration/AuthenticationFlowTest.php`

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Actions\AuthenticateMerchantAction;
use Modules\Auth\Data\LoginData;
use Modules\Auth\Events\MerchantAuthenticated;
use Modules\Auth\Listeners\LogAuthenticationAttempt;

it('dispara evento MerchantAuthenticated y ejecuta listener al autenticar', function () {
    Event::fake();
    Log::spy();

    // Crear usuario
    $user = User::factory()->owner()->create([
        'email' => 'test@example.com',
        'password' => \App\ValueObjects\PasswordValueObject::fromPlain('password123'),
    ]);

    // Autenticar
    $action = app(AuthenticateMerchantAction::class);
    $data = new LoginData(
        email: new \App\ValueObjects\EmailValueObject('test@example.com'),
        password: 'password123',
        remember: false,
    );

    $result = $action->execute($data);

    // Validar que autenticó
    expect($result->success)->toBeTrue()
        ->and($result->user->id)->toBe($user->id);

    // Validar que disparó el evento
    Event::assertDispatched(MerchantAuthenticated::class, function ($event) use ($user) {
        return $event->userId === $user->id
            && $event->email->toString() === 'test@example.com';
    });
});

it('el listener registra en logs cuando se autentica un merchant', function () {
    Log::spy();

    // Crear usuario
    $user = User::factory()->owner()->create([
        'email' => 'test@example.com',
        'password' => \App\ValueObjects\PasswordValueObject::fromPlain('password123'),
    ]);

    // Autenticar (sin Event::fake para que los listeners se ejecuten)
    $action = app(AuthenticateMerchantAction::class);
    $data = new LoginData(
        email: new \App\ValueObjects\EmailValueObject('test@example.com'),
        password: 'password123',
        remember: false,
    );

    $result = $action->execute($data);

    expect($result->success)->toBeTrue();

    // Validar que el listener registró en logs
    Log::shouldHaveReceived('info')
        ->once()
        ->with('Merchant authenticated successfully', \Mockery::on(function ($context) use ($user) {
            return $context['user_id'] === $user->id
                && $context['email'] === 'test@example.com'
                && $context['context'] === 'security_audit';
        }));
});
```

---

### Test: Flujo Completo de Password Reset Request

**Ubicación**: `Modules/Auth/tests/Feature/Integration/PasswordResetRequestFlowTest.php`

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Actions\RequestPasswordResetAction;
use Modules\Auth\Data\RequestPasswordResetData;
use Modules\Auth\Events\PasswordResetRequested;

it('dispara evento PasswordResetRequested y ejecuta listener', function () {
    Event::fake();
    Log::spy();

    // Crear usuario
    User::factory()->create([
        'email' => 'reset@example.com',
    ]);

    // Solicitar reset
    $action = app(RequestPasswordResetAction::class);
    $data = new RequestPasswordResetData(
        email: new \App\ValueObjects\EmailValueObject('reset@example.com'),
    );

    $action->execute($data);

    // Validar que disparó el evento
    Event::assertDispatched(PasswordResetRequested::class, function ($event) {
        return $event->email->toString() === 'reset@example.com';
    });
});

it('el listener registra en logs cuando se solicita password reset', function () {
    Log::spy();

    User::factory()->create([
        'email' => 'reset@example.com',
    ]);

    $action = app(RequestPasswordResetAction::class);
    $data = new RequestPasswordResetData(
        email: new \App\ValueObjects\EmailValueObject('reset@example.com'),
    );

    $action->execute($data);

    // Validar que el listener registró en logs
    Log::shouldHaveReceived('info')
        ->once()
        ->with('Password reset requested', \Mockery::on(function ($context) {
            return $context['email'] === 'reset@example.com'
                && $context['context'] === 'security_audit';
        }));
});
```

---

### Test: Flujo Completo de Password Reset Completion

**Ubicación**: `Modules/Auth/tests/Feature/Integration/PasswordResetCompletionFlowTest.php`

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Actions\ResetPasswordAction;
use Modules\Auth\Data\ResetPasswordData;
use Modules\Auth\Events\PasswordResetCompleted;
use Modules\Auth\Models\PasswordResetToken;

it('dispara evento PasswordResetCompleted y ejecuta listener', function () {
    Event::fake();
    Log::spy();

    $user = User::factory()->create([
        'email' => 'completed@example.com',
    ]);

    $token = PasswordResetToken::factory()->create([
        'email' => new \App\ValueObjects\EmailValueObject('completed@example.com'),
    ]);

    $action = app(ResetPasswordAction::class);
    $data = new ResetPasswordData(
        email: new \App\ValueObjects\EmailValueObject('completed@example.com'),
        token: 'plain-token-value', // Asume que el factory guarda token hasheado
        password: 'NewPassword123',
    );

    $action->execute($data);

    Event::assertDispatched(PasswordResetCompleted::class, function ($event) use ($user) {
        return $event->userId === $user->id
            && $event->email->toString() === 'completed@example.com';
    });
});

it('el listener registra en logs cuando se completa password reset', function () {
    Log::spy();

    $user = User::factory()->create([
        'email' => 'completed@example.com',
    ]);

    $token = PasswordResetToken::factory()->create([
        'email' => new \App\ValueObjects\EmailValueObject('completed@example.com'),
    ]);

    $action = app(ResetPasswordAction::class);
    $data = new ResetPasswordData(
        email: new \App\ValueObjects\EmailValueObject('completed@example.com'),
        token: 'plain-token-value',
        password: 'NewPassword123',
    );

    $action->execute($data);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Password reset completed successfully', \Mockery::on(function ($context) use ($user) {
            return $context['user_id'] === $user->id
                && $context['email'] === 'completed@example.com'
                && $context['context'] === 'security_audit';
        }));
});
```

---

## Validación de Configuración Completa

### Checklist de Verificación

```bash
# 1. Verificar que eventos están registrados
./vendor/bin/sail artisan event:list | grep Auth

# Salida esperada:
# Modules\Auth\Events\MerchantAuthenticated
#   Modules\Auth\Listeners\LogAuthenticationAttempt
# Modules\Auth\Events\PasswordResetRequested
#   Modules\Auth\Listeners\LogPasswordResetRequest
# Modules\Auth\Events\PasswordResetCompleted
#   Modules\Auth\Listeners\LogPasswordResetCompleted

# 2. Verificar que Job está programado
./vendor/bin/sail artisan schedule:list | grep auth:clean-expired-tokens

# 3. Ejecutar tests de integración
./vendor/bin/sail test Modules/Auth/Tests/Feature/Integration

# 4. PHPStan
./vendor/bin/sail composer run phpstan

# 5. Pint
./vendor/bin/sail bin pint --dirty
```

---

## Documentación del Sistema de Eventos

### Documento: Sistema de Eventos del Módulo Auth

**Ubicación**: `Modules/Auth/docs/EVENTS.md`

```markdown
# Sistema de Eventos - Módulo Auth

## Eventos Disponibles

### 1. MerchantAuthenticated

**Disparado cuando**: Un merchant se autentica exitosamente.

**Datos del evento**:
- `userId`: ID del usuario autenticado
- `email`: Email del merchant
- `ipAddress`: IP desde donde se autenticó
- `authenticatedAt`: Timestamp de autenticación

**Listeners**:
- `LogAuthenticationAttempt`: Registra en logs para auditoría

**Uso**:
```php
event(new MerchantAuthenticated(
    userId: $user->id,
    email: $user->email,
    ipAddress: request()->ip(),
    authenticatedAt: now(),
));
```

---

### 2. PasswordResetRequested

**Disparado cuando**: Un usuario solicita recuperar su contraseña.

**Datos del evento**:
- `email`: Email del solicitante
- `ipAddress`: IP desde donde se solicitó
- `requestedAt`: Timestamp de la solicitud

**Listeners**:
- `LogPasswordResetRequest`: Registra en logs para detectar patrones sospechosos

**Uso**:
```php
event(new PasswordResetRequested(
    email: $data->email,
    ipAddress: request()->ip(),
    requestedAt: now(),
));
```

---

### 3. PasswordResetCompleted

**Disparado cuando**: Un usuario completa exitosamente el reset de contraseña.

**Datos del evento**:
- `userId`: ID del usuario
- `email`: Email del usuario
- `ipAddress`: IP desde donde se completó
- `completedAt`: Timestamp del reset

**Listeners**:
- `LogPasswordResetCompleted`: Registra en logs para auditoría

**Uso**:
```php
event(new PasswordResetCompleted(
    userId: $user->id,
    email: $data->email,
    ipAddress: request()->ip(),
    completedAt: now(),
));
```

---

## Jobs Programados

### CleanExpiredPasswordResetTokensJob

**Frecuencia**: Diaria a las 02:00 AM

**Descripción**: Elimina tokens de password reset expirados (> 60 minutos).

**Ejecución manual**:
```bash
./vendor/bin/sail artisan tinker
>>> dispatch(new \Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob());
```

---

## Testing

Todos los eventos, listeners y jobs están 100% cubiertos por tests:

```bash
# Tests de eventos
./vendor/bin/sail test Modules/Auth/Tests/Unit/Events

# Tests de listeners
./vendor/bin/sail test Modules/Auth/Tests/Feature/Listeners

# Tests de jobs
./vendor/bin/sail test Modules/Auth/Tests/Feature/Jobs

# Tests de integración
./vendor/bin/sail test Modules/Auth/Tests/Feature/Integration
```

---

## Auditoría y Seguridad

Todos los logs incluyen `'context' => 'security_audit'` para facilitar:
- Filtrado en sistemas de logs (ELK, CloudWatch, etc.)
- Identificación de eventos de seguridad
- Compliance y auditorías externas

**Ejemplo de log**:
```json
{
  "message": "Merchant authenticated successfully",
  "user_id": 1,
  "email": "admin@example.com",
  "ip_address": "192.168.1.1",
  "authenticated_at": "2025-12-18T10:00:00Z",
  "context": "security_audit"
}
```
```

**Crear archivo**:
```bash
mkdir -p Modules/Auth/docs
# Copiar contenido del markdown anterior
```

---

## Validación de Calidad

### Checklist Final

- [ ] **Events registrados correctamente**
  ```bash
  ./vendor/bin/sail artisan event:list | grep Auth
  ```

- [ ] **Scheduler configurado**
  ```bash
  ./vendor/bin/sail artisan schedule:list | grep auth
  ```

- [ ] **Tests de integración verdes**
  ```bash
  ./vendor/bin/sail test Modules/Auth/Tests/Feature/Integration
  ```

- [ ] **PHPStan level 6 sin errores**
  ```bash
  ./vendor/bin/sail composer run phpstan
  ```

- [ ] **Pint sin cambios pendientes**
  ```bash
  ./vendor/bin/sail bin pint --dirty
  ```

- [ ] **Documentación EVENTS.md creada**

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Actions disparan eventos en momentos correctos
2. ✅ Listeners reaccionan correctamente
3. ✅ Tests de integración verdes (100% cobertura)
4. ✅ `event:list` muestra todos los eventos registrados
5. ✅ `schedule:list` muestra el Job programado
6. ✅ PHPStan level 6 sin errores
7. ✅ Documentación EVENTS.md completa

---

## Notas Importantes

### ⚠️ Modificar Actions

Si las Actions no disparan eventos, los tests de integración **fallarán**. Es responsabilidad del **Agente B** agregar los `event()` calls en las Actions.

### 🎯 Sistema Completo

Esta tarea completa el sistema de eventos del módulo Auth:
- ✅ Eventos definidos (Task 01)
- ✅ Listeners implementados (Task 02)
- ✅ Jobs programados (Task 03)
- ✅ Integración validada (Task 04)

### 📊 Monitoring y Rate Limiting

Los logs con contexto `security_audit` permiten monitorear:
- Patrones de autenticación
- Intentos sospechosos de reset
- Volumen de actividad

**Rate Limiting**: Aunque los eventos en sí no tienen throttling (son síncronos y rápidos), los **endpoints** que disparan estos eventos SÍ deben tener rate limiting según `project_definition.md` sección 7:

```php
// En routes/api.php o similar
Route::middleware(['throttle:auth'])->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/password/reset/request', [PasswordResetController::class, 'request']);
    Route::post('/password/reset', [PasswordResetController::class, 'reset']);
});

// En RouteServiceProvider o config/auth.php
RateLimiter::for('auth', function (Request $request) {
    return Limit::perHour(5)->by($request->ip());
});
```

**Métricas de alerta recomendadas**:
- Si `MerchantAuthenticated` se dispara > 100 veces/hora desde misma IP → posible ataque
- Si `PasswordResetRequested` se dispara > 10 veces/hora para mismo email → posible abuso

---

## Referencias

- **agente-e-task-01**: Eventos del módulo Auth
- **agente-e-task-02**: Listeners de auditoría
- **agente-e-task-03**: Job de limpieza
- **Agente B**: Actions que disparan eventos
- **Laravel Events**: https://laravel.com/docs/events

---

**Última actualización**: 2025-12-18  
**Autor**: Alejandro Leone
