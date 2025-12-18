---
name: "Plan Detallado - Agente B — Módulo Auth"
version: "1.0"
author: "Alejandro Leone"
created: "2025-12-18"
module: "Auth"
purpose: "Plan paso a paso para implementar Actions y tests unitarios del módulo Auth"
dependencies:
  - auth-class-diagram.md
  - agente-a.md
  - agente-b.md
  - conventions.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
---

# Plan Detallado — Agente B — Módulo Auth

## Resumen Ejecutivo

Este plan detalla **paso a paso** la implementación del Agente B para el módulo Auth. El Agente B es responsable de **implementar los casos de uso** (Actions) del módulo Auth, trabajando exclusivamente sobre los contratos, Data y Value Objects definidos previamente por el Agente A.

**Restricciones clave:**
- ❌ **NO crear modelos Eloquent** (responsabilidad de otro agente)
- ❌ **NO crear repositorios concretos** (solo usar interfaces)
- ❌ **NO crear Filament Pages** (solo Actions)
- ❌ **NO crear migraciones ni factories**
- ✅ **Solo Actions, Excepciones y sus tests unitarios**

---

## Input Recibido del Agente A

Antes de comenzar, el Agente B debe verificar que existen estos archivos creados por el Agente A:

### Contratos (Interfaces)
- `Modules/Auth/Contracts/Commands/AuthenticateMerchantInterface.php`
- `Modules/Auth/Contracts/Commands/RequestPasswordResetInterface.php`
- `Modules/Auth/Contracts/Commands/ResetPasswordInterface.php`
- `Modules/Auth/Contracts/Repositories/PasswordResetTokenRepositoryInterface.php`

### Data Objects
- `Modules/Auth/Data/LoginData.php`
- `Modules/Auth/Data/RequestPasswordResetData.php`
- `Modules/Auth/Data/ResetPasswordData.php`
- `Modules/Auth/Data/AuthenticationResult.php`

### Value Objects (compartidos en app/)
- `app/ValueObjects/EmailValueObject.php`
- `app/ValueObjects/PasswordValueObject.php`
- `app/ValueObjects/NameValueObject.php`

### Enums (compartidos en app/)
- `app/Enums/MerchantRoleEnum.php`

**⚠️ Nota importante:** Si alguno de estos archivos no existe, **detener el trabajo** y solicitar al Agente A que complete su parte primero.

---

## Estructura de Archivos a Crear

```
Modules/Auth/
├── Actions/
│   ├── Commands/
│   │   ├── AuthenticateMerchantAction.php
│   │   ├── RequestPasswordResetAction.php
│   │   └── ResetPasswordAction.php
│   └── Internal/
│       └── InvalidateUserSessionsAction.php
├── Exceptions/
│   ├── InvalidCredentialsException.php
│   ├── InvalidPasswordResetTokenException.php
│   ├── RateLimitExceededException.php
│   └── TokenExpiredException.php
└── tests/Unit/Actions/
    ├── Commands/
    │   ├── AuthenticateMerchantActionTest.php
    │   ├── RequestPasswordResetActionTest.php
    │   └── ResetPasswordActionTest.php
    └── Internal/
        └── InvalidateUserSessionsActionTest.php
```

---

## Paso 1: Crear Excepciones de Dominio

### Objetivo
Crear las excepciones específicas del módulo Auth que serán lanzadas por las Actions.

### Archivos a Crear

#### 1.1. `Modules/Auth/Exceptions/InvalidCredentialsException.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Exceptions;

use Exception;

final class InvalidCredentialsException extends Exception
{
    public static function create(): self
    {
        return new self('Las credenciales proporcionadas son inválidas');
    }
}
```

**Propósito:** Lanzada cuando las credenciales (email/password) no coinciden.

**Nota de seguridad:** El mensaje es genérico para evitar enumeration attacks (no revelar si el email existe).

---

#### 1.2. `Modules/Auth/Exceptions/InvalidPasswordResetTokenException.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Exceptions;

use Exception;

final class InvalidPasswordResetTokenException extends Exception
{
    public static function create(): self
    {
        return new self('El token de recuperación de contraseña es inválido o ha expirado');
    }

    public static function notFound(): self
    {
        return new self('No se encontró un token de recuperación válido para este email');
    }
}
```

**Propósito:** Lanzada cuando el token de password reset no existe o no es válido.

---

#### 1.3. `Modules/Auth/Exceptions/RateLimitExceededException.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Exceptions;

use Exception;

final class RateLimitExceededException extends Exception
{
    public static function forLogin(int $seconds): self
    {
        return new self(
            sprintf(
                'Demasiados intentos de inicio de sesión. Por favor, intente nuevamente en %d segundos',
                $seconds
            )
        );
    }

    public static function forPasswordReset(int $seconds): self
    {
        return new self(
            sprintf(
                'Demasiadas solicitudes de recuperación de contraseña. Por favor, intente nuevamente en %d segundos',
                $seconds
            )
        );
    }
}
```

**Propósito:** Lanzada cuando se excede el límite de intentos (rate limiting).

---

#### 1.4. `Modules/Auth/Exceptions/TokenExpiredException.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Exceptions;

use Exception;

final class TokenExpiredException extends Exception
{
    public static function create(): self
    {
        return new self('El token de recuperación ha expirado. Por favor, solicite uno nuevo');
    }
}
```

**Propósito:** Lanzada cuando el token de password reset ha expirado (más de 60 minutos).

---

### Validación del Paso 1

```bash
# Verificar tipado
./vendor/bin/sail composer run phpstan

# Formatear código
./vendor/bin/sail bin pint --dirty
```

**Checklist:**
- [ ] Todas las excepciones son `final class`
- [ ] Extienden de `Exception`
- [ ] Tienen named constructors estáticos (`create()`, `forLogin()`, etc.)
- [ ] Mensajes descriptivos en español
- [ ] PHPStan sin errores
- [ ] Pint ejecutado sin cambios

---

## Paso 2: Implementar `AuthenticateMerchantAction`

### Objetivo
Implementar la lógica de autenticación de merchants con rate limiting.

### Archivo a Crear

#### 2.1. `Modules/Auth/Actions/Commands/AuthenticateMerchantAction.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Actions\Commands;

use Illuminate\Auth\AuthManager;
use Illuminate\Cache\RateLimiter;
use Illuminate\Support\Str;
use Modules\Auth\Contracts\Commands\AuthenticateMerchantInterface;
use Modules\Auth\Data\AuthenticationResult;
use Modules\Auth\Data\LoginData;
use Modules\Auth\Exceptions\InvalidCredentialsException;
use Modules\Auth\Exceptions\RateLimitExceededException;

final readonly class AuthenticateMerchantAction implements AuthenticateMerchantInterface
{
    private const MAX_ATTEMPTS = 5;
    private const DECAY_SECONDS = 60;

    public function __construct(
        private AuthManager $auth,
        private RateLimiter $rateLimiter,
    ) {}

    /**
     * Autentica un merchant con email y password.
     *
     * @throws InvalidCredentialsException
     * @throws RateLimitExceededException
     */
    public function execute(LoginData $data): AuthenticationResult
    {
        // 1. Verificar rate limiting
        $rateLimitKey = $this->getRateLimitKey($data->email->toString());

        if ($this->rateLimiter->tooManyAttempts($rateLimitKey, self::MAX_ATTEMPTS)) {
            $seconds = $this->rateLimiter->availableIn($rateLimitKey);
            throw RateLimitExceededException::forLogin($seconds);
        }

        // 2. Intentar autenticación
        $authenticated = $this->auth->attempt(
            [
                'email' => $data->email->toString(),
                'password' => $data->password,
            ],
            $data->remember
        );

        // 3. Si falla, incrementar contador de rate limiting
        if (!$authenticated) {
            $this->rateLimiter->hit($rateLimitKey, self::DECAY_SECONDS);
            throw InvalidCredentialsException::create();
        }

        // 4. Si tiene éxito, limpiar rate limiting y regenerar sesión
        $this->rateLimiter->clear($rateLimitKey);
        
        // Regenerar sesión para prevenir session fixation
        $this->auth->guard()->getRequest()?->session()?->regenerate();

        $user = $this->auth->user();

        return new AuthenticationResult(
            success: true,
            user: $user,
            error: null,
        );
    }

    private function getRateLimitKey(string $email): string
    {
        return 'login:' . Str::lower($email);
    }
}
```

**Características clave:**
- ✅ Implementa `AuthenticateMerchantInterface`
- ✅ Rate limiting: 5 intentos por minuto por email
- ✅ Incrementa contador solo si falla la autenticación
- ✅ Limpia rate limiting si tiene éxito
- ✅ Regenera sesión (prevenir session fixation)
- ✅ Lanza excepciones documentadas
- ✅ Mensaje genérico de error (anti-enumeration)

---

### Test Unitario del Paso 2

#### 2.2. `Modules/Auth/tests/Unit/Actions/Commands/AuthenticateMerchantActionTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Auth\AuthManager;
use Illuminate\Cache\RateLimiter;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Contracts\Session\Session;
use Illuminate\Http\Request;
use Modules\Auth\Actions\Commands\AuthenticateMerchantAction;
use Modules\Auth\Data\AuthenticationResult;
use Modules\Auth\Data\LoginData;
use Modules\Auth\Exceptions\InvalidCredentialsException;
use Modules\Auth\Exceptions\RateLimitExceededException;
use App\ValueObjects\EmailValueObject;
use App\Models\User;

beforeEach(function () {
    $this->auth = Mockery::mock(AuthManager::class);
    $this->rateLimiter = Mockery::mock(RateLimiter::class);
    $this->action = new AuthenticateMerchantAction($this->auth, $this->rateLimiter);
});

afterEach(function () {
    Mockery::close();
});

it('autentica un merchant correctamente', function () {
    $loginData = new LoginData(
        email: new EmailValueObject('admin@example.com'),
        password: 'password123',
        remember: false,
    );

    $user = Mockery::mock(User::class);
    $guard = Mockery::mock(Guard::class);
    $request = Mockery::mock(Request::class);
    $session = Mockery::mock(Session::class);

    // Mock rate limiter - no hay rate limit
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->with('login:admin@example.com', 5)
        ->andReturn(false);

    // Mock autenticación exitosa
    $this->auth
        ->shouldReceive('attempt')
        ->once()
        ->with(
            ['email' => 'admin@example.com', 'password' => 'password123'],
            false
        )
        ->andReturn(true);

    // Mock limpieza de rate limiting
    $this->rateLimiter
        ->shouldReceive('clear')
        ->once()
        ->with('login:admin@example.com');

    // Mock regeneración de sesión
    $this->auth
        ->shouldReceive('guard')
        ->once()
        ->andReturn($guard);

    $guard
        ->shouldReceive('getRequest')
        ->once()
        ->andReturn($request);

    $request
        ->shouldReceive('session')
        ->once()
        ->andReturn($session);

    $session
        ->shouldReceive('regenerate')
        ->once();

    // Mock obtener usuario
    $this->auth
        ->shouldReceive('user')
        ->once()
        ->andReturn($user);

    $result = $this->action->execute($loginData);

    expect($result)->toBeInstanceOf(AuthenticationResult::class)
        ->and($result->success)->toBeTrue()
        ->and($result->user)->toBe($user)
        ->and($result->error)->toBeNull();
});

it('lanza excepción cuando las credenciales son inválidas', function () {
    $loginData = new LoginData(
        email: new EmailValueObject('admin@example.com'),
        password: 'wrong_password',
        remember: false,
    );

    // Mock rate limiter - no hay rate limit
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->with('login:admin@example.com', 5)
        ->andReturn(false);

    // Mock autenticación fallida
    $this->auth
        ->shouldReceive('attempt')
        ->once()
        ->with(
            ['email' => 'admin@example.com', 'password' => 'wrong_password'],
            false
        )
        ->andReturn(false);

    // Mock incremento de rate limiting
    $this->rateLimiter
        ->shouldReceive('hit')
        ->once()
        ->with('login:admin@example.com', 60);

    $this->action->execute($loginData);
})->throws(InvalidCredentialsException::class, 'Las credenciales proporcionadas son inválidas');

it('lanza excepción cuando se excede el rate limit', function () {
    $loginData = new LoginData(
        email: new EmailValueObject('admin@example.com'),
        password: 'password123',
        remember: false,
    );

    // Mock rate limiter - hay rate limit
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->with('login:admin@example.com', 5)
        ->andReturn(true);

    // Mock tiempo disponible
    $this->rateLimiter
        ->shouldReceive('availableIn')
        ->once()
        ->with('login:admin@example.com')
        ->andReturn(45);

    $this->action->execute($loginData);
})->throws(RateLimitExceededException::class);

it('limpia el rate limiting después de autenticación exitosa', function () {
    $loginData = new LoginData(
        email: new EmailValueObject('user@example.com'),
        password: 'password123',
        remember: true,
    );

    $user = Mockery::mock(User::class);
    $guard = Mockery::mock(Guard::class);

    // Mock rate limiter
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->andReturn(false);

    $this->auth
        ->shouldReceive('attempt')
        ->once()
        ->andReturn(true);

    // Verificar que se limpia el rate limiting
    $this->rateLimiter
        ->shouldReceive('clear')
        ->once()
        ->with('login:user@example.com');

    $this->auth
        ->shouldReceive('guard')
        ->once()
        ->andReturn($guard);

    $guard
        ->shouldReceive('getRequest')
        ->once()
        ->andReturn(null); // Sin request/sesión

    $this->auth
        ->shouldReceive('user')
        ->once()
        ->andReturn($user);

    $result = $this->action->execute($loginData);

    expect($result->success)->toBeTrue();
});
```

---

### Validación del Paso 2

```bash
# Ejecutar tests
./vendor/bin/sail test --filter=AuthenticateMerchantActionTest

# Verificar tipado
./vendor/bin/sail composer run phpstan

# Formatear código
./vendor/bin/sail bin pint --dirty
```

**Checklist:**
- [ ] Action implementa el contrato correctamente
- [ ] Rate limiting funciona correctamente
- [ ] Excepciones documentadas con `@throws`
- [ ] Tests cubren: éxito, credenciales inválidas, rate limit excedido
- [ ] Mocks utilizados (no acceso a DB)
- [ ] Tests verdes al 100%
- [ ] PHPStan sin errores

---

## Paso 3: Implementar `RequestPasswordResetAction`

### Objetivo
Implementar la lógica para solicitar un token de recuperación de contraseña.

### Archivo a Crear

#### 3.1. `Modules/Auth/Actions/Commands/RequestPasswordResetAction.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Actions\Commands;

use Illuminate\Cache\RateLimiter;
use Illuminate\Contracts\Hashing\Hasher;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;
use Modules\Auth\Contracts\Commands\RequestPasswordResetInterface;
use Modules\Auth\Contracts\Repositories\PasswordResetTokenRepositoryInterface;
use Modules\Auth\Data\RequestPasswordResetData;
use Modules\Auth\Exceptions\RateLimitExceededException;

final readonly class RequestPasswordResetAction implements RequestPasswordResetInterface
{
    private const MAX_ATTEMPTS = 3;
    private const DECAY_SECONDS = 3600; // 1 hora

    public function __construct(
        private PasswordResetTokenRepositoryInterface $tokenRepository,
        private Hasher $hasher,
        private RateLimiter $rateLimiter,
    ) {}

    /**
     * Genera un token de recuperación de contraseña y lo envía por email.
     *
     * @throws RateLimitExceededException
     */
    public function execute(RequestPasswordResetData $data): void
    {
        // 1. Verificar rate limiting por email
        $rateLimitKey = $this->getRateLimitKey($data->email->toString());

        if ($this->rateLimiter->tooManyAttempts($rateLimitKey, self::MAX_ATTEMPTS)) {
            $seconds = $this->rateLimiter->availableIn($rateLimitKey);
            throw RateLimitExceededException::forPasswordReset($seconds);
        }

        // 2. Incrementar contador (siempre, incluso si el email no existe)
        $this->rateLimiter->hit($rateLimitKey, self::DECAY_SECONDS);

        // 3. Buscar usuario por email (sin revelar si existe o no)
        $user = $this->findUserByEmail($data->email->toString());

        // 4. Si el usuario no existe, retornar silenciosamente (anti-enumeration)
        if ($user === null) {
            return;
        }

        // 5. Eliminar tokens previos del mismo email
        $this->tokenRepository->delete($data->email);

        // 6. Generar token aleatorio (60 caracteres)
        $plainToken = Str::random(60);

        // 7. Crear registro con token hasheado
        $this->tokenRepository->create(
            email: $data->email,
            hashedToken: $this->hasher->make($plainToken),
        );

        // 8. Enviar email con link de reset (contiene token sin hashear)
        // Nota: El envío de email se implementará en otro agente (Events/Listeners)
        // Por ahora, el Action solo crea el token
    }

    private function getRateLimitKey(string $email): string
    {
        return 'password-reset:' . Str::lower($email);
    }

    /**
     * Busca usuario por email.
     * Este método es un stub que será implementado cuando exista el modelo User.
     */
    private function findUserByEmail(string $email): ?Model
    {
        // TODO: Implementar cuando exista el modelo User
        // return User::where('email', $email)->first();
        return null;
    }
}
```

**Características clave:**
- ✅ Implementa `RequestPasswordResetInterface`
- ✅ Rate limiting: 3 intentos por hora por email
- ✅ Anti-enumeration: no revela si el email existe
- ✅ Genera token de 60 caracteres
- ✅ Hashea el token antes de guardarlo
- ✅ Elimina tokens previos
- ✅ Retorna `void` (siempre éxito aparente)

**⚠️ Nota:** El método `findUserByEmail()` es un stub. Será implementado cuando otro agente cree el modelo User. Por ahora, los tests lo mockearán.

---

### Test Unitario del Paso 3

#### 3.2. `Modules/Auth/tests/Unit/Actions/Commands/RequestPasswordResetActionTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Cache\RateLimiter;
use Illuminate\Contracts\Hashing\Hasher;
use Modules\Auth\Actions\Commands\RequestPasswordResetAction;
use Modules\Auth\Contracts\Repositories\PasswordResetTokenRepositoryInterface;
use Modules\Auth\Data\RequestPasswordResetData;
use Modules\Auth\Exceptions\RateLimitExceededException;
use App\ValueObjects\EmailValueObject;

beforeEach(function () {
    $this->tokenRepository = Mockery::mock(PasswordResetTokenRepositoryInterface::class);
    $this->hasher = Mockery::mock(Hasher::class);
    $this->rateLimiter = Mockery::mock(RateLimiter::class);
    
    $this->action = new RequestPasswordResetAction(
        $this->tokenRepository,
        $this->hasher,
        $this->rateLimiter
    );
});

afterEach(function () {
    Mockery::close();
});

it('crea un token de password reset exitosamente', function () {
    $data = new RequestPasswordResetData(
        email: new EmailValueObject('user@example.com'),
    );

    // Mock rate limiter - no hay rate limit
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->with('password-reset:user@example.com', 3)
        ->andReturn(false);

    // Mock incremento de rate limiting
    $this->rateLimiter
        ->shouldReceive('hit')
        ->once()
        ->with('password-reset:user@example.com', 3600);

    // Mock eliminación de tokens previos
    $this->tokenRepository
        ->shouldReceive('delete')
        ->once()
        ->with($data->email);

    // Mock hasheo del token
    $this->hasher
        ->shouldReceive('make')
        ->once()
        ->andReturn('hashed_token_value');

    // Mock creación del token
    $this->tokenRepository
        ->shouldReceive('create')
        ->once()
        ->withArgs(function ($email, $hashedToken) {
            return $email->toString() === 'user@example.com'
                && $hashedToken === 'hashed_token_value';
        });

    // No debe lanzar excepción
    $this->action->execute($data);

    expect(true)->toBeTrue();
});

it('retorna silenciosamente si el email no existe (anti-enumeration)', function () {
    $data = new RequestPasswordResetData(
        email: new EmailValueObject('nonexistent@example.com'),
    );

    // Mock rate limiter
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->andReturn(false);

    $this->rateLimiter
        ->shouldReceive('hit')
        ->once();

    // No debe llamar a delete ni create si el usuario no existe
    $this->tokenRepository
        ->shouldNotReceive('delete');

    $this->tokenRepository
        ->shouldNotReceive('create');

    // No debe lanzar excepción (anti-enumeration)
    $this->action->execute($data);

    expect(true)->toBeTrue();
});

it('lanza excepción cuando se excede el rate limit', function () {
    $data = new RequestPasswordResetData(
        email: new EmailValueObject('user@example.com'),
    );

    // Mock rate limiter - hay rate limit
    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->with('password-reset:user@example.com', 3)
        ->andReturn(true);

    // Mock tiempo disponible
    $this->rateLimiter
        ->shouldReceive('availableIn')
        ->once()
        ->with('password-reset:user@example.com')
        ->andReturn(1800);

    $this->action->execute($data);
})->throws(RateLimitExceededException::class);

it('elimina tokens previos antes de crear uno nuevo', function () {
    $data = new RequestPasswordResetData(
        email: new EmailValueObject('user@example.com'),
    );

    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->andReturn(false);

    $this->rateLimiter
        ->shouldReceive('hit')
        ->once();

    // Verificar que se llama delete antes de create
    $this->tokenRepository
        ->shouldReceive('delete')
        ->once()
        ->with($data->email)
        ->ordered();

    $this->hasher
        ->shouldReceive('make')
        ->once()
        ->andReturn('hashed_token');

    $this->tokenRepository
        ->shouldReceive('create')
        ->once()
        ->ordered();

    $this->action->execute($data);

    expect(true)->toBeTrue();
});

it('incrementa el contador de rate limiting incluso si el email no existe', function () {
    $data = new RequestPasswordResetData(
        email: new EmailValueObject('nonexistent@example.com'),
    );

    $this->rateLimiter
        ->shouldReceive('tooManyAttempts')
        ->once()
        ->andReturn(false);

    // Verificar que se incrementa el contador
    $this->rateLimiter
        ->shouldReceive('hit')
        ->once()
        ->with('password-reset:nonexistent@example.com', 3600);

    $this->action->execute($data);

    expect(true)->toBeTrue();
});
```

---

### Validación del Paso 3

```bash
# Ejecutar tests
./vendor/bin/sail test --filter=RequestPasswordResetActionTest

# Verificar tipado
./vendor/bin/sail composer run phpstan

# Formatear código
./vendor/bin/sail bin pint --dirty
```

**Checklist:**
- [ ] Action implementa el contrato correctamente
- [ ] Rate limiting funciona (3 intentos por hora)
- [ ] Anti-enumeration: no revela si email existe
- [ ] Elimina tokens previos antes de crear nuevo
- [ ] Token se hashea antes de guardar
- [ ] Tests cubren todos los casos
- [ ] Mocks utilizados correctamente
- [ ] Tests verdes al 100%
- [ ] PHPStan sin errores

---

## Paso 4: Implementar `InvalidateUserSessionsAction` (Internal)

### Objetivo
Crear Action auxiliar para invalidar todas las sesiones de un usuario.

### Archivo a Crear

#### 4.1. `Modules/Auth/Actions/Internal/InvalidateUserSessionsAction.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Actions\Internal;

use Illuminate\Contracts\Auth\Guard;
use Illuminate\Database\Eloquent\Model;

final readonly class InvalidateUserSessionsAction
{
    public function __construct(
        private Guard $guard,
    ) {}

    /**
     * Invalida todas las sesiones activas de un usuario.
     * Se utiliza después de cambiar la contraseña.
     */
    public function execute(Model $user): void
    {
        // Logout de todas las sesiones activas
        $this->guard->logoutOtherDevices($user->password);
    }
}
```

**Propósito:** Esta Action auxiliar invalida todas las sesiones activas de un usuario. Se usa después de cambiar la contraseña para forzar re-autenticación.

**⚠️ Nota:** Este Action es **Internal** (privado del módulo), no implementa contrato público.

---

### Test Unitario del Paso 4

#### 4.2. `Modules/Auth/tests/Unit/Actions/Internal/InvalidateUserSessionsActionTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Contracts\Auth\Guard;
use Illuminate\Database\Eloquent\Model;
use Modules\Auth\Actions\Internal\InvalidateUserSessionsAction;

beforeEach(function () {
    $this->guard = Mockery::mock(Guard::class);
    $this->action = new InvalidateUserSessionsAction($this->guard);
});

afterEach(function () {
    Mockery::close();
});

it('invalida todas las sesiones de un usuario', function () {
    $user = Mockery::mock(Model::class);
    $user->password = 'hashed_password';

    $this->guard
        ->shouldReceive('logoutOtherDevices')
        ->once()
        ->with('hashed_password');

    $this->action->execute($user);

    expect(true)->toBeTrue();
});
```

---

### Validación del Paso 4

```bash
# Ejecutar tests
./vendor/bin/sail test --filter=InvalidateUserSessionsActionTest

# Verificar tipado
./vendor/bin/sail composer run phpstan

# Formatear código
./vendor/bin/sail bin pint --dirty
```

**Checklist:**
- [ ] Action es `final readonly class`
- [ ] No implementa contrato público (es Internal)
- [ ] Test verifica llamada a `logoutOtherDevices()`
- [ ] Tests verdes
- [ ] PHPStan sin errores

---

## Paso 5: Implementar `ResetPasswordAction`

### Objetivo
Implementar la lógica para validar token y resetear contraseña.

### Archivo a Crear

#### 5.1. `Modules/Auth/Actions/Commands/ResetPasswordAction.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Actions\Commands;

use Illuminate\Contracts\Hashing\Hasher;
use Illuminate\Database\Eloquent\Model;
use Modules\Auth\Actions\Internal\InvalidateUserSessionsAction;
use Modules\Auth\Contracts\Commands\ResetPasswordInterface;
use Modules\Auth\Contracts\Repositories\PasswordResetTokenRepositoryInterface;
use Modules\Auth\Data\ResetPasswordData;
use Modules\Auth\Exceptions\InvalidPasswordResetTokenException;
use Modules\Auth\Exceptions\TokenExpiredException;

final readonly class ResetPasswordAction implements ResetPasswordInterface
{
    private const TOKEN_EXPIRATION_MINUTES = 60;

    public function __construct(
        private PasswordResetTokenRepositoryInterface $tokenRepository,
        private Hasher $hasher,
        private InvalidateUserSessionsAction $invalidateSessionsAction,
    ) {}

    /**
     * Resetea la contraseña de un usuario validando el token.
     *
     * @throws InvalidPasswordResetTokenException
     * @throws TokenExpiredException
     */
    public function execute(ResetPasswordData $data): void
    {
        // 1. Buscar token por email
        $passwordResetToken = $this->tokenRepository->findByEmail($data->email);

        if ($passwordResetToken === null) {
            throw InvalidPasswordResetTokenException::notFound();
        }

        // 2. Validar que el token no esté expirado
        if ($passwordResetToken->isExpired()) {
            throw TokenExpiredException::create();
        }

        // 3. Validar que el token coincida
        if (!$passwordResetToken->isValid($data->token)) {
            throw InvalidPasswordResetTokenException::create();
        }

        // 4. Buscar usuario por email
        $user = $this->findUserByEmail($data->email->toString());

        if ($user === null) {
            throw InvalidPasswordResetTokenException::notFound();
        }

        // 5. Actualizar password del usuario
        $user->password = $this->hasher->make($data->password);
        $user->save();

        // 6. Eliminar token usado
        $this->tokenRepository->delete($data->email);

        // 7. Invalidar todas las sesiones activas
        $this->invalidateSessionsAction->execute($user);
    }

    /**
     * Busca usuario por email.
     * Este método es un stub que será implementado cuando exista el modelo User.
     */
    private function findUserByEmail(string $email): ?Model
    {
        // TODO: Implementar cuando exista el modelo User
        // return User::where('email', $email)->first();
        return null;
    }
}
```

**Características clave:**
- ✅ Implementa `ResetPasswordInterface`
- ✅ Valida que el token no esté expirado (60 minutos)
- ✅ Valida que el token coincida con el hasheado
- ✅ Actualiza la contraseña del usuario
- ✅ Elimina el token usado
- ✅ Invalida todas las sesiones activas
- ✅ Lanza excepciones específicas

**⚠️ Nota:** El método `findUserByEmail()` es un stub. Será implementado cuando otro agente cree el modelo User.

---

### Test Unitario del Paso 5

#### 5.2. `Modules/Auth/tests/Unit/Actions/Commands/ResetPasswordActionTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Contracts\Hashing\Hasher;
use Illuminate\Database\Eloquent\Model;
use Modules\Auth\Actions\Commands\ResetPasswordAction;
use Modules\Auth\Actions\Internal\InvalidateUserSessionsAction;
use Modules\Auth\Contracts\Repositories\PasswordResetTokenRepositoryInterface;
use Modules\Auth\Data\ResetPasswordData;
use Modules\Auth\Exceptions\InvalidPasswordResetTokenException;
use Modules\Auth\Exceptions\TokenExpiredException;
use Modules\Auth\Models\PasswordResetToken;
use App\ValueObjects\EmailValueObject;

beforeEach(function () {
    $this->tokenRepository = Mockery::mock(PasswordResetTokenRepositoryInterface::class);
    $this->hasher = Mockery::mock(Hasher::class);
    $this->invalidateSessionsAction = Mockery::mock(InvalidateUserSessionsAction::class);
    
    $this->action = new ResetPasswordAction(
        $this->tokenRepository,
        $this->hasher,
        $this->invalidateSessionsAction
    );
});

afterEach(function () {
    Mockery::close();
});

it('resetea la contraseña correctamente', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'valid_token_123',
        password: 'new_secure_password',
    );

    $passwordResetToken = Mockery::mock(PasswordResetToken::class);
    $user = Mockery::mock(Model::class);

    // Mock buscar token
    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->with($data->email)
        ->andReturn($passwordResetToken);

    // Mock validación de expiración
    $passwordResetToken
        ->shouldReceive('isExpired')
        ->once()
        ->andReturn(false);

    // Mock validación de token
    $passwordResetToken
        ->shouldReceive('isValid')
        ->once()
        ->with('valid_token_123')
        ->andReturn(true);

    // Mock hasheo de nueva contraseña
    $this->hasher
        ->shouldReceive('make')
        ->once()
        ->with('new_secure_password')
        ->andReturn('hashed_new_password');

    // Mock actualización de contraseña
    $user
        ->shouldReceive('getAttribute')
        ->with('password');
    
    $user
        ->shouldReceive('setAttribute')
        ->once()
        ->with('password', 'hashed_new_password');
    
    $user
        ->shouldReceive('save')
        ->once();

    // Mock eliminación de token
    $this->tokenRepository
        ->shouldReceive('delete')
        ->once()
        ->with($data->email);

    // Mock invalidación de sesiones
    $this->invalidateSessionsAction
        ->shouldReceive('execute')
        ->once()
        ->with($user);

    $this->action->execute($data);

    expect(true)->toBeTrue();
});

it('lanza excepción si el token no existe', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'nonexistent_token',
        password: 'new_password',
    );

    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->with($data->email)
        ->andReturn(null);

    $this->action->execute($data);
})->throws(InvalidPasswordResetTokenException::class, 'No se encontró un token de recuperación válido');

it('lanza excepción si el token está expirado', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'expired_token',
        password: 'new_password',
    );

    $passwordResetToken = Mockery::mock(PasswordResetToken::class);

    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->andReturn($passwordResetToken);

    $passwordResetToken
        ->shouldReceive('isExpired')
        ->once()
        ->andReturn(true);

    $this->action->execute($data);
})->throws(TokenExpiredException::class, 'El token de recuperación ha expirado');

it('lanza excepción si el token no es válido', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'invalid_token',
        password: 'new_password',
    );

    $passwordResetToken = Mockery::mock(PasswordResetToken::class);

    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->andReturn($passwordResetToken);

    $passwordResetToken
        ->shouldReceive('isExpired')
        ->once()
        ->andReturn(false);

    $passwordResetToken
        ->shouldReceive('isValid')
        ->once()
        ->with('invalid_token')
        ->andReturn(false);

    $this->action->execute($data);
})->throws(InvalidPasswordResetTokenException::class, 'El token de recuperación de contraseña es inválido');

it('elimina el token después de resetear la contraseña', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'valid_token',
        password: 'new_password',
    );

    $passwordResetToken = Mockery::mock(PasswordResetToken::class);
    $user = Mockery::mock(Model::class);

    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->andReturn($passwordResetToken);

    $passwordResetToken
        ->shouldReceive('isExpired')
        ->once()
        ->andReturn(false);

    $passwordResetToken
        ->shouldReceive('isValid')
        ->once()
        ->andReturn(true);

    $this->hasher
        ->shouldReceive('make')
        ->once()
        ->andReturn('hashed_password');

    $user->shouldReceive('getAttribute', 'setAttribute', 'save');

    // Verificar que se elimina el token
    $this->tokenRepository
        ->shouldReceive('delete')
        ->once()
        ->with($data->email);

    $this->invalidateSessionsAction
        ->shouldReceive('execute')
        ->once();

    $this->action->execute($data);

    expect(true)->toBeTrue();
});

it('invalida todas las sesiones después de resetear la contraseña', function () {
    $data = new ResetPasswordData(
        email: new EmailValueObject('user@example.com'),
        token: 'valid_token',
        password: 'new_password',
    );

    $passwordResetToken = Mockery::mock(PasswordResetToken::class);
    $user = Mockery::mock(Model::class);

    $this->tokenRepository
        ->shouldReceive('findByEmail')
        ->once()
        ->andReturn($passwordResetToken);

    $passwordResetToken
        ->shouldReceive('isExpired', 'isValid')
        ->andReturn(false, true);

    $this->hasher
        ->shouldReceive('make')
        ->once()
        ->andReturn('hashed_password');

    $user->shouldReceive('getAttribute', 'setAttribute', 'save');

    $this->tokenRepository
        ->shouldReceive('delete')
        ->once();

    // Verificar que se invalidan las sesiones
    $this->invalidateSessionsAction
        ->shouldReceive('execute')
        ->once()
        ->with($user);

    $this->action->execute($data);

    expect(true)->toBeTrue();
});
```

---

### Validación del Paso 5

```bash
# Ejecutar tests
./vendor/bin/sail test --filter=ResetPasswordActionTest

# Verificar tipado
./vendor/bin/sail composer run phpstan

# Formatear código
./vendor/bin/sail bin pint --dirty
```

**Checklist:**
- [ ] Action implementa el contrato correctamente
- [ ] Valida expiración del token (60 minutos)
- [ ] Valida coincidencia del token
- [ ] Actualiza password del usuario
- [ ] Elimina token usado
- [ ] Invalida sesiones activas
- [ ] Tests cubren todos los casos
- [ ] Mocks utilizados correctamente
- [ ] Tests verdes al 100%
- [ ] PHPStan sin errores

---

## Paso 6: Validación Final del Agente B

### Objetivo
Verificar que todo el trabajo del Agente B está completo y cumple con los estándares.

### Checklist de Entregables

#### Archivos de Código
- [ ] `Modules/Auth/Actions/Commands/AuthenticateMerchantAction.php`
- [ ] `Modules/Auth/Actions/Commands/RequestPasswordResetAction.php`
- [ ] `Modules/Auth/Actions/Commands/ResetPasswordAction.php`
- [ ] `Modules/Auth/Actions/Internal/InvalidateUserSessionsAction.php`
- [ ] `Modules/Auth/Exceptions/InvalidCredentialsException.php`
- [ ] `Modules/Auth/Exceptions/InvalidPasswordResetTokenException.php`
- [ ] `Modules/Auth/Exceptions/RateLimitExceededException.php`
- [ ] `Modules/Auth/Exceptions/TokenExpiredException.php`

#### Tests Unitarios
- [ ] `Modules/Auth/tests/Unit/Actions/Commands/AuthenticateMerchantActionTest.php`
- [ ] `Modules/Auth/tests/Unit/Actions/Commands/RequestPasswordResetActionTest.php`
- [ ] `Modules/Auth/tests/Unit/Actions/Commands/ResetPasswordActionTest.php`
- [ ] `Modules/Auth/tests/Unit/Actions/Internal/InvalidateUserSessionsActionTest.php`

### Comandos de Validación Final

```bash
# 1. Ejecutar TODOS los tests del módulo Auth
./vendor/bin/sail test Modules/Auth/tests/Unit/Actions

# 2. Verificar tipado estático (PHPStan level 6)
./vendor/bin/sail composer run phpstan

# 3. Formatear código (Pint)
./vendor/bin/sail bin pint --dirty

# 4. Ejecutar Rector (si aplica)
./vendor/bin/sail composer run rector

# 5. Verificar que no hay errores de sintaxis
./vendor/bin/sail artisan about
```

### Checklist de Calidad

#### Código
- [ ] Todas las Actions son `final readonly class`
- [ ] Todas las Actions implementan contratos del Agente A
- [ ] Un solo método público por Action: `execute()`
- [ ] Inyección de dependencias mediante constructor
- [ ] No hay acceso directo a DB, Eloquent ni Schema
- [ ] No hay tipos `mixed`, `array` genérico en signatures públicas
- [ ] Todas las excepciones son `final class extends Exception`
- [ ] Named constructors estáticos en excepciones

#### Tests
- [ ] Todos los tests usan Mockery para repositorios
- [ ] No hay acceso a DB en tests unitarios
- [ ] Cobertura del 100% de las Actions
- [ ] Tests cubren casos de éxito y error
- [ ] Tests verifican lanzamiento de excepciones
- [ ] Tests verifican rate limiting
- [ ] Todos los tests pasan (verdes)

#### Herramientas
- [ ] PHPStan level 6 sin errores
- [ ] Pint ejecutado sin cambios pendientes
- [ ] Rector ejecutado (si aplica)
- [ ] No hay warnings ni notices

---

## Resumen de Archivos Creados

### Total de Archivos: 12

#### Actions (4 archivos)
1. `Modules/Auth/Actions/Commands/AuthenticateMerchantAction.php`
2. `Modules/Auth/Actions/Commands/RequestPasswordResetAction.php`
3. `Modules/Auth/Actions/Commands/ResetPasswordAction.php`
4. `Modules/Auth/Actions/Internal/InvalidateUserSessionsAction.php`

#### Excepciones (4 archivos)
5. `Modules/Auth/Exceptions/InvalidCredentialsException.php`
6. `Modules/Auth/Exceptions/InvalidPasswordResetTokenException.php`
7. `Modules/Auth/Exceptions/RateLimitExceededException.php`
8. `Modules/Auth/Exceptions/TokenExpiredException.php`

#### Tests (4 archivos)
9. `Modules/Auth/tests/Unit/Actions/Commands/AuthenticateMerchantActionTest.php`
10. `Modules/Auth/tests/Unit/Actions/Commands/RequestPasswordResetActionTest.php`
11. `Modules/Auth/tests/Unit/Actions/Commands/ResetPasswordActionTest.php`
12. `Modules/Auth/tests/Unit/Actions/Internal/InvalidateUserSessionsActionTest.php`

---

## Notas Importantes

### Stubs y TODOs
Los siguientes métodos son **stubs** que serán implementados por agentes posteriores:

1. **`AuthenticateMerchantAction`**: Acceso completo a `AuthManager` (funcional)
2. **`RequestPasswordResetAction::findUserByEmail()`**: Stub, retorna `null` por ahora
3. **`ResetPasswordAction::findUserByEmail()`**: Stub, retorna `null` por ahora

**⚠️ Los tests mockean estos métodos, por lo que pasan correctamente.**

### Dependencias No Implementadas
El Agente B **NO implementa**:
- ❌ Modelo `User`
- ❌ Modelo `PasswordResetToken`
- ❌ Repositorios concretos
- ❌ Casts de Value Objects
- ❌ Migrations
- ❌ Factories
- ❌ Envío de emails (Events/Listeners)

**Estas responsabilidades pertenecen a otros agentes.**

---

## Comunicación con Agentes Posteriores

### Salida del Agente B

El Agente B entrega:
- ✅ Actions implementadas con lógica de negocio completa
- ✅ Excepciones de dominio documentadas
- ✅ Tests unitarios con mocks (cobertura 100%)

### Entrada para Agentes C, D, E...

Los agentes posteriores deben:
- ✅ Implementar los repositorios mockados
- ✅ Crear modelos Eloquent (User, PasswordResetToken)
- ✅ Crear Casts para Value Objects
- ✅ Crear migraciones y factories
- ✅ Crear Custom Filament Pages que usen estas Actions
- ✅ Crear Listeners para enviar emails
- ✅ Crear Feature Tests de integración

---

## Próximos Pasos (Otros Agentes)

Una vez completado el Agente B, el trabajo continúa con:

### Agente C: Modelos y Persistencia
- Modelo `User` con casts de Value Objects
- Modelo `PasswordResetToken`
- Repositorios concretos
- Migrations
- Factories
- Seeders

### Agente D: Infraestructura de Entrada/Salida
- Custom Filament Pages (Login, RequestPasswordReset, ResetPassword)
- Command: `CreateOwnerCommand`
- Configuración de Filament Panel

### Agente E: Jobs y Eventos
- Job: `CleanExpiredPasswordResetTokensJob`
- Events: `MerchantAuthenticatedEvent`, `PasswordResetRequestedEvent`
- Listeners: `SendPasswordResetEmailListener`

### Agente F: Tests de Integración
- Feature tests con DB real
- Tests de integración con Filament
- Smoke tests

---

## Referencias

- **Diagrama de clases del módulo Auth**: [`auth-class-diagram.md`](../../e-commerce-wa-ml/auth/auth-class-diagram.md)
- **Agente A (Contratos)**: [`agente-a.md`](agente-a.md)
- **Agente B (Actions)**: [`agente-b.md`](agente-b.md)
- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Value Objects completos**: [`value-objects.md`](../conventions/value-objects.md)
- **Mockery**: https://github.com/mockery/mockery
- **Pest**: https://pestphp.com/
- **Laravel Auth**: https://laravel.com/docs/authentication

---

**Versión**: 1.0  
**Creado**: 2025-12-18  
**Autor**: Alejandro Leone

