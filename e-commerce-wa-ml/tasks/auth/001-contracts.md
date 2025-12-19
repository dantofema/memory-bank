---
task_id: "auth-001-contracts"
module: "Auth"
agent: "Agente A - Contratos, Data, VOs y Enums"
title: "Auth - Contratos, Data Transfer Objects y Value Objects"
priority: "HIGH"
estimated_time: "4 hours"
dependencies: []
status: "pending"
references:
  - "@laravel/agents/agent-a-contracts.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
  - "@laravel/conventions/value-objects.md"
phase: "Fase 1 - Fundamentos"
---

# Task 001: Auth - Contratos, Data, VOs y Enums

## 🎯 Objetivo

Implementar los contratos públicos del módulo Auth: Value Objects, Data Transfer Objects y estructuras de datos que definen la frontera pública del módulo.

## 📋 Contexto

El módulo Auth es TRANSVERSAL y proporciona autenticación exclusivamente para el backoffice de Filament. Esta tarea establece los fundamentos del módulo creando los tipos de datos inmutables que garantizan la validez de la información desde el punto de entrada.

### Referencias del Domain Model
- **Value Objects:** Email (líneas 189-226)
- **DTOs:** AuthenticateData (líneas 346-361), AuthResult (líneas 363-372)
- **Business Rules:** Authentication (líneas 410-419)

## 📦 Artefactos a Crear

### 1. Value Object: Email

**Ubicación:** `Modules/Auth/ValueObjects/Email.php`

**Especificaciones:**

```php
final readonly class Email implements Wireable
{
    public string $value;
    public string $normalized;
    public string $domain;
    
    public function __construct(string $value)
    {
        // Validar formato RFC 5322
        // Lanzar InvalidArgumentException si inválido
        // Normalizar (lowercase, trim)
        // Extraer dominio
    }
    
    public static function fromString(string $value): self;
    public function normalize(): string;
    public function getDomain(): string;
    public function matches(Email $other): bool;
    public function isValid(): bool;
    
    // Wireable interface
    public function toLivewire(): string;
    public static function fromLivewire($value): self;
}
```

**Reglas de Negocio:**
- Debe cumplir formato RFC 5322
- Longitud máxima de 255 caracteres
- Se normaliza a lowercase y trim
- El dominio debe existir (validación opcional)
- Nunca debe existir un Email inválido (validación en constructor)

**Justificación del VO:**
- ✅ No debe existir inválido (criterio 1)
- ✅ Se reutiliza en múltiples contextos (criterio 2)
- ✅ Tiene reglas de negocio propias (criterio 3)

---

### 2. Data Object: AuthenticateData

**Ubicación:** `Modules/Auth/Data/AuthenticateData.php`

**Especificaciones:**

```php
use Spatie\LaravelData\Data;

final class AuthenticateData extends Data
{
    public function __construct(
        public string $email,
        public string $password,
        public bool $remember = false,
    ) {}
    
    public static function rules(): array
    {
        return [
            'email' => ['required', 'email', 'max:255'],
            'password' => ['required', 'string', 'min:8'],
            'remember' => ['boolean'],
        ];
    }
}
```

**Reglas de Negocio:**
- Email formato válido, máximo 255 caracteres
- Password requerido, mínimo 8 caracteres
- Remember opcional, default false

---

### 3. Data Object: AuthResult

**Ubicación:** `Modules/Auth/Data/AuthResult.php`

**Especificaciones:**

```php
use Spatie\LaravelData\Data;
use App\Models\User;

final class AuthResult extends Data
{
    public function __construct(
        public bool $success,
        public ?User $user,
        public string $message,
    ) {}
    
    public static function success(User $user, string $message = 'Login exitoso'): self;
    public static function failure(string $message = 'Credenciales inválidas'): self;
    public function isSuccess(): bool;
    public function isFailure(): bool;
}
```

**Reglas de Negocio:**
- Success true solo si user != null
- Mensaje descriptivo pero genérico (prevenir enumeración de usuarios)
- User nullable (null si falla autenticación)

---

## 🧪 Tests Requeridos

### Test 1: Email Value Object

**Ubicación:** `tests/Unit/Auth/ValueObjects/EmailTest.php`

**Casos a cubrir:**

```php
describe('Email Value Object', function () {
    it('creates email from valid string', function () {
        $email = Email::fromString('user@example.com');
        expect($email->value)->toBe('user@example.com')
            ->and($email->normalized)->toBe('user@example.com')
            ->and($email->domain)->toBe('example.com');
    });
    
    it('normalizes email to lowercase', function () {
        $email = Email::fromString('USER@EXAMPLE.COM');
        expect($email->normalized)->toBe('user@example.com');
    });
    
    it('trims whitespace from email', function () {
        $email = Email::fromString('  user@example.com  ');
        expect($email->normalized)->toBe('user@example.com');
    });
    
    it('extracts domain correctly', function () {
        $email = Email::fromString('user@subdomain.example.com');
        expect($email->getDomain())->toBe('subdomain.example.com');
    });
    
    it('throws exception for invalid email format', function () {
        expect(fn() => Email::fromString('invalid-email'))
            ->toThrow(InvalidArgumentException::class);
    });
    
    it('throws exception for empty email', function () {
        expect(fn() => Email::fromString(''))
            ->toThrow(InvalidArgumentException::class);
    });
    
    it('throws exception for email exceeding max length', function () {
        $longEmail = str_repeat('a', 246) . '@test.com'; // > 255 chars
        expect(fn() => Email::fromString($longEmail))
            ->toThrow(InvalidArgumentException::class);
    });
    
    it('matches same email regardless of case', function () {
        $email1 = Email::fromString('user@example.com');
        $email2 = Email::fromString('USER@EXAMPLE.COM');
        expect($email1->matches($email2))->toBeTrue();
    });
    
    it('implements Wireable for Livewire', function () {
        $email = Email::fromString('user@example.com');
        $livewireValue = $email->toLivewire();
        $reconstructed = Email::fromLivewire($livewireValue);
        
        expect($livewireValue)->toBe('user@example.com')
            ->and($reconstructed->value)->toBe($email->value);
    });
});
```

---

### Test 2: AuthenticateData DTO

**Ubicación:** `tests/Unit/Auth/Data/AuthenticateDataTest.php`

**Casos a cubrir:**

```php
describe('AuthenticateData DTO', function () {
    it('creates from valid data', function () {
        $data = AuthenticateData::from([
            'email' => 'user@example.com',
            'password' => 'password123',
            'remember' => true,
        ]);
        
        expect($data->email)->toBe('user@example.com')
            ->and($data->password)->toBe('password123')
            ->and($data->remember)->toBeTrue();
    });
    
    it('defaults remember to false', function () {
        $data = new AuthenticateData(
            email: 'user@example.com',
            password: 'password123',
        );
        
        expect($data->remember)->toBeFalse();
    });
    
    it('validates email format', function () {
        expect(fn() => AuthenticateData::from([
            'email' => 'invalid-email',
            'password' => 'password123',
        ]))->toThrow(\Spatie\LaravelData\Exceptions\ValidationException::class);
    });
    
    it('validates password minimum length', function () {
        expect(fn() => AuthenticateData::from([
            'email' => 'user@example.com',
            'password' => 'short',
        ]))->toThrow(\Spatie\LaravelData\Exceptions\ValidationException::class);
    });
    
    it('requires email field', function () {
        expect(fn() => AuthenticateData::from([
            'password' => 'password123',
        ]))->toThrow(\Spatie\LaravelData\Exceptions\ValidationException::class);
    });
    
    it('requires password field', function () {
        expect(fn() => AuthenticateData::from([
            'email' => 'user@example.com',
        ]))->toThrow(\Spatie\LaravelData\Exceptions\ValidationException::class);
    });
});
```

---

### Test 3: AuthResult DTO

**Ubicación:** `tests/Unit/Auth/Data/AuthResultTest.php`

**Casos a cubrir:**

```php
describe('AuthResult DTO', function () {
    it('creates success result with user', function () {
        $user = User::factory()->make();
        $result = AuthResult::success($user, 'Welcome back!');
        
        expect($result->success)->toBeTrue()
            ->and($result->user)->toBe($user)
            ->and($result->message)->toBe('Welcome back!')
            ->and($result->isSuccess())->toBeTrue()
            ->and($result->isFailure())->toBeFalse();
    });
    
    it('creates success result with default message', function () {
        $user = User::factory()->make();
        $result = AuthResult::success($user);
        
        expect($result->message)->toBe('Login exitoso');
    });
    
    it('creates failure result without user', function () {
        $result = AuthResult::failure('Invalid credentials');
        
        expect($result->success)->toBeFalse()
            ->and($result->user)->toBeNull()
            ->and($result->message)->toBe('Invalid credentials')
            ->and($result->isSuccess())->toBeFalse()
            ->and($result->isFailure())->toBeTrue();
    });
    
    it('creates failure result with default message', function () {
        $result = AuthResult::failure();
        
        expect($result->message)->toBe('Credenciales inválidas');
    });
});
```

---

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Email VO valida formato RFC 5322 correctamente
- [ ] Email VO normaliza a lowercase y trim
- [ ] Email VO extrae dominio correctamente
- [ ] Email VO implementa Wireable para Livewire
- [ ] Email VO lanza excepciones con datos inválidos
- [ ] AuthenticateData valida email, password y remember
- [ ] AuthenticateData tiene defaults apropiados
- [ ] AuthResult diferencia entre éxito y fallo
- [ ] AuthResult tiene factory methods para success/failure

### Técnicos
- [ ] Todas las clases son `final`
- [ ] Email VO es `readonly`
- [ ] Tipado fuerte completo (sin mixed, sin any)
- [ ] Sin dependencias externas en Value Objects
- [ ] DTOs usan Spatie Laravel Data correctamente
- [ ] Tests con Pest 4 (describe/it syntax)
- [ ] Cobertura de tests: 100%
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado sin advertencias

### Documentación
- [ ] Docblocks en clases públicas
- [ ] `@param` y `@return` en métodos públicos
- [ ] Justificación de Value Objects documentada
- [ ] Referencias al domain model incluidas

## 🔧 Comandos de Validación

```bash
# Ejecutar tests unitarios de esta task
./vendor/bin/sail test tests/Unit/Auth/ValueObjects
./vendor/bin/sail test tests/Unit/Auth/Data

# Análisis estático
./vendor/bin/sail composer run phpstan -- --paths=Modules/Auth/ValueObjects,Modules/Auth/Data

# Formateo de código
./vendor/bin/sail bin pint Modules/Auth/ValueObjects
./vendor/bin/sail bin pint Modules/Auth/Data
```

## 📚 Referencias

- **Domain Model:** `@e-commerce-wa-ml/auth/domain_model.md` (líneas 189-372)
- **Metodología:** `@laravel/agents/agent-a-contracts.md`
- **Value Objects Guide:** `@laravel/conventions/value-objects.md`
- **Project Definition:** `@e-commerce-wa-ml/project_definition.md`

## 📝 Notas de Implementación

### Email Value Object
- Usar `filter_var($email, FILTER_VALIDATE_EMAIL)` para validación básica
- Normalización: `strtolower(trim($value))`
- Dominio: `explode('@', $normalized)[1]`
- Wireable: implementar `toLivewire()` → string, `fromLivewire($value)` → self

### AuthenticateData
- Usar atributos de validación de Spatie Data
- Constructor con defaults claros
- No hashear password aquí (eso va en Action)

### AuthResult
- Factory methods preferidos sobre constructor directo
- Mensajes genéricos para prevenir enumeración de usuarios
- User nullable y manejado correctamente

### Seguridad
- Email: prevenir SQL injection (no aplica aquí, se hace en persistencia)
- Passwords: nunca logear, nunca exponer en respuestas
- Mensajes de error genéricos para login (no revelar si email existe)

---

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Bloqueante para:** Task 002 (Actions), Task 003 (Persistencia)
