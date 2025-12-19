---
task_id: "agente-c-task-03"
module: "Auth"
agent: "Agente C"
title: "Modelo PasswordResetToken y Repositorio"
priority: "high"
estimated_time: "2.5h"
dependencies:
  - "agente-c-task-01: Modelo User y Casts"
  - "Agente A: PasswordResetTokenRepository contract"
status: "pending"
---

# Task: Modelo PasswordResetToken y Repositorio

## Contexto

Implementar el modelo `PasswordResetToken` en el módulo Auth y su repositorio para gestionar tokens de recuperación de contraseña. Utiliza la tabla nativa de Laravel `password_reset_tokens`.

**Ubicación**: `Modules/Auth/app/Models/PasswordResetToken.php`

---

## Objetivos

1. ✅ Crear modelo `PasswordResetToken` como Eloquent
2. ✅ Implementar métodos de validación (`isExpired()`, `isValid()`)
3. ✅ Crear migración de tabla
4. ✅ Implementar `PasswordResetTokenRepository`
5. ✅ Tests unitarios y de integración

---

## Archivos a Crear

### 1. Modelo PasswordResetToken

**Ubicación**: `Modules/Auth/app/Models/PasswordResetToken.php`

**Especificaciones**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Models;

use Carbon\Carbon;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Hash;
use App\ValueObjects\EmailValueObject;
use App\Casts\EmailCast;

/**
 * @property EmailValueObject $email
 * @property string $token
 * @property Carbon $created_at
 */
final class PasswordResetToken extends Model
{
    protected $table = 'password_reset_tokens';

    protected $primaryKey = 'email';

    protected $keyType = 'string';

    public $incrementing = false;

    public $timestamps = false; // Solo usa created_at manual

    protected $fillable = [
        'email',
        'token',
        'created_at',
    ];

    /**
     * Token plano temporal (solo en memoria, no se persiste).
     * Se usa para enviarlo por email después de la creación.
     */
    public ?string $plainToken = null;

    public function casts(): array
    {
        return [
            'email' => EmailCast::class,
            'created_at' => 'datetime',
        ];
    }

    /**
     * Verifica si el token ha expirado.
     * Expiración: 60 minutos desde created_at.
     */
    public function isExpired(): bool
    {
        return $this->created_at->addMinutes(60)->isPast();
    }

    /**
     * Verifica si el token proporcionado coincide con el hasheado.
     */
    public function isValid(string $plainToken): bool
    {
        return Hash::check($plainToken, $this->token);
    }
}
```

**Características obligatorias**:

- ✅ `final class extends Model`
- ✅ Primary key es `email` (string, no autoincremental)
- ✅ `$timestamps = false` (solo `created_at` manual)
- ✅ Cast para `email` → `EmailValueObject`
- ✅ Métodos `isExpired()` y `isValid()`

**Referencia**: Ver sección "2. Modelo PasswordResetToken" en `auth-class-diagram.md` líneas 529-561

---

### 2. Migración de PasswordResetToken

**Ubicación**: `Modules/Auth/database/migrations/2025_12_17_000002_create_password_reset_tokens_table.php`

**Especificaciones**:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('password_reset_tokens', function (Blueprint $table): void {
            // Email como PK
            $table->string('email', 255)->primary();

            // Token hasheado
            $table->string('token', 255);

            // Timestamp de creación (no usa timestamps estándar)
            $table->timestamp('created_at');

            // Índice para limpieza automática de expirados
            $table->index('created_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('password_reset_tokens');
    }
};
```

**Características obligatorias**:

- ✅ PK en `email` (no autoincremental)
- ✅ `token` hasheado con bcrypt (255 caracteres)
- ✅ Índice en `created_at` para limpieza de expirados
- ✅ NO usa `$table->timestamps()`
- ✅ `created_at` NO nullable (siempre tiene valor al crear)

**Referencia**: Ver sección "2. Modelo PasswordResetToken" en `auth-class-diagram.md` líneas 541-561

---

### 3. Repositorio PasswordResetTokenRepository

**Ubicación**: `Modules/Auth/app/Repositories/PasswordResetTokenRepository.php`

**Especificaciones**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Repositories;

use Modules\Auth\Contracts\Repositories\PasswordResetTokenRepositoryInterface;
use Modules\Auth\Models\PasswordResetToken;
use App\ValueObjects\EmailValueObject;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

final readonly class PasswordResetTokenRepository implements PasswordResetTokenRepositoryInterface
{
    /**
     * Crea un nuevo token de reset de password.
     * Genera token aleatorio de 60 caracteres y lo hashea.
     */
    public function create(EmailValueObject $email): PasswordResetToken
    {
        // Eliminar token anterior si existe
        $this->delete($email);

        // Generar token aleatorio (60 caracteres)
        $plainToken = Str::random(60);
        $hashedToken = Hash::make($plainToken);

        /** @var PasswordResetToken */
        $token = PasswordResetToken::create([
            'email' => $email,
            'token' => $hashedToken,
            'created_at' => now(),
        ]);

        // Guardar el token plano para enviarlo por email
        // (se guarda temporalmente en una propiedad pública)
        $token->plainToken = $plainToken;

        return $token;
    }

    /**
     * Busca un token por email.
     */
    public function findByEmail(EmailValueObject $email): ?PasswordResetToken
    {
        return PasswordResetToken::find($email->toString());
    }

    /**
     * Elimina un token por email.
     */
    public function delete(EmailValueObject $email): void
    {
        PasswordResetToken::where('email', $email->toString())->delete();
    }

    /**
     * Elimina tokens expirados (más de 60 minutos).
     * Ejecutado por Job diario.
     */
    public function deleteExpired(): int
    {
        $expiredAt = now()->subMinutes(60);

        return PasswordResetToken::where('created_at', '<', $expiredAt)->delete();
    }
}
```

**Características obligatorias**:

- ✅ `final readonly class`
- ✅ Implementa `PasswordResetTokenRepositoryInterface` (del Agente A)
- ✅ `create()` elimina token anterior y genera nuevo
- ✅ `create()` guarda token plano temporalmente en `$token->plainToken`
- ✅ `deleteExpired()` retorna cantidad de tokens eliminados

**Nota**: El token plano (`$plainToken`) solo existe en memoria durante la creación para enviarlo por email. NO se persiste en DB. La propiedad `$plainToken` está definida en el modelo como propiedad pública opcional para evitar problemas con PHPStan.

**Referencia**: Ver sección "7. Repository PasswordResetTokenRepository" en `auth-class-diagram.md` líneas 987-1007

---

### 4. Factory de PasswordResetToken

**Ubicación**: `Modules/Auth/database/factories/PasswordResetTokenFactory.php`

**Especificaciones**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Modules\Auth\Models\PasswordResetToken;
use App\ValueObjects\EmailValueObject;

/**
 * @extends Factory<PasswordResetToken>
 */
final class PasswordResetTokenFactory extends Factory
{
    protected $model = PasswordResetToken::class;

    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'email' => new EmailValueObject($this->faker->unique()->safeEmail()),
            'token' => Hash::make(Str::random(60)),
            'created_at' => now(),
        ];
    }

    /**
     * Estado: token expirado (más de 60 minutos).
     */
    public function expired(): self
    {
        return $this->state(fn (array $attributes): array => [
            'created_at' => now()->subMinutes(61),
        ]);
    }
}
```

**Características obligatorias**:

- ✅ `final class extends Factory`
- ✅ Estado `expired()` para testing de expiración
- ✅ Token hasheado con `Hash::make()`

**Referencia**: Ver sección "2. Modelo PasswordResetToken" en `auth-class-diagram.md` línea 558

---

## Tests a Crear

### Unit Tests para Modelo

**Ubicación**: `Modules/Auth/tests/Unit/Models/PasswordResetTokenTest.php`

**Tests obligatorios**:

```php
it('detecta correctamente si el token ha expirado', function () {
    $expired = PasswordResetToken::factory()->expired()->create();
    $valid = PasswordResetToken::factory()->create();

    expect($expired->isExpired())->toBeTrue()
        ->and($valid->isExpired())->toBeFalse();
});

it('valida correctamente un token plano contra el hasheado', function () {
    $plainToken = 'plain-token-123';
    $token = PasswordResetToken::factory()->create([
        'token' => Hash::make($plainToken),
    ]);

    expect($token->isValid($plainToken))->toBeTrue()
        ->and($token->isValid('wrong-token'))->toBeFalse();
});

it('castea correctamente el email a EmailValueObject', function () {
    $token = PasswordResetToken::factory()->create();

    expect($token->email)->toBeInstanceOf(EmailValueObject::class);
});

it('castea correctamente created_at a Carbon', function () {
    $token = PasswordResetToken::factory()->create();

    expect($token->created_at)->toBeInstanceOf(\Carbon\Carbon::class);
});
```

**Referencia**: Ver sección "Testing" en `auth-class-diagram.md` líneas 1170-1173

---

### Feature Tests para Repositorio

**Ubicación**: `Modules/Auth/tests/Feature/Repositories/PasswordResetTokenRepositoryTest.php`

**Tests obligatorios**:

```php
use Modules\Auth\Repositories\PasswordResetTokenRepository;

beforeEach(function () {
    $this->repository = new PasswordResetTokenRepository();
});

it('crea un token correctamente', function () {
    $email = new EmailValueObject('test@example.com');

    $token = $this->repository->create($email);

    expect($token)->toBeInstanceOf(PasswordResetToken::class)
        ->and($token->email->toString())->toBe('test@example.com')
        ->and($token->plainToken)->toBeString()
        ->and(strlen($token->plainToken))->toBe(60);
});

it('elimina token anterior al crear uno nuevo', function () {
    $email = new EmailValueObject('test@example.com');

    $token1 = $this->repository->create($email);
    $token2 = $this->repository->create($email);

    expect(PasswordResetToken::count())->toBe(1)
        ->and($token2->token)->not->toBe($token1->token);
});

it('encuentra un token por email', function () {
    $email = new EmailValueObject('find@example.com');
    $this->repository->create($email);

    $found = $this->repository->findByEmail($email);

    expect($found)->not->toBeNull()
        ->and($found->email->toString())->toBe('find@example.com');
});

it('retorna null si el token no existe', function () {
    $email = new EmailValueObject('notfound@example.com');

    $found = $this->repository->findByEmail($email);

    expect($found)->toBeNull();
});

it('elimina un token correctamente', function () {
    $email = new EmailValueObject('delete@example.com');
    $this->repository->create($email);

    $this->repository->delete($email);

    expect($this->repository->findByEmail($email))->toBeNull();
});

it('elimina tokens expirados correctamente', function () {
    // Token válido
    PasswordResetToken::factory()->create();

    // Tokens expirados
    PasswordResetToken::factory()->expired()->count(3)->create();

    $deleted = $this->repository->deleteExpired();

    expect($deleted)->toBe(3)
        ->and(PasswordResetToken::count())->toBe(1);
});
```

---

## Validación de Calidad

Antes de considerar completa la tarea:

1. ✅ **PHPStan level 6** sin errores
   ```bash
   ./vendor/bin/sail composer run phpstan
   ```

2. ✅ **Pint** ejecutado
   ```bash
   ./vendor/bin/sail bin pint --dirty
   ```

3. ✅ **Migración ejecuta sin errores**
   ```bash
   ./vendor/bin/sail artisan migrate:fresh
   ```

4. ✅ **Tests verdes** con cobertura del 100%
   ```bash
   ./vendor/bin/sail test Modules/Auth/tests/Unit/Models/PasswordResetTokenTest.php
   ./vendor/bin/sail test Modules/Auth/tests/Feature/Repositories/PasswordResetTokenRepositoryTest.php
   ```

5. ✅ **Factory funciona correctamente**
   ```bash
   ./vendor/bin/sail tinker
   >>> Modules\Auth\Models\PasswordResetToken::factory()->create()
   >>> Modules\Auth\Models\PasswordResetToken::factory()->expired()->create()
   ```

---

## Dependencias

### Input del Agente A

- ✅ `PasswordResetTokenRepositoryInterface` en `Modules/Auth/app/Contracts/Repositories/`
- ✅ `EmailValueObject` en `app/ValueObjects/`

### Input del Agente B

- ✅ Actions que usan el repositorio (`RequestPasswordResetAction`, `ResetPasswordAction`)

---

## Restricciones

❌ **NO crear**:
- Actions (ya creadas por Agente B)
- Lógica de envío de emails (va en Actions)
- Validaciones de negocio (va en Actions)

✅ **Solo crear**:
- Modelo PasswordResetToken
- Repositorio
- Migración
- Factory
- Tests

---

## Referencias

- **Diagrama de clases Auth**: `e-commerce-wa-ml/auth/auth-class-diagram.md` líneas 529-561, 987-1007
- **Guía Agente C**: `laravel/agents/agente-c.md` líneas 296-404
- **Convenciones del proyecto**: `e-commerce-wa-ml/project_definition.md`
- **Laravel Password Reset**: https://laravel.com/docs/passwords

---

## Notas

- El token se hashea con bcrypt antes de guardarlo en DB
- El token plano solo existe en memoria durante la creación (para enviarlo por email)
- La expiración es de 60 minutos desde `created_at`
- Solo un token activo por email (el anterior se elimina)
- El Job de limpieza se ejecuta diariamente (ver `agente-c-task-04`)
