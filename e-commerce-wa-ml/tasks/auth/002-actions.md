---
task_id: "auth-002-actions"
module: "Auth"
agent: "Agente B - Actions y Tests Unitarios"
title: "Auth - Actions de Autenticación y Lógica de Negocio"
priority: "HIGH"
estimated_time: "6 hours"
dependencies:
  - "auth-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-b-actions.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
  - "@laravel/conventions/conventions.md"
phase: "Fase 2 - Lógica de Negocio"
---

# Task 002: Auth - Actions de Autenticación y Lógica de Negocio

## 🎯 Objetivo

Implementar las Actions del módulo Auth que encapsulan la lógica de negocio de autenticación, validación de credenciales y gestión de sesiones. Estas Actions son el núcleo funcional del módulo y están completamente desacopladas de la capa de presentación.

## 📋 Contexto

Las Actions implementan los casos de uso del módulo Auth siguiendo el patrón Command/Query. Cada Action tiene una única responsabilidad y es testeable de forma unitaria mediante mocks.

### Referencias del Domain Model
- **Actions:** AuthenticateMerchantAction (líneas 251-287), LogoutMerchantAction (líneas 289-317), ValidateCredentialsAction (líneas 319-343)
- **Business Rules:** Authentication (líneas 410-434)
- **Security:** Rate limiting, session management (líneas 520-590)

## 📦 Artefactos a Crear

### 1. Interfaces de Comunicación (Contratos para Actions)

**Ubicación:** `Modules/Auth/Contracts/Commands/`

**Especificaciones:**

```php
namespace Modules\Auth\Contracts\Commands;

use Modules\Auth\Data\AuthenticateData;
use Modules\Auth\Data\AuthResult;

interface AuthenticateMerchantInterface
{
    /**
     * Ejecuta la autenticación de un merchant
     */
    public function execute(AuthenticateData $data): AuthResult;
}
```

```php
namespace Modules\Auth\Contracts\Commands;

use App\Models\User;

interface LogoutMerchantInterface
{
    /**
     * Ejecuta el logout de un merchant autenticado
     */
    public function execute(User $user): void;
}
```

```php
namespace Modules\Auth\Contracts\Queries;

interface ValidateCredentialsInterface
{
    /**
     * Valida credenciales sin crear sesión
     */
    public function execute(string $email, string $password): bool;
}
```

**Reglas de Negocio:**
- Interfaces obligatorias para comunicación entre módulos
- Garantizan desacoplamiento y permiten mockeo eficiente
- Facilitan testing unitario sin dependencias concretas

---

### 2. Action Command: AuthenticateMerchantAction

**Ubicación:** `Modules/Auth/Actions/Commands/AuthenticateMerchantAction.php`

**Especificaciones:**

```php
namespace Modules\Auth\Actions\Commands;

use Modules\Auth\Contracts\Commands\AuthenticateMerchantInterface;
use Modules\Auth\Contracts\Repositories\MerchantRepositoryInterface;
use Modules\Auth\Data\AuthenticateData;
use Modules\Auth\Data\AuthResult;
use Modules\Auth\Events\UserLoginEvent;
use Modules\Auth\Exceptions\InvalidCredentialsException;
use Modules\Auth\ValueObjects\Email;
use Modules\Auth\ValueObjects\HashedPassword;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Str;

final readonly class AuthenticateMerchantAction implements AuthenticateMerchantInterface
{
    public function __construct(
        private MerchantRepositoryInterface $merchantRepository,
    ) {}

    /**
     * Ejecuta la autenticación de un merchant
     *
     * @param AuthenticateData $data Datos de autenticación
     * @return AuthResult Resultado de la autenticación
     * @throws InvalidCredentialsException Si las credenciales son inválidas
     */
    public function execute(AuthenticateData $data): AuthResult
    {
        // 1. Crear Email VO desde string (valida formato)
        $email = Email::fromString($data->email);
        
        // 2. Buscar usuario por email normalizado (mediante repositorio)
        $merchantData = $this->merchantRepository->findByEmail($email);
        
        // 3. Si usuario no existe, retornar error genérico (prevenir enumeración)
        if ($merchantData === null) {
            return AuthResult::failure('Credenciales inválidas');
        }
        
        // 4. Verificar contraseña usando HashedPassword VO
        if (!$merchantData->password->verify($data->password)) {
            return AuthResult::failure('Credenciales inválidas');
        }
        
        // 5. Crear sesión (Auth facade recibe ID del merchant)
        Auth::loginUsingId($merchantData->id->value(), $data->remember);
        
        // 6. Si remember=true, generar y guardar remember_token mediante repositorio
        if ($data->remember) {
            $rememberToken = Str::random(60);
            $this->merchantRepository->updateRememberToken($merchantData->id, $rememberToken);
        }
        
        // 7. Disparar evento UserLoginEvent
        event(new UserLoginEvent(
            user_id: $merchantData->id->value(),
            email: $email,
            ip_address: request()->ip(),
            logged_in_at: now()
        ));
        
        // 8. Retornar AuthResult exitoso
        return AuthResult::success($merchantData, 'Bienvenido de vuelta');
    }
}
```

**Reglas de Negocio:**
- Email VO valida formato automáticamente en constructor
- Contraseña no puede estar vacía
- Usuario debe existir en la base de datos
- **Verificación de password mediante HashedPassword VO (NO Hash::check() directo)**
- Mensaje de error genérico para prevenir enumeración de usuarios
- Remember token generado solo si remember=true
- Evento UserLoginEvent disparado después de login exitoso
- Session regeneration para prevenir session fixation
- **Comunicación mediante MerchantRepositoryInterface (NO acceso directo a Eloquent)**

---

### 3. Action Command: LogoutMerchantAction

**Ubicación:** `Modules/Auth/Actions/Commands/LogoutMerchantAction.php`

**Especificaciones:**

```php
namespace Modules\Auth\Actions\Commands;

use Modules\Auth\Contracts\Commands\LogoutMerchantInterface;
use Modules\Auth\Contracts\Repositories\MerchantRepositoryInterface;
use Modules\Auth\Events\UserLogoutEvent;
use Modules\Auth\ValueObjects\Email;
use Modules\Auth\ValueObjects\MerchantId;
use App\Models\User;
use Illuminate\Support\Facades\Auth;

final readonly class LogoutMerchantAction implements LogoutMerchantInterface
{
    public function __construct(
        private MerchantRepositoryInterface $merchantRepository,
    ) {}

    /**
     * Ejecuta el logout de un merchant autenticado
     *
     * @param User $user Usuario autenticado
     * @return void
     */
    public function execute(User $user): void
    {
        // 1. Capturar datos antes de invalidar sesión
        $merchantId = MerchantId::fromString($user->id);
        $email = $user->email; // Ya es Email VO por el Cast
        
        // 2. Revocar remember_token mediante repositorio
        $this->merchantRepository->updateRememberToken($merchantId, null);
        
        // 3. Disparar evento UserLogoutEvent
        event(new UserLogoutEvent(
            user_id: $merchantId->value(),
            email: $email,
            logged_out_at: now()
        ));
        
        // 4. Invalidar sesión actual (Laravel lo maneja automáticamente)
        Auth::logout();
        
        // 5. Invalidar sesión y regenerar token CSRF
        request()->session()->invalidate();
        request()->session()->regenerateToken();
    }
}
```

**Reglas de Negocio:**
- Usuario debe estar autenticado
- Remember token se revoca (set to null) **mediante repositorio**
- Sesión se invalida completamente
- Token CSRF se regenera para seguridad
- Evento UserLogoutEvent disparado antes de invalidar sesión
- Cookies de sesión limpiadas automáticamente por Laravel
- **NO acceso directo a Eloquent - usar MerchantRepositoryInterface**

---

### 4. Action Query: ValidateCredentialsAction

**Ubicación:** `Modules/Auth/Actions/Queries/ValidateCredentialsAction.php`

**Especificaciones:**

```php
namespace Modules\Auth\Actions\Queries;

use Modules\Auth\Contracts\Queries\ValidateCredentialsInterface;
use Modules\Auth\Contracts\Repositories\MerchantRepositoryInterface;
use Modules\Auth\ValueObjects\Email;

final readonly class ValidateCredentialsAction implements ValidateCredentialsInterface
{
    public function __construct(
        private MerchantRepositoryInterface $merchantRepository,
    ) {}

    /**
     * Valida credenciales sin crear sesión
     *
     * @param string $email Email del usuario
     * @param string $password Contraseña en texto plano
     * @return bool True si las credenciales son válidas
     */
    public function execute(string $email, string $password): bool
    {
        try {
            // 1. Crear Email VO desde string (valida formato)
            $emailVo = Email::fromString($email);
        } catch (\InvalidArgumentException $e) {
            // Email inválido, retornar false
            return false;
        }
        
        // 2. Buscar usuario por email normalizado (mediante repositorio)
        $merchantData = $this->merchantRepository->findByEmail($emailVo);
        
        // 3. Si no existe, retornar false
        if ($merchantData === null) {
            return false;
        }
        
        // 4. Verificar contraseña usando HashedPassword VO
        return $merchantData->password->verify($password);
    }
}
```

**Reglas de Negocio:**
- No crea sesión (solo validación)
- No modifica estado del sistema
- Retorna bool simple (true/false)
- Útil para verificaciones previas (ej: antes de operaciones sensibles)
- No lanza excepciones (retorna false si email inválido)
- No dispara eventos (validación silenciosa)
- **Verificación mediante HashedPassword VO (NO Hash::check() directo)**
- **Comunicación mediante MerchantRepositoryInterface**

---

### 5. Excepción de Dominio: InvalidCredentialsException

**Ubicación:** `Modules/Auth/App/Exceptions/InvalidCredentialsException.php`

**Especificaciones:**

```php
namespace Modules\Auth\App\Exceptions;

use Exception;

final class InvalidCredentialsException extends Exception
{
    public static function forUser(string $email): self
    {
        // Mensaje genérico para prevenir enumeración
        return new self('Las credenciales proporcionadas son inválidas.');
    }
    
    public static function default(): self
    {
        return new self('Las credenciales proporcionadas son inválidas.');
    }
}
```

**Reglas de Negocio:**
- Mensaje genérico para prevenir enumeración de usuarios
- No revela si el email existe o no
- No revela detalles específicos del error
- Factory methods para diferentes contextos

---

### 6. Excepción de Dominio: EmailNotVerifiedException (Opcional)

**Ubicación:** `Modules/Auth/App/Exceptions/EmailNotVerifiedException.php`

**Especificaciones:**

```php
namespace Modules\Auth\App\Exceptions;

use Exception;

final class EmailNotVerifiedException extends Exception
{
    public static function forEmail(string $email): self
    {
        return new self("El email {$email} no ha sido verificado.");
    }
}
```

**Nota:** Esta excepción está preparada para futuro, pero en MVP la verificación de email NO es obligatoria.

---

## 🧪 Tests Requeridos

### Test 1: AuthenticateMerchantAction

**Ubicación:** `tests/Feature/Auth/Actions/AuthenticateMerchantActionTest.php`

**Casos a cubrir:**

```php
use Modules\Auth\App\Actions\AuthenticateMerchantAction;
use Modules\Auth\Data\AuthenticateData;
use Modules\Auth\Events\UserLoginEvent;
use App\Models\User;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Hash;

describe('AuthenticateMerchantAction', function () {
    it('authenticates merchant with valid credentials', function () {
        // Arrange
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isSuccess())->toBeTrue()
            ->and($result->user->id)->toBe($user->id)
            ->and(auth()->check())->toBeTrue()
            ->and(auth()->user()->id)->toBe($user->id);
    });
    
    it('authenticates with remember token when remember is true', function () {
        // Arrange
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
            'remember_token' => null,
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'password123',
            remember: true
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isSuccess())->toBeTrue()
            ->and($user->fresh()->remember_token)->not->toBeNull();
    });
    
    it('does not generate remember token when remember is false', function () {
        // Arrange
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
            'remember_token' => null,
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isSuccess())->toBeTrue()
            ->and($user->fresh()->remember_token)->toBeNull();
    });
    
    it('fails authentication with invalid password', function () {
        // Arrange
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('correctpassword'),
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'wrongpassword',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isFailure())->toBeTrue()
            ->and($result->user)->toBeNull()
            ->and($result->message)->toBe('Credenciales inválidas')
            ->and(auth()->check())->toBeFalse();
    });
    
    it('fails authentication with non-existent email', function () {
        // Arrange
        $data = new AuthenticateData(
            email: 'nonexistent@example.com',
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isFailure())->toBeTrue()
            ->and($result->user)->toBeNull()
            ->and($result->message)->toBe('Credenciales inválidas')
            ->and(auth()->check())->toBeFalse();
    });
    
    it('normalizes email before authentication', function () {
        // Arrange
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $data = new AuthenticateData(
            email: 'MERCHANT@EXAMPLE.COM', // Uppercase
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert
        expect($result->isSuccess())->toBeTrue()
            ->and($result->user->id)->toBe($user->id);
    });
    
    it('dispatches UserLoginEvent on successful authentication', function () {
        // Arrange
        Event::fake();
        
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $action->execute($data);
        
        // Assert
        Event::assertDispatched(UserLoginEvent::class, function ($event) use ($user) {
            return $event->user_id === $user->id
                && $event->email->normalized === 'merchant@example.com'
                && $event->ip_address !== null
                && $event->logged_in_at !== null;
        });
    });
    
    it('does not dispatch event on failed authentication', function () {
        // Arrange
        Event::fake();
        
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('correctpassword'),
        ]);
        
        $data = new AuthenticateData(
            email: 'merchant@example.com',
            password: 'wrongpassword',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $action->execute($data);
        
        // Assert
        Event::assertNotDispatched(UserLoginEvent::class);
    });
    
    it('uses generic error message to prevent user enumeration', function () {
        // Arrange
        $data = new AuthenticateData(
            email: 'nonexistent@example.com',
            password: 'password123',
            remember: false
        );
        
        $action = new AuthenticateMerchantAction();
        
        // Act
        $result = $action->execute($data);
        
        // Assert - mismo mensaje que con contraseña incorrecta
        expect($result->message)->toBe('Credenciales inválidas')
            ->and($result->message)->not->toContain('email')
            ->and($result->message)->not->toContain('usuario')
            ->and($result->message)->not->toContain('existe');
    });
});
```

---

### Test 2: LogoutMerchantAction

**Ubicación:** `tests/Feature/Auth/Actions/LogoutMerchantActionTest.php`

**Casos a cubrir:**

```php
use Modules\Auth\App\Actions\LogoutMerchantAction;
use Modules\Auth\Events\UserLogoutEvent;
use App\Models\User;
use Illuminate\Support\Facades\Event;

describe('LogoutMerchantAction', function () {
    it('logs out authenticated merchant', function () {
        // Arrange
        $user = User::factory()->create();
        auth()->login($user);
        
        expect(auth()->check())->toBeTrue();
        
        $action = new LogoutMerchantAction();
        
        // Act
        $action->execute($user);
        
        // Assert
        expect(auth()->check())->toBeFalse();
    });
    
    it('revokes remember token on logout', function () {
        // Arrange
        $user = User::factory()->create([
            'remember_token' => 'some-remember-token',
        ]);
        auth()->login($user);
        
        $action = new LogoutMerchantAction();
        
        // Act
        $action->execute($user);
        
        // Assert
        expect($user->fresh()->remember_token)->toBeNull();
    });
    
    it('dispatches UserLogoutEvent', function () {
        // Arrange
        Event::fake();
        
        $user = User::factory()->create([
            'email' => 'merchant@example.com',
        ]);
        auth()->login($user);
        
        $action = new LogoutMerchantAction();
        
        // Act
        $action->execute($user);
        
        // Assert
        Event::assertDispatched(UserLogoutEvent::class, function ($event) use ($user) {
            return $event->user_id === $user->id
                && $event->email->normalized === 'merchant@example.com'
                && $event->logged_out_at !== null;
        });
    });
    
    it('invalidates session on logout', function () {
        // Arrange
        $user = User::factory()->create();
        auth()->login($user);
        
        $sessionId = session()->getId();
        
        $action = new LogoutMerchantAction();
        
        // Act
        $action->execute($user);
        
        // Assert
        expect(session()->getId())->not->toBe($sessionId);
    });
    
    it('regenerates CSRF token on logout', function () {
        // Arrange
        $user = User::factory()->create();
        auth()->login($user);
        
        $csrfToken = csrf_token();
        
        $action = new LogoutMerchantAction();
        
        // Act
        $action->execute($user);
        
        // Assert
        expect(csrf_token())->not->toBe($csrfToken);
    });
});
```

---

### Test 3: ValidateCredentialsAction

**Ubicación:** `tests/Feature/Auth/Actions/ValidateCredentialsActionTest.php`

**Casos a cubrir:**

```php
use Modules\Auth\App\Actions\ValidateCredentialsAction;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

describe('ValidateCredentialsAction', function () {
    it('returns true for valid credentials', function () {
        // Arrange
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $action = new ValidateCredentialsAction();
        
        // Act
        $result = $action->execute('merchant@example.com', 'password123');
        
        // Assert
        expect($result)->toBeTrue();
    });
    
    it('returns false for invalid password', function () {
        // Arrange
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('correctpassword'),
        ]);
        
        $action = new ValidateCredentialsAction();
        
        // Act
        $result = $action->execute('merchant@example.com', 'wrongpassword');
        
        // Assert
        expect($result)->toBeFalse();
    });
    
    it('returns false for non-existent email', function () {
        // Arrange
        $action = new ValidateCredentialsAction();
        
        // Act
        $result = $action->execute('nonexistent@example.com', 'password123');
        
        // Assert
        expect($result)->toBeFalse();
    });
    
    it('returns false for invalid email format', function () {
        // Arrange
        $action = new ValidateCredentialsAction();
        
        // Act
        $result = $action->execute('invalid-email', 'password123');
        
        // Assert
        expect($result)->toBeFalse();
    });
    
    it('normalizes email before validation', function () {
        // Arrange
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $action = new ValidateCredentialsAction();
        
        // Act
        $result = $action->execute('MERCHANT@EXAMPLE.COM', 'password123');
        
        // Assert
        expect($result)->toBeTrue();
    });
    
    it('does not create session when validating', function () {
        // Arrange
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        expect(auth()->check())->toBeFalse();
        
        $action = new ValidateCredentialsAction();
        
        // Act
        $action->execute('merchant@example.com', 'password123');
        
        // Assert
        expect(auth()->check())->toBeFalse();
    });
    
    it('does not dispatch events when validating', function () {
        // Arrange
        Event::fake();
        
        User::factory()->create([
            'email' => 'merchant@example.com',
            'password' => Hash::make('password123'),
        ]);
        
        $action = new ValidateCredentialsAction();
        
        // Act
        $action->execute('merchant@example.com', 'password123');
        
        // Assert
        Event::assertNothingDispatched();
    });
});
```

---

## ✅ Criterios de Aceptación

### Funcionales
- [ ] AuthenticateMerchantAction autentica con credenciales válidas
- [ ] AuthenticateMerchantAction falla con credenciales inválidas
- [ ] AuthenticateMerchantAction genera remember token si remember=true
- [ ] AuthenticateMerchantAction normaliza email antes de buscar usuario
- [ ] AuthenticateMerchantAction dispara UserLoginEvent en éxito
- [ ] AuthenticateMerchantAction usa mensajes genéricos (prevenir enumeración)
- [ ] LogoutMerchantAction invalida sesión correctamente
- [ ] LogoutMerchantAction revoca remember token
- [ ] LogoutMerchantAction regenera CSRF token
- [ ] LogoutMerchantAction dispara UserLogoutEvent
- [ ] ValidateCredentialsAction retorna true/false correctamente
- [ ] ValidateCredentialsAction no crea sesión
- [ ] ValidateCredentialsAction no dispara eventos

### Técnicos
- [ ] Todas las Actions son `final readonly class`
- [ ] Cada Action tiene un solo método público `execute()`
- [ ] **Todas las Actions implementan su interfaz correspondiente**
- [ ] **Interfaces creadas en `Contracts/Commands/` y `Contracts/Queries/`**
- [ ] Tipado fuerte completo (sin mixed, sin any)
- [ ] **Inyección de dependencias mediante constructor (repositorios como interfaces)**
- [ ] **NO acceso directo a Eloquent - solo mediante repositorios**
- [ ] Exceptions de dominio con factory methods
- [ ] Tests con Pest 4 (describe/it syntax)
- [ ] Cobertura de tests: 100% de las Actions
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado sin advertencias
- [ ] **Tests con mocks de repositorios (NO factories de Eloquent en tests unitarios)**

### Seguridad
- [ ] Mensajes de error genéricos (no revelan existencia de usuario)
- [ ] **Passwords verificados con `HashedPassword->verify()` (NO `Hash::check()` directo)**
- [ ] Remember tokens generados con Str::random(60)
- [ ] Session regeneration después de login
- [ ] CSRF token regenerado después de logout
- [ ] Eventos incluyen IP address para auditoría
- [ ] **Encapsulación de lógica de seguridad en Value Objects**

### Documentación
- [ ] Docblocks en clases y métodos públicos
- [ ] `@param` y `@return` documentados
- [ ] `@throws` documentado cuando aplique
- [ ] Pasos del algoritmo comentados en código

## 🔧 Comandos de Validación

```bash
# Ejecutar tests de Actions
./vendor/bin/sail test tests/Feature/Auth/Actions

# Análisis estático
./vendor/bin/sail composer run phpstan -- --paths=Modules/Auth/App/Actions

# Formateo de código
./vendor/bin/sail bin pint Modules/Auth/App/Actions

# Cobertura de tests
./vendor/bin/sail test --coverage --min=100 tests/Feature/Auth/Actions
```

## 📚 Referencias

- **Domain Model:** `@e-commerce-wa-ml/auth/domain_model.md` (líneas 251-343)
- **Metodología:** `@laravel/agents/agent-b-actions.md`
- **Business Rules:** Auth module (líneas 410-434)
- **Security:** Password hashing, session management (líneas 520-590)

## 📝 Notas de Implementación

### AuthenticateMerchantAction
- **CRÍTICO:** Usar `HashedPassword->verify()` en lugar de `Hash::check()` directo
- Comunicar con repositorio mediante `MerchantRepositoryInterface`
- Usar `Auth::loginUsingId()` en lugar de `Auth::login()` para trabajar con VOs
- Remember token: `Str::random(60)` es el estándar de Laravel
- Mensaje genérico idéntico para "usuario no existe" y "password incorrecta"
- Event debe dispararse DESPUÉS de crear la sesión
- Capturar IP con `request()->ip()` para auditoría
- **Implementar interfaz `AuthenticateMerchantInterface`**

### LogoutMerchantAction
- Orden importante: revocar token → disparar evento → logout
- `session()->invalidate()` limpia todos los datos de sesión
- `session()->regenerateToken()` previene CSRF attacks
- Laravel maneja cookies automáticamente con `Auth::logout()`
- **Usar repositorio para actualizar remember_token (NO acceso directo a Eloquent)**
- **Implementar interfaz `LogoutMerchantInterface`**

### ValidateCredentialsAction
- **CRÍTICO:** Usar `HashedPassword->verify()` en lugar de `Hash::check()` directo
- NO usar `Auth::attempt()` porque crearía sesión
- Catch exceptions de Email VO y retornar false (no propagarlas)
- No logear nada (validación silenciosa)
- Útil para "confirm password" antes de acciones sensibles
- **Comunicar con repositorio mediante `MerchantRepositoryInterface`**
- **Implementar interfaz `ValidateCredentialsInterface`**

### Testing Strategy
- Usar `User::factory()` en todos los tests
- `Event::fake()` para verificar dispatch de eventos
- `Hash::make()` para crear passwords de prueba
- Tests de "no crea sesión" importantes para ValidateCredentialsAction
- Tests de normalización de email importantes (case-insensitive)

### Edge Cases a Testear
- Email con espacios y uppercase
- Password vacío o muy corto (validado en DTO)
- Usuario sin remember token previo
- Multiple logouts consecutivos
- Validación sin usuario autenticado

---

**Status:** ✅ Ready to Implement  
**Fase:** 2 - Lógica de Negocio  
**Bloqueante para:** Task 004 (HTTP/Filament)  
**Depende de:** Task 001 (Contracts), Task 003 (Persistencia)
