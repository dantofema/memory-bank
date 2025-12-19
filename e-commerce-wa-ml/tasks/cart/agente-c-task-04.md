---
task_id: "agente-c-task-04"
module: "Auth"
agent: "Agente C"
title: "Tests de Integración de Persistencia"
priority: "medium"
estimated_time: "1.5h"
dependencies:
  - "agente-c-task-01: Modelo User"
  - "agente-c-task-02: Migración users"
  - "agente-c-task-03: Modelo PasswordResetToken y Repositorio"
  - "Agente B: Actions implementadas"
status: "pending"
---

# Task: Tests de Integración de Persistencia

## Contexto

Crear tests de integración que validen el flujo completo de persistencia entre Modelos, Repositorios, Casts y Base de Datos. Estos tests aseguran que toda la capa de persistencia funciona correctamente en conjunto.

---

## Objetivos

1. ✅ Validar integración completa User + Casts + DB
2. ✅ Validar integración completa PasswordResetToken + Repository + DB
3. ✅ Validar flujo de password reset (integración repository)
4. ✅ Validar transacciones y rollbacks
5. ✅ Validar índices y performance

---

## Tests a Crear

### 1. Integración User + Casts + DB

**Ubicación**: `tests/Feature/Database/UserPersistenceTest.php`

**Tests obligatorios**:

```php
use App\Models\User;
use App\ValueObjects\EmailValueObject;
use App\ValueObjects\NameValueObject;
use App\ValueObjects\PasswordValueObject;
use App\Enums\MerchantRoleEnum;

it('persiste y recupera correctamente un usuario con Value Objects', function () {
    $user = User::factory()->create([
        'name' => new NameValueObject('John Doe'),
        'email' => new EmailValueObject('john@example.com'),
        'password' => PasswordValueObject::fromPlain('SecurePass123'),
        'role' => MerchantRoleEnum::OWNER,
    ]);

    $retrieved = User::find($user->id);

    expect($retrieved->name)->toBeInstanceOf(NameValueObject::class)
        ->and($retrieved->name->toString())->toBe('John Doe')
        ->and($retrieved->email)->toBeInstanceOf(EmailValueObject::class)
        ->and($retrieved->email->toString())->toBe('john@example.com')
        ->and($retrieved->password)->toBeInstanceOf(PasswordValueObject::class)
        ->and($retrieved->password->verify('SecurePass123'))->toBeTrue()
        ->and($retrieved->role)->toBe(MerchantRoleEnum::OWNER);
});

it('actualiza correctamente los Value Objects', function () {
    $user = User::factory()->create();

    $user->name = new NameValueObject('Updated Name');
    $user->email = new EmailValueObject('updated@example.com');
    $user->password = PasswordValueObject::fromPlain('NewPassword123');
    $user->save();

    $user->refresh();

    expect($user->name->toString())->toBe('Updated Name')
        ->and($user->email->toString())->toBe('updated@example.com')
        ->and($user->password->verify('NewPassword123'))->toBeTrue();
});

it('mantiene la integridad de email único', function () {
    $email = new EmailValueObject('duplicate@example.com');
    
    User::factory()->create(['email' => $email]);

    expect(fn() => User::factory()->create(['email' => $email]))
        ->toThrow(\Illuminate\Database\QueryException::class);
});

it('el índice en email mejora la búsqueda', function () {
    User::factory()->count(1000)->create();

    $email = new EmailValueObject('test@example.com');
    User::factory()->create(['email' => $email]);

    // Medir tiempo de búsqueda con índice
    $start = microtime(true);
    $user = User::where('email', $email->toString())->first();
    $time = microtime(true) - $start;

    expect($user)->not->toBeNull()
        ->and($time)->toBeLessThan(0.2); // Menos de 200ms con índice (ajustable según hardware)
});

it('filtra correctamente por rol con índice', function () {
    User::factory()->owner()->count(5)->create();
    User::factory()->admin()->count(10)->create();

    $owners = User::where('role', MerchantRoleEnum::OWNER->value)->get();
    $admins = User::where('role', MerchantRoleEnum::ADMIN->value)->get();

    expect($owners)->toHaveCount(5)
        ->and($admins)->toHaveCount(10);
});
```

---

### 2. Integración PasswordResetToken + Repository + DB

**Ubicación**: `Modules/Auth/tests/Feature/Database/PasswordResetTokenPersistenceTest.php`

**Tests obligatorios**:

```php
use Modules\Auth\Models\PasswordResetToken;
use Modules\Auth\Repositories\PasswordResetTokenRepository;
use App\Models\User;
use App\ValueObjects\EmailValueObject;

beforeEach(function () {
    $this->repository = new PasswordResetTokenRepository();
});

it('persiste y recupera correctamente un token con Cast', function () {
    $email = new EmailValueObject('persist@example.com');

    $token = $this->repository->create($email);

    $retrieved = PasswordResetToken::find($email->toString());

    expect($retrieved)->not->toBeNull()
        ->and($retrieved->email)->toBeInstanceOf(EmailValueObject::class)
        ->and($retrieved->email->toString())->toBe('persist@example.com')
        ->and($retrieved->token)->toBeString()
        ->and($retrieved->created_at)->toBeInstanceOf(\Carbon\Carbon::class);
});

it('mantiene integridad de PK única (email)', function () {
    $email = new EmailValueObject('unique@example.com');

    $token1 = $this->repository->create($email);
    $token2 = $this->repository->create($email);

    expect(PasswordResetToken::count())->toBe(1)
        ->and($token2->token)->not->toBe($token1->token);
});

it('el índice en created_at permite búsqueda eficiente de expirados', function () {
    // Crear 1000 tokens válidos
    PasswordResetToken::factory()->count(1000)->create();

    // Crear 100 tokens expirados
    PasswordResetToken::factory()->expired()->count(100)->create();

    // Medir tiempo de búsqueda de expirados con índice
    $start = microtime(true);
    $deleted = $this->repository->deleteExpired();
    $time = microtime(true) - $start;

    expect($deleted)->toBe(100)
        ->and($time)->toBeLessThan(1.0) // Menos de 1s con índice (ajustable según hardware)
        ->and(PasswordResetToken::count())->toBe(1000);
});
```

---

### 3. Flujo Completo de Password Reset (Integración)

**Ubicación**: `Modules/Auth/tests/Feature/Database/PasswordResetFlowIntegrationTest.php`

**Tests obligatorios**:

```php
use Modules\Auth\Repositories\PasswordResetTokenRepository;
use App\Models\User;
use App\ValueObjects\EmailValueObject;
use App\ValueObjects\PasswordValueObject;

beforeEach(function () {
    $this->repository = new PasswordResetTokenRepository();
});

it('flujo completo: crear token -> validar -> actualizar password -> eliminar token', function () {
    // 1. Crear usuario
    $user = User::factory()->create([
        'email' => new EmailValueObject('reset@example.com'),
        'password' => PasswordValueObject::fromPlain('OldPassword123'),
    ]);

    // 2. Crear token de reset
    $token = $this->repository->create($user->email);
    expect($token->plainToken)->toBeString();
    $plainToken = $token->plainToken;

    // 3. Validar token
    $retrieved = $this->repository->findByEmail($user->email);
    expect($retrieved)->not->toBeNull()
        ->and($retrieved->isExpired())->toBeFalse()
        ->and($retrieved->isValid($plainToken))->toBeTrue();

    // 4. Actualizar password
    $user->password = PasswordValueObject::fromPlain('NewPassword456');
    $user->save();

    // 5. Eliminar token usado
    $this->repository->delete($user->email);

    // 6. Validar estado final
    $user->refresh();
    expect($user->password->verify('NewPassword456'))->toBeTrue()
        ->and($user->password->verify('OldPassword123'))->toBeFalse()
        ->and($this->repository->findByEmail($user->email))->toBeNull();
});

it('flujo de expiración: token expirado no es válido', function () {
    $user = User::factory()->create();

    // Crear token expirado
    $token = PasswordResetToken::factory()->expired()->create([
        'email' => $user->email,
    ]);

    $retrieved = $this->repository->findByEmail($user->email);

    expect($retrieved)->not->toBeNull()
        ->and($retrieved->isExpired())->toBeTrue();
});

it('flujo de token inválido: hash no coincide', function () {
    $user = User::factory()->create();
    $token = $this->repository->create($user->email);

    $retrieved = $this->repository->findByEmail($user->email);

    expect($retrieved->isValid('wrong-token'))->toBeFalse()
        ->and($retrieved->isValid($token->plainToken))->toBeTrue();
});
```

---

### 4. Tests de Transacciones

**Ubicación**: `tests/Feature/Database/TransactionTest.php`

**Tests obligatorios**:

```php
use App\Models\User;
use App\ValueObjects\EmailValueObject;
use Illuminate\Support\Facades\DB;

it('rollback de transacción revierte la creación de usuario', function () {
    $initialCount = User::count();

    try {
        DB::transaction(function () {
            User::factory()->create([
                'email' => new EmailValueObject('transaction@example.com'),
            ]);

            throw new \Exception('Force rollback');
        });
    } catch (\Exception $e) {
        // Exception expected
    }

    expect(User::count())->toBe($initialCount)
        ->and(User::where('email', 'transaction@example.com')->exists())->toBeFalse();
});

it('commit de transacción persiste correctamente', function () {
    $initialCount = User::count();

    DB::transaction(function () {
        User::factory()->count(3)->create();
    });

    expect(User::count())->toBe($initialCount + 3);
});
```

---

## Validación de Calidad

Antes de considerar completa la tarea:

1. ✅ **Base de datos limpia**
   ```bash
   ./vendor/bin/sail artisan migrate:fresh
   ```

2. ✅ **Tests verdes** con cobertura del 100%
   ```bash
   ./vendor/bin/sail test tests/Feature/Database/
   ./vendor/bin/sail test Modules/Auth/tests/Feature/Database/
   ```

3. ✅ **Performance de índices validada**
   - Verificar que búsquedas con índices son rápidas (<100ms para 1000 registros)

4. ✅ **PHPStan level 6** sin errores
   ```bash
   ./vendor/bin/sail composer run phpstan
   ```

5. ✅ **Pint** ejecutado
   ```bash
   ./vendor/bin/sail bin pint --dirty
   ```

---

## Restricciones

❌ **NO crear**:
- Actions (ya creadas por Agente B)
- Lógica de negocio
- Controllers o Filament Pages

✅ **Solo crear**:
- Tests de integración de persistencia
- Tests de performance de índices
- Tests de transacciones

---

## Referencias

- **Diagrama de clases Auth**: `e-commerce-wa-ml/auth/auth-class-diagram.md`
- **Guía Agente C**: `laravel/agents/agente-c.md` líneas 680-782
- **Convenciones del proyecto**: `e-commerce-wa-ml/project_definition.md`
- **Laravel Testing**: https://laravel.com/docs/database-testing

---

## Notas

- Estos tests validan la integración completa de la capa de persistencia
- Los tests de performance validan que los índices funcionan correctamente
- Los tests de transacciones aseguran integridad de datos
- Usar `migrate:fresh` antes de ejecutar tests para asegurar estado limpio
