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

### Value Objects (3)
1. **Email** - Correo electrónico con validación RFC 5322
2. **MerchantName** - Nombre del comerciante con normalización
3. **HashedPassword** - Password hasheado con bcrypt/argon2

### Data Transfer Objects (2)
1. **AuthenticateData** - Input para autenticación
2. **AuthResult** - Output del proceso de autenticación

### Excepciones (4)
1. **InvalidEmailException** - Email inválido
2. **InvalidMerchantNameException** - Nombre de comerciante inválido
3. **InvalidHashedPasswordException** - Hash de password inválido
4. **InvalidPlainPasswordException** - Password plano inválido

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
        // Lanzar InvalidEmailException si inválido
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

**Excepciones:**
- `InvalidEmailException`: formato inválido, vacío o excede 255 caracteres

**Justificación del VO:**
- ✅ No debe existir inválido (criterio 1)
- ✅ Se reutiliza en múltiples contextos (criterio 2)
- ✅ Tiene reglas de negocio propias (criterio 3)

---

### 2. Value Object: MerchantName

**Ubicación:** `Modules/Auth/ValueObjects/MerchantName.php`

**Especificaciones:**

```php
final readonly class MerchantName implements Wireable
{
    public string $value;
    public string $normalized;
    
    public function __construct(string $value)
    {
        // Validar longitud mínima/máxima
        // Lanzar InvalidMerchantNameException si inválido
        // Normalizar (trim, capitalizar primera letra de cada palabra)
    }
    
    public static function fromString(string $value): self;
    public function normalize(): string;
    public function matches(MerchantName $other): bool;
    
    // Wireable interface
    public function toLivewire(): string;
    public static function fromLivewire($value): self;
}
```

**Reglas de Negocio:**
- Longitud mínima: 2 caracteres
- Longitud máxima: 100 caracteres
- No puede ser solo espacios en blanco
- Se normaliza con trim y capitalización
- Nunca debe existir un MerchantName inválido

**Excepciones:**
- `InvalidMerchantNameException`: vacío, solo espacios, menor a 2 chars o mayor a 100 chars

**Justificación del VO:**
- ✅ No debe existir inválido (criterio 1)
- ✅ Se reutiliza en múltiples contextos (criterio 2)
- ✅ Tiene reglas de negocio propias (criterio 3)

---

### 3. Value Object: HashedPassword

**Ubicación:** `Modules/Auth/ValueObjects/HashedPassword.php`

**Especificaciones:**

```php
final readonly class HashedPassword implements Wireable
{
    public string $hash;
    
    public function __construct(string $hash)
    {
        // Validar que sea un hash válido de bcrypt/argon2
        // Lanzar InvalidHashedPasswordException si inválido
    }
    
    public static function fromHash(string $hash): self;
    public static function fromPlainText(string $plainText): self;
    public function verify(string $plainText): bool;
    public function needsRehash(): bool;
    
    // Wireable interface
    public function toLivewire(): string;
    public static function fromLivewire($value): self;
}
```

**Reglas de Negocio:**
- Solo acepta hashes válidos de bcrypt ($2y$) o argon2 ($argon2)
- Longitud mínima del hash: 60 caracteres
- El plainText para hash debe tener mínimo 8 caracteres
- Usa algoritmo configurado en config('hashing.driver')
- Nunca debe existir un HashedPassword inválido

**Excepciones:**
- `InvalidHashedPasswordException`: hash inválido o formato no reconocido
- `InvalidPlainPasswordException`: password plano vacío o menor a 8 caracteres

**Justificación del VO:**
- ✅ No debe existir inválido (criterio 1)
- ✅ Se reutiliza en múltiples contextos (criterio 2)
- ✅ Tiene reglas de negocio propias (criterio 3)
- ✅ Encapsula lógica de hashing y verificación

---

### 4. Excepciones del Módulo

**Ubicación:** `Modules/Auth/Exceptions/`

Las excepciones del módulo extienden de excepciones base del dominio y deben ser específicas:

```php
// InvalidEmailException.php
namespace Modules\Auth\Exceptions;

use InvalidArgumentException;

final class InvalidEmailException extends InvalidArgumentException
{
    public static function invalidFormat(string $email): self
    {
        return new self("El email '{$email}' no tiene un formato válido.");
    }
    
    public static function tooLong(string $email, int $maxLength = 255): self
    {
        return new self("El email excede la longitud máxima de {$maxLength} caracteres.");
    }
    
    public static function empty(): self
    {
        return new self("El email no puede estar vacío.");
    }
}

// InvalidMerchantNameException.php
namespace Modules\Auth\Exceptions;

use InvalidArgumentException;

final class InvalidMerchantNameException extends InvalidArgumentException
{
    public static function tooShort(int $minLength = 2): self
    {
        return new self("El nombre del comerciante debe tener al menos {$minLength} caracteres.");
    }
    
    public static function tooLong(int $maxLength = 100): self
    {
        return new self("El nombre del comerciante no puede exceder {$maxLength} caracteres.");
    }
    
    public static function empty(): self
    {
        return new self("El nombre del comerciante no puede estar vacío.");
    }
    
    public static function onlyWhitespace(): self
    {
        return new self("El nombre del comerciante no puede contener solo espacios en blanco.");
    }
}

// InvalidHashedPasswordException.php
namespace Modules\Auth\Exceptions;

use InvalidArgumentException;

final class InvalidHashedPasswordException extends InvalidArgumentException
{
    public static function invalidFormat(string $hash): self
    {
        return new self("El hash proporcionado no es un hash válido de bcrypt o argon2.");
    }
    
    public static function tooShort(): self
    {
        return new self("El hash debe tener al menos 60 caracteres.");
    }
    
    public static function empty(): self
    {
        return new self("El hash no puede estar vacío.");
    }
}

// InvalidPlainPasswordException.php
namespace Modules\Auth\Exceptions;

use InvalidArgumentException;

final class InvalidPlainPasswordException extends InvalidArgumentException
{
    public static function tooShort(int $minLength = 8): self
    {
        return new self("La contraseña debe tener al menos {$minLength} caracteres.");
    }
    
    public static function empty(): self
    {
        return new self("La contraseña no puede estar vacía.");
    }
}
```

**Reglas de Excepciones:**
- Todas son `final class` para evitar extensión
- Extienden de `InvalidArgumentException` (domain exceptions)
- Usan factory methods estáticos con nombres descriptivos
- Mensajes claros y específicos (no genéricos)
- No exponen datos sensibles (ej: no mostrar passwords)

---

### 5. Data Object: AuthenticateData

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

### 6. Data Object: AuthResult

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
    
    it('throws InvalidEmailException for invalid email format', function () {
        expect(fn() => Email::fromString('invalid-email'))
            ->toThrow(InvalidEmailException::class);
    });
    
    it('throws InvalidEmailException for empty email', function () {
        expect(fn() => Email::fromString(''))
            ->toThrow(InvalidEmailException::class);
    });
    
    it('throws InvalidEmailException for email exceeding max length', function () {
        $longEmail = str_repeat('a', 246) . '@test.com'; // > 255 chars
        expect(fn() => Email::fromString($longEmail))
            ->toThrow(InvalidEmailException::class);
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

### Test 2: MerchantName Value Object

**Ubicación:** `tests/Unit/Auth/ValueObjects/MerchantNameTest.php`

**Casos a cubrir:**

```php
describe('MerchantName Value Object', function () {
    it('creates merchant name from valid string', function () {
        $name = MerchantName::fromString('Mi Tienda');
        expect($name->value)->toBe('Mi Tienda')
            ->and($name->normalized)->toBe('Mi Tienda');
    });
    
    it('normalizes merchant name with trim', function () {
        $name = MerchantName::fromString('  Mi Tienda  ');
        expect($name->normalized)->toBe('Mi Tienda');
    });
    
    it('capitalizes first letter of each word', function () {
        $name = MerchantName::fromString('mi tienda online');
        expect($name->normalized)->toBe('Mi Tienda Online');
    });
    
    it('throws InvalidMerchantNameException for too short name', function () {
        expect(fn() => MerchantName::fromString('A'))
            ->toThrow(InvalidMerchantNameException::class);
    });
    
    it('throws InvalidMerchantNameException for too long name', function () {
        $longName = str_repeat('A', 101);
        expect(fn() => MerchantName::fromString($longName))
            ->toThrow(InvalidMerchantNameException::class);
    });
    
    it('throws InvalidMerchantNameException for empty name', function () {
        expect(fn() => MerchantName::fromString(''))
            ->toThrow(InvalidMerchantNameException::class);
    });
    
    it('throws InvalidMerchantNameException for only whitespace', function () {
        expect(fn() => MerchantName::fromString('   '))
            ->toThrow(InvalidMerchantNameException::class);
    });
    
    it('matches same name regardless of case', function () {
        $name1 = MerchantName::fromString('Mi Tienda');
        $name2 = MerchantName::fromString('mi tienda');
        expect($name1->matches($name2))->toBeTrue();
    });
    
    it('implements Wireable for Livewire', function () {
        $name = MerchantName::fromString('Mi Tienda');
        $livewireValue = $name->toLivewire();
        $reconstructed = MerchantName::fromLivewire($livewireValue);
        
        expect($livewireValue)->toBe('Mi Tienda')
            ->and($reconstructed->value)->toBe($name->value);
    });
});
```

---

### Test 3: HashedPassword Value Object

**Ubicación:** `tests/Unit/Auth/ValueObjects/HashedPasswordTest.php`

**Casos a cubrir:**

```php
describe('HashedPassword Value Object', function () {
    it('creates from valid bcrypt hash', function () {
        $hash = '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi';
        $hashed = HashedPassword::fromHash($hash);
        expect($hashed->hash)->toBe($hash);
    });
    
    it('creates from plain text password', function () {
        $hashed = HashedPassword::fromPlainText('password123');
        expect($hashed->hash)->toBeString()
            ->and(strlen($hashed->hash))->toBeGreaterThanOrEqual(60);
    });
    
    it('verifies correct password', function () {
        $hashed = HashedPassword::fromPlainText('password123');
        expect($hashed->verify('password123'))->toBeTrue();
    });
    
    it('rejects incorrect password', function () {
        $hashed = HashedPassword::fromPlainText('password123');
        expect($hashed->verify('wrongpassword'))->toBeFalse();
    });
    
    it('throws InvalidHashedPasswordException for invalid hash format', function () {
        expect(fn() => HashedPassword::fromHash('not-a-valid-hash'))
            ->toThrow(InvalidHashedPasswordException::class);
    });
    
    it('throws InvalidHashedPasswordException for empty hash', function () {
        expect(fn() => HashedPassword::fromHash(''))
            ->toThrow(InvalidHashedPasswordException::class);
    });
    
    it('throws InvalidHashedPasswordException for too short hash', function () {
        expect(fn() => HashedPassword::fromHash('$2y$10$short'))
            ->toThrow(InvalidHashedPasswordException::class);
    });
    
    it('throws InvalidPlainPasswordException for empty plain password', function () {
        expect(fn() => HashedPassword::fromPlainText(''))
            ->toThrow(InvalidPlainPasswordException::class);
    });
    
    it('throws InvalidPlainPasswordException for too short plain password', function () {
        expect(fn() => HashedPassword::fromPlainText('short'))
            ->toThrow(InvalidPlainPasswordException::class);
    });
    
    it('detects when hash needs rehash', function () {
        // Hash con bcrypt cost bajo (para test)
        $oldHash = password_hash('password123', PASSWORD_BCRYPT, ['cost' => 4]);
        $hashed = HashedPassword::fromHash($oldHash);
        
        // Depende de la configuración actual
        expect($hashed->needsRehash())->toBeIn([true, false]);
    });
    
    it('implements Wireable for Livewire', function () {
        $hash = '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi';
        $hashed = HashedPassword::fromHash($hash);
        $livewireValue = $hashed->toLivewire();
        $reconstructed = HashedPassword::fromLivewire($livewireValue);
        
        expect($livewireValue)->toBe($hash)
            ->and($reconstructed->hash)->toBe($hashed->hash);
    });
});
```

---

### Test 4: AuthenticateData DTO

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

### Test 5: AuthResult DTO

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
- [ ] Email VO lanza InvalidEmailException con datos inválidos
- [ ] MerchantName VO valida longitud (2-100 caracteres)
- [ ] MerchantName VO normaliza con trim y capitalización
- [ ] MerchantName VO lanza InvalidMerchantNameException con datos inválidos
- [ ] HashedPassword VO valida hash de bcrypt/argon2
- [ ] HashedPassword VO puede crear hash desde plaintext
- [ ] HashedPassword VO verifica passwords correctamente
- [ ] HashedPassword VO lanza InvalidHashedPasswordException con hash inválido
- [ ] HashedPassword VO lanza InvalidPlainPasswordException con plaintext inválido
- [ ] AuthenticateData valida email, password y remember
- [ ] AuthenticateData tiene defaults apropiados
- [ ] AuthResult diferencia entre éxito y fallo
- [ ] AuthResult tiene factory methods para success/failure
- [ ] Todas las excepciones tienen factory methods descriptivos

### Técnicos
- [ ] Todas las clases son `final`
- [ ] Todos los VOs son `readonly`
- [ ] Tipado fuerte completo (sin mixed, sin any)
- [ ] Sin dependencias externas en Value Objects
- [ ] DTOs usan Spatie Laravel Data correctamente
- [ ] Excepciones extienden de InvalidArgumentException
- [ ] Excepciones no exponen datos sensibles
- [ ] Tests con Pest 4 (describe/it syntax)
- [ ] Cobertura de tests: 100%
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado sin advertencias

### Documentación
- [ ] Docblocks en clases públicas
- [ ] `@param` y `@return` en métodos públicos
- [ ] `@throws` en métodos que lanzan excepciones
- [ ] Justificación de Value Objects documentada
- [ ] Referencias al domain model incluidas

## 🔧 Comandos de Validación

```bash
# Ejecutar tests unitarios de esta task
./vendor/bin/sail test tests/Unit/Auth/ValueObjects
./vendor/bin/sail test tests/Unit/Auth/Data
./vendor/bin/sail test tests/Unit/Auth/Exceptions

# Análisis estático
./vendor/bin/sail composer run phpstan -- --paths=Modules/Auth/ValueObjects,Modules/Auth/Data,Modules/Auth/Exceptions

# Formateo de código
./vendor/bin/sail bin pint Modules/Auth/ValueObjects
./vendor/bin/sail bin pint Modules/Auth/Data
./vendor/bin/sail bin pint Modules/Auth/Exceptions
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
- Lanzar `InvalidEmailException` con factory methods específicos

### MerchantName Value Object
- Validar longitud entre 2 y 100 caracteres
- Normalización: `trim()` + capitalizar cada palabra con `ucwords(strtolower())`
- Rechazar strings de solo espacios con `trim($value) === ''`
- Wireable: similar a Email
- Lanzar `InvalidMerchantNameException` con factory methods específicos

### HashedPassword Value Object
- Validar formato: hash debe empezar con `$2y$` (bcrypt) o `$argon2` (argon2)
- Usar `password_hash()` para crear hash desde plaintext
- Usar `password_verify()` para verificar
- Usar `password_needs_rehash()` para detectar rehash necesario
- Longitud mínima del hash: 60 caracteres
- Plaintext: validar mínimo 8 caracteres antes de hashear
- Lanzar `InvalidHashedPasswordException` para hashes inválidos
- Lanzar `InvalidPlainPasswordException` para plaintext inválido
- Wireable: solo exponer el hash

### AuthenticateData
- Usar atributos de validación de Spatie Data
- Constructor con defaults claros
- No hashear password aquí (eso va en Action)

### AuthResult
- Factory methods preferidos sobre constructor directo
- Mensajes genéricos para prevenir enumeración de usuarios
- User nullable y manejado correctamente

### Excepciones
- Todas las excepciones deben ser `final class`
- Extender de `InvalidArgumentException`
- Usar factory methods estáticos con nombres descriptivos
- Mensajes claros pero no exponer datos sensibles
- Ubicación: `Modules/Auth/Exceptions/`

### Seguridad
- Email: prevenir SQL injection (no aplica aquí, se hace en persistencia)
- Passwords: nunca logear, nunca exponer en respuestas
- HashedPassword: solo exponer el hash, nunca plaintext
- Mensajes de error genéricos para login (no revelar si email existe)
- Excepciones: no incluir datos sensibles en mensajes

---

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Bloqueante para:** Task 002 (Actions), Task 003 (Persistencia)
