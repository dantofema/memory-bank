---
task_id: "auth-003-persistence"
module: "Auth"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Auth - Modelo User, Migraciones y Persistencia"
priority: "HIGH"
estimated_time: "5 hours"
dependencies:
  - "auth-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
  - "@laravel/conventions/conventions.md"
phase: "Fase 2 - Persistencia"
---

# Task 003: Auth - Modelo User, Migraciones y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Auth: modelo User con Eloquent Casts para Value Objects, migraciones de base de datos, y Factory para testing. Esta capa maneja el almacenamiento y recuperación de datos de autenticación.

## ⚠️ ISSUES CRÍTICOS RESUELTOS

Esta task implementa correcciones obligatorias identificadas en la revisión de arquitectura:

### 1. Cast para MerchantName (CRÍTICO)
**Problema:** Solo se había planificado EmailCast, pero MerchantName también requiere un Cast.

**Solución:**
- Implementar `MerchantNameCast` que asegure la reconstrucción del VO desde la base de datos
- Garantizar normalización: trim, collapse multiple spaces
- Validar invariantes del VO en persistencia

**Riesgo sin Cast:**
- ❌ Base de datos almacenaría nombres sin normalizar
- ❌ Inconsistencia: "Test  Merchant" vs "Test Merchant"
- ❌ Ruptura de invariante de MerchantName (2-100 chars, normalized)
- ❌ Tests podrían pasar con datos inválidos en DB

### 2. Validación de Tipos en Casts (CRÍTICO)
**Problema:** Los Casts deben validar tipos estrictamente para prevenir persistencia de datos inválidos.

**Solución:**
- Validación obligatoria en `get()`: tipo string, no null
- Validación obligatoria en `set()`: MerchantName VO o string válido
- Lanzar `InvalidArgumentException` si tipos incorrectos
- Delegar validación de invariantes al VO

**Riesgo sin validación:**
- ❌ Persistencia de tipos incorrectos (int, array, etc.)
- ❌ Null values rompiendo constraints del dominio
- ❌ Bypass de validaciones del VO
- ❌ Errores silenciosos en producción

### 3. Consistencia de Persistencia
**Objetivo:** Garantizar que todos los Value Objects se persisten normalizados.

**Implementación:**
- EmailCast: normaliza a lowercase, trim
- MerchantNameCast: normaliza con trim, single spaces
- Tests de roundtrip: crear → guardar → recargar → validar normalización
- Tests de edge cases: validar rechazo de datos inválidos

## 📋 Contexto

El modelo User es el aggregate root del módulo Auth. Usa Eloquent Casts para mapear Value Objects a/desde la base de datos, garantizando que los datos siempre cumplan las invariantes del dominio.

### Referencias del Domain Model
- **Entity User:** (líneas 133-158)
- **Value Objects:** Email (líneas 189-226)
- **Database Schema:** (líneas 438-467)
- **Security:** Password hashing, indexes (líneas 562-590)

## 📦 Artefactos a Crear

**Total:** 6 archivos principales + 5 suites de tests

### Archivos de Código
1. **User Model** (`app/Models/User.php`) - Aggregate root con VOs
2. **EmailCast** (`app/Casts/EmailCast.php`) - Cast para Email VO
3. **MerchantNameCast** (`app/Casts/MerchantNameCast.php`) - ⚠️ CRÍTICO: Cast para MerchantName VO
4. **Migration: users** (`database/migrations/..._create_users_table.php`)
5. **Migration: sessions** (`database/migrations/..._create_sessions_table.php`)
6. **UserFactory** (`database/factories/UserFactory.php`)

### Tests
1. **User Model Tests** - Lifecycle, casts, FilamentUser contract
2. **EmailCast Tests** - Normalización, validación de tipos
3. **MerchantNameCast Tests** - ⚠️ CRÍTICO: Normalización, validación de invariantes
4. **Migration Tests** - Estructura DB, constraints, indexes
5. **Factory Tests** - Estados, passwords, validaciones

---

### 1. Modelo Eloquent: User

**Ubicación:** `app/Models/User.php` (Laravel core, NO en módulo)

**Especificaciones:**

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Modules\Auth\ValueObjects\Email;
use Modules\Auth\ValueObjects\MerchantName;
use App\Casts\EmailCast;
use App\Casts\MerchantNameCast;
use Filament\Models\Contracts\FilamentUser;
use Filament\Panel;

/**
 * User Model
 *
 * Represents a merchant with access to the Filament backoffice.
 *
 * @property int $id
 * @property MerchantName $name
 * @property Email $email
 * @property string $password
 * @property \Carbon\Carbon|null $email_verified_at
 * @property string|null $remember_token
 * @property \Carbon\Carbon $created_at
 * @property \Carbon\Carbon $updated_at
 */
final class User extends Authenticatable implements FilamentUser
{
    use HasFactory;
    use Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array<int, string>
     */
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    /**
     * The attributes that should be hidden for serialization.
     *
     * @var array<int, string>
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];

    /**
     * Get the attributes that should be cast.
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'name' => MerchantNameCast::class,
            'email' => EmailCast::class,
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }

    /**
     * Determine if the user can access the Filament panel.
     *
     * @param Panel $panel
     * @return bool
     */
    public function canAccessPanel(Panel $panel): bool
    {
        // Single-tenant: all users are merchants with backoffice access
        return true;
    }

    /**
     * Check if email has been verified.
     *
     * @return bool
     */
    public function hasVerifiedEmail(): bool
    {
        return $this->email_verified_at !== null;
    }

    /**
     * Mark email as verified.
     *
     * @return bool
     */
    public function markEmailAsVerified(): bool
    {
        $this->email_verified_at = now();
        return $this->save();
    }
}
```

**Reglas de Negocio:**
- **Name es MerchantName VO** (gracias al MerchantNameCast)
- Email siempre es Email VO (gracias al EmailCast)
- Password siempre hasheado con bcrypt (cast 'hashed')
- FilamentUser contract para acceso al backoffice
- Single-tenant: todos los users son merchants
- Email verification opcional (no bloqueante en MVP)
- Factory obligatorio para testing
- **Cast obligatorio para name:** Garantiza normalización y validación

---

### 2. Eloquent Cast: EmailCast

**Ubicación:** `app/Casts/EmailCast.php`

**Especificaciones:**

```php
namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use Modules\Auth\ValueObjects\Email;

/**
 * Cast Eloquent attribute to/from Email Value Object
 *
 * @implements CastsAttributes<Email, string>
 */
final class EmailCast implements CastsAttributes
{
    /**
     * Cast the given value from database to Email VO.
     *
     * @param array<string, mixed> $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): Email
    {
        if ($value === null) {
            throw new \InvalidArgumentException('Email cannot be null');
        }

        if (!is_string($value)) {
            throw new \InvalidArgumentException('Email must be a string');
        }

        return Email::fromString($value);
    }

    /**
     * Prepare the given value for storage in database.
     *
     * @param array<string, mixed> $attributes
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): string
    {
        if ($value instanceof Email) {
            return $value->normalized;
        }

        if (is_string($value)) {
            $email = Email::fromString($value);
            return $email->normalized;
        }

        throw new \InvalidArgumentException('Value must be Email VO or string');
    }
}
```

**Reglas de Negocio:**
- Siempre almacena email normalizado (lowercase, trim)
- Acepta Email VO o string en set()
- Siempre retorna Email VO en get()
- **VALIDACIÓN OBLIGATORIA:** Lanza excepciones si tipos inválidos
- Null no permitido (email es required)

---

### 3. Eloquent Cast: MerchantNameCast

**Ubicación:** `app/Casts/MerchantNameCast.php`

> ⚠️ **CRÍTICO:** Cast obligatorio para MerchantName. Sin este Cast, la base de datos podría almacenar nombres sin normalizar, rompiendo la invariante del Value Object.

**Especificaciones:**

```php
namespace App\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;
use Modules\Auth\ValueObjects\MerchantName;

/**
 * Cast Eloquent attribute to/from MerchantName Value Object
 *
 * Ensures merchant names are always normalized and validated
 * when persisting to/from database.
 *
 * @implements CastsAttributes<MerchantName, string>
 */
final class MerchantNameCast implements CastsAttributes
{
    /**
     * Cast the given value from database to MerchantName VO.
     *
     * @param array<string, mixed> $attributes
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): MerchantName
    {
        if ($value === null) {
            throw new \InvalidArgumentException('MerchantName cannot be null');
        }

        if (!is_string($value)) {
            throw new \InvalidArgumentException('MerchantName must be a string');
        }

        return MerchantName::fromString($value);
    }

    /**
     * Prepare the given value for storage in database.
     *
     * @param array<string, mixed> $attributes
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): string
    {
        if ($value instanceof MerchantName) {
            return $value->normalized;
        }

        if (is_string($value)) {
            $name = MerchantName::fromString($value);
            return $name->normalized;
        }

        throw new \InvalidArgumentException('Value must be MerchantName VO or string');
    }
}
```

**Reglas de Negocio:**
- Siempre almacena nombre normalizado (trim, single spaces)
- Acepta MerchantName VO o string en set()
- Siempre retorna MerchantName VO en get()
- **VALIDACIÓN OBLIGATORIA:** Lanza excepciones si tipos inválidos
- **VALIDACIÓN OBLIGATORIA:** Valida longitud mínima/máxima (2-100 chars)
- Null no permitido (name es required)
- Previene persistencia de nombres sin normalizar

---

### 4. Migration: create_users_table

**Ubicación:** `database/migrations/2024_01_01_000000_create_users_table.php`

**Especificaciones:**

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->timestamp('email_verified_at')->nullable();
            $table->rememberToken();
            $table->timestamps();

            // Indexes
            $table->index('email'); // Already unique, but explicit for clarity
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

**Reglas de Negocio:**
- Email unique constraint (un email = un usuario)
- Password string (hash bcrypt)
- Email verified nullable (verificación opcional)
- Remember token nullable
- Timestamps para auditoría
- NO soft deletes (single-tenant)

---

### 5. Migration: create_sessions_table

**Ubicación:** `database/migrations/2024_01_01_000001_create_sessions_table.php`

**Especificaciones:**

```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('sessions');
    }
};
```

**Reglas de Negocio:**
- Session driver: database (configurado en .env)
- User_id nullable (sesiones guest y autenticadas)
- IP address y user agent para auditoría
- Last activity indexado para limpieza de sesiones expiradas
- Payload longText para datos de sesión

---

### 6. Factory: UserFactory

**Ubicación:** `database/factories/UserFactory.php`

**Especificaciones:**

```php
namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

/**
 * @extends Factory<User>
 */
final class UserFactory extends Factory
{
    /**
     * The current password being used by the factory.
     */
    protected static ?string $password = null;

    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => static::$password ??= Hash::make('password'),
            'remember_token' => Str::random(10),
        ];
    }

    /**
     * Indicate that the model's email address should be unverified.
     */
    public function unverified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => null,
        ]);
    }

    /**
     * Indicate that the model's email address should be verified.
     */
    public function verified(): static
    {
        return $this->state(fn (array $attributes) => [
            'email_verified_at' => now(),
        ]);
    }

    /**
     * Indicate that the user has a remember token.
     */
    public function withRememberToken(): static
    {
        return $this->state(fn (array $attributes) => [
            'remember_token' => Str::random(60),
        ]);
    }

    /**
     * Indicate that the user does not have a remember token.
     */
    public function withoutRememberToken(): static
    {
        return $this->state(fn (array $attributes) => [
            'remember_token' => null,
        ]);
    }

    /**
     * Set a specific password for the user.
     */
    public function withPassword(string $password): static
    {
        return $this->state(fn (array $attributes) => [
            'password' => Hash::make($password),
        ]);
    }
}
```

**Reglas de Negocio:**
- Password default: "password" (hasheado)
- Email verified por default (estados: verified/unverified)
- Remember token generado por default
- Estados útiles para testing
- Password hasheado siempre (nunca texto plano)

---

## 🧪 Tests Requeridos

### Test 1: User Model

**Ubicación:** `tests/Unit/Models/UserTest.php`

**Casos a cubrir:**

```php
use App\Models\User;

describe('User Model', function () {
    it('can be instantiated', function () {
        $user = User::factory()->make();
        
        expect($user)->toBeInstanceOf(User::class)
            ->and($user->name)->toBeInstanceOf(\Modules\Auth\ValueObjects\MerchantName::class)
            ->and($user->email)->toBeInstanceOf(\Modules\Auth\ValueObjects\Email::class)
            ->and($user->password)->toBeString();
    });
    
    it('casts name to MerchantName VO', function () {
        $user = User::factory()->create([
            'name' => 'Test Merchant',
        ]);
        
        expect($user->name)->toBeInstanceOf(\Modules\Auth\ValueObjects\MerchantName::class)
            ->and($user->name->normalized)->toBe('Test Merchant');
    });
    
    it('casts email to Email VO', function () {
        $user = User::factory()->create([
            'email' => 'test@example.com',
        ]);
        
        expect($user->email)->toBeInstanceOf(\Modules\Auth\ValueObjects\Email::class)
            ->and($user->email->normalized)->toBe('test@example.com');
    });
    
    it('hides password in array', function () {
        $user = User::factory()->make();
        $array = $user->toArray();
        
        expect($array)->not->toHaveKey('password')
            ->and($array)->not->toHaveKey('remember_token');
    });
    
    it('implements FilamentUser contract', function () {
        $user = User::factory()->make();
        
        expect($user)->toBeInstanceOf(\Filament\Models\Contracts\FilamentUser::class);
    });
    
    it('can access Filament panel', function () {
        $user = User::factory()->make();
        $panel = mock(\Filament\Panel::class);
        
        expect($user->canAccessPanel($panel))->toBeTrue();
    });
    
    it('checks if email is verified', function () {
        $verifiedUser = User::factory()->verified()->make();
        $unverifiedUser = User::factory()->unverified()->make();
        
        expect($verifiedUser->hasVerifiedEmail())->toBeTrue()
            ->and($unverifiedUser->hasVerifiedEmail())->toBeFalse();
    });
    
    it('can mark email as verified', function () {
        $user = User::factory()->unverified()->create();
        
        expect($user->hasVerifiedEmail())->toBeFalse();
        
        $user->markEmailAsVerified();
        
        expect($user->hasVerifiedEmail())->toBeTrue()
            ->and($user->email_verified_at)->not->toBeNull();
    });
});
```

---

### Test 2: EmailCast

**Ubicación:** `tests/Unit/Casts/EmailCastTest.php`

**Casos a cubrir:**

```php
use App\Casts\EmailCast;
use Modules\Auth\ValueObjects\Email;
use App\Models\User;

describe('EmailCast', function () {
    it('casts string from database to Email VO', function () {
        $user = User::factory()->create([
            'email' => 'test@example.com',
        ]);
        
        $user = User::find($user->id);
        
        expect($user->email)->toBeInstanceOf(Email::class)
            ->and($user->email->normalized)->toBe('test@example.com');
    });
    
    it('casts Email VO to string for database', function () {
        $email = Email::fromString('test@example.com');
        
        $user = User::factory()->create([
            'email' => $email,
        ]);
        
        expect($user->email)->toBeInstanceOf(Email::class)
            ->and($user->getAttributes()['email'])->toBe('test@example.com');
    });
    
    it('accepts string and converts to Email VO', function () {
        $user = User::factory()->create([
            'email' => 'test@example.com',
        ]);
        
        $user->email = 'updated@example.com';
        $user->save();
        
        expect($user->fresh()->email->normalized)->toBe('updated@example.com');
    });
    
    it('normalizes email when casting to database', function () {
        $user = User::factory()->create([
            'email' => 'TEST@EXAMPLE.COM',
        ]);
        
        expect($user->getAttributes()['email'])->toBe('test@example.com');
    });
    
    it('throws exception for null email', function () {
        expect(fn() => User::factory()->create(['email' => null]))
            ->toThrow(\InvalidArgumentException::class);
    });
    
    it('throws exception for invalid type', function () {
        expect(fn() => User::factory()->create(['email' => 123]))
            ->toThrow(\InvalidArgumentException::class);
    });
});
```

---

### Test 3: MerchantNameCast

**Ubicación:** `tests/Unit/Casts/MerchantNameCastTest.php`

> ⚠️ **CRÍTICO:** Estos tests validan que MerchantName se normaliza y valida correctamente, previniendo inconsistencias en la base de datos.

**Casos a cubrir:**

```php
use App\Casts\MerchantNameCast;
use Modules\Auth\ValueObjects\MerchantName;
use App\Models\User;

describe('MerchantNameCast', function () {
    it('casts string from database to MerchantName VO', function () {
        $user = User::factory()->create([
            'name' => 'Test Merchant',
        ]);
        
        $user = User::find($user->id);
        
        expect($user->name)->toBeInstanceOf(MerchantName::class)
            ->and($user->name->normalized)->toBe('Test Merchant');
    });
    
    it('casts MerchantName VO to string for database', function () {
        $name = MerchantName::fromString('Test Merchant');
        
        $user = User::factory()->create([
            'name' => $name,
        ]);
        
        expect($user->name)->toBeInstanceOf(MerchantName::class)
            ->and($user->getAttributes()['name'])->toBe('Test Merchant');
    });
    
    it('accepts string and converts to MerchantName VO', function () {
        $user = User::factory()->create([
            'name' => 'Original Name',
        ]);
        
        $user->name = 'Updated Name';
        $user->save();
        
        expect($user->fresh()->name->normalized)->toBe('Updated Name');
    });
    
    it('normalizes merchant name when casting to database', function () {
        $user = User::factory()->create([
            'name' => '  Multiple   Spaces  Name  ',
        ]);
        
        // Should normalize: trim and collapse multiple spaces
        expect($user->getAttributes()['name'])->toBe('Multiple Spaces Name');
    });
    
    it('throws exception for null merchant name', function () {
        expect(fn() => User::factory()->create(['name' => null]))
            ->toThrow(\InvalidArgumentException::class, 'MerchantName cannot be null');
    });
    
    it('throws exception for invalid type in set', function () {
        $cast = new MerchantNameCast();
        $model = new User();
        
        expect(fn() => $cast->set($model, 'name', 123, []))
            ->toThrow(\InvalidArgumentException::class, 'Value must be MerchantName VO or string');
    });
    
    it('throws exception for invalid type in get', function () {
        $cast = new MerchantNameCast();
        $model = new User();
        
        expect(fn() => $cast->get($model, 'name', 123, []))
            ->toThrow(\InvalidArgumentException::class, 'MerchantName must be a string');
    });
    
    it('validates minimum length through VO', function () {
        // MerchantName VO should validate min length
        expect(fn() => User::factory()->create(['name' => 'A']))
            ->toThrow(\InvalidArgumentException::class);
    });
    
    it('validates maximum length through VO', function () {
        // MerchantName VO should validate max length (100 chars)
        $longName = str_repeat('A', 101);
        
        expect(fn() => User::factory()->create(['name' => $longName]))
            ->toThrow(\InvalidArgumentException::class);
    });
    
    it('preserves normalized invariant on persistence roundtrip', function () {
        $originalName = '  Test   Merchant   Name  ';
        $expectedNormalized = 'Test Merchant Name';
        
        $user = User::factory()->create(['name' => $originalName]);
        
        // Verify database has normalized value
        expect($user->getAttributes()['name'])->toBe($expectedNormalized);
        
        // Verify VO has normalized value
        expect($user->name->normalized)->toBe($expectedNormalized);
        
        // Verify after reload from DB
        $user->refresh();
        expect($user->name->normalized)->toBe($expectedNormalized);
    });
});
```

---

### Test 4: User Migration and Database

**Ubicación:** `tests/Feature/Database/UserMigrationTest.php`

**Casos a cubrir:**

```php
use App\Models\User;
use Illuminate\Support\Facades\Schema;

describe('User Migration', function () {
    it('creates users table with correct columns', function () {
        expect(Schema::hasTable('users'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'id'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'name'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'email'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'password'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'email_verified_at'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'remember_token'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'created_at'))->toBeTrue()
            ->and(Schema::hasColumn('users', 'updated_at'))->toBeTrue();
    });
    
    it('enforces unique constraint on email', function () {
        User::factory()->create(['email' => 'duplicate@example.com']);
        
        expect(fn() => User::factory()->create(['email' => 'duplicate@example.com']))
            ->toThrow(\Illuminate\Database\QueryException::class);
    });
    
    it('allows nullable email_verified_at', function () {
        $user = User::factory()->unverified()->create();
        
        expect($user->email_verified_at)->toBeNull();
    });
    
    it('allows nullable remember_token', function () {
        $user = User::factory()->withoutRememberToken()->create();
        
        expect($user->remember_token)->toBeNull();
    });
});

describe('Sessions Migration', function () {
    it('creates sessions table with correct columns', function () {
        expect(Schema::hasTable('sessions'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'id'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'user_id'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'ip_address'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'user_agent'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'payload'))->toBeTrue()
            ->and(Schema::hasColumn('sessions', 'last_activity'))->toBeTrue();
    });
    
    it('allows nullable user_id for guest sessions', function () {
        DB::table('sessions')->insert([
            'id' => 'test-session-id',
            'user_id' => null,
            'ip_address' => '127.0.0.1',
            'user_agent' => 'Test Agent',
            'payload' => 'test-payload',
            'last_activity' => time(),
        ]);
        
        $session = DB::table('sessions')->where('id', 'test-session-id')->first();
        
        expect($session->user_id)->toBeNull();
    });
});
```

---

### Test 5: UserFactory

**Ubicación:** `tests/Unit/Factories/UserFactoryTest.php`

**Casos a cubrir:**

```php
use App\Models\User;

describe('UserFactory', function () {
    it('creates user with default state', function () {
        $user = User::factory()->create();
        
        expect($user->name)->toBeString()
            ->and($user->email)->toBeInstanceOf(\Modules\Auth\ValueObjects\Email::class)
            ->and($user->password)->toBeString()
            ->and($user->email_verified_at)->not->toBeNull()
            ->and($user->remember_token)->not->toBeNull();
    });
    
    it('creates verified user', function () {
        $user = User::factory()->verified()->create();
        
        expect($user->hasVerifiedEmail())->toBeTrue();
    });
    
    it('creates unverified user', function () {
        $user = User::factory()->unverified()->create();
        
        expect($user->hasVerifiedEmail())->toBeFalse();
    });
    
    it('creates user with remember token', function () {
        $user = User::factory()->withRememberToken()->create();
        
        expect($user->remember_token)->not->toBeNull()
            ->and(strlen($user->remember_token))->toBe(60);
    });
    
    it('creates user without remember token', function () {
        $user = User::factory()->withoutRememberToken()->create();
        
        expect($user->remember_token)->toBeNull();
    });
    
    it('creates user with custom password', function () {
        $user = User::factory()->withPassword('custom123')->create();
        
        expect(Hash::check('custom123', $user->password))->toBeTrue();
    });
    
    it('hashes passwords automatically', function () {
        $user = User::factory()->create();
        
        // Password should be hashed, not plain text
        expect($user->password)->not->toBe('password')
            ->and(Hash::check('password', $user->password))->toBeTrue();
    });
    
    it('generates unique emails', function () {
        $user1 = User::factory()->create();
        $user2 = User::factory()->create();
        
        expect($user1->email->normalized)->not->toBe($user2->email->normalized);
    });
});
```

---

## ✅ Criterios de Aceptación

### Funcionales
- [ ] User model se puede crear y persistir
- [ ] **Name se convierte automáticamente a MerchantName VO**
- [ ] **Name se normaliza antes de guardar en DB (trim, single spaces)**
- [ ] Email se convierte automáticamente a Email VO
- [ ] Email se normaliza antes de guardar en DB
- [ ] Password se hashea automáticamente (cast 'hashed')
- [ ] Unique constraint en email funciona correctamente
- [ ] Sessions table almacena sesiones correctamente
- [ ] Factory genera usuarios válidos
- [ ] Estados del Factory (verified/unverified) funcionan
- [ ] FilamentUser contract implementado correctamente

### Técnicos
- [ ] User model es `final class`
- [ ] **MerchantNameCast implementa CastsAttributes correctamente**
- [ ] **MerchantNameCast valida tipos y lanza excepciones**
- [ ] EmailCast implementa CastsAttributes correctamente
- [ ] EmailCast valida tipos y lanza excepciones
- [ ] Tipado fuerte completo en model y casts
- [ ] Docblocks con `@property` en modelo (name: MerchantName, email: Email)
- [ ] Migraciones ejecutan sin errores
- [ ] Índices creados correctamente
- [ ] Factory es `final class`
- [ ] Tests con Pest 4 (describe/it syntax)
- [ ] **Tests de MerchantNameCast con cobertura completa**
- [ ] **Tests de validación de invariantes del VO**
- [ ] Cobertura de tests: 100%
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado sin advertencias

### Base de Datos
- [ ] Tabla users creada con todas las columnas
- [ ] Tabla sessions creada correctamente
- [ ] Index unique en email
- [ ] Index en last_activity de sessions
- [ ] Foreign key constraints no aplican (módulo base)
- [ ] Migraciones pueden rollback sin errores

### Documentación
- [ ] Docblocks en User model completos
- [ ] `@property` para todas las propiedades del modelo
- [ ] Comments en migraciones explicando constraints
- [ ] Factory methods documentados

## 🔧 Comandos de Validación

```bash
# Ejecutar migraciones
./vendor/bin/sail artisan migrate:fresh

# Verificar estructura de tablas
./vendor/bin/sail artisan db:show

# Ejecutar tests de persistencia
./vendor/bin/sail test tests/Unit/Models
./vendor/bin/sail test tests/Unit/Casts/EmailCastTest.php
./vendor/bin/sail test tests/Unit/Casts/MerchantNameCastTest.php
./vendor/bin/sail test tests/Feature/Database

# Análisis estático
./vendor/bin/sail composer run phpstan -- --paths=app/Models/User.php,app/Casts

# Formateo de código
./vendor/bin/sail bin pint app/Models/User.php
./vendor/bin/sail bin pint app/Casts
./vendor/bin/sail bin pint database/factories
```

## 📚 Referencias

- **Domain Model:** `@e-commerce-wa-ml/auth/domain_model.md` (líneas 133-158, 438-467)
- **Metodología:** `@laravel/agents/agent-c-persistence.md`
- **Database Schema:** Auth module (líneas 438-467)
- **Value Objects:** Email (líneas 189-226)

## 📝 Notas de Implementación

### User Model
- Extends `Authenticatable` (no `Model`)
- FilamentUser contract required para backoffice
- `canAccessPanel()` retorna true (single-tenant)
- Cast 'hashed' para password (auto-hashing)
- **Cast obligatorio para name (MerchantNameCast):** Garantiza normalización
- Cast obligatorio para email (EmailCast): Garantiza normalización
- NO soft deletes (single-tenant, un merchant)

### MerchantNameCast
- **CRÍTICO:** Sin este Cast, nombres sin normalizar romperían invariantes
- Siempre almacena normalizado (trim, collapse multiple spaces)
- Acepta MerchantName VO o string en set()
- Convierte a MerchantName VO en get()
- **Validación de tipos obligatoria:** Lanza excepciones si tipos inválidos
- **Validación de invariantes:** Delega a MerchantName VO (min/max length)
- No permite null (name es required)
- Previene inconsistencias de persistencia

### EmailCast
- Siempre almacena normalizado (lowercase, trim)
- Acepta Email VO o string en set()
- Convierte a Email VO en get()
- **Validación de tipos obligatoria:** Lanza excepciones si tipos inválidos
- No permite null (email es required)

### Migraciones
- users table: unique email, nullable email_verified_at
- sessions table: nullable user_id (guest sessions)
- NO foreign keys (módulo base sin dependencias)
- Index en email para búsquedas rápidas
- Index en last_activity para limpieza de sesiones

### Factory
- Password default: "password" (conveniente para testing)
- Estados útiles: verified, unverified, withRememberToken, withoutRememberToken
- withPassword() para tests con credenciales específicas
- Email siempre único (fake()->unique()->safeEmail())
- Remember token default: 10 chars (suficiente para tests)

### Testing Strategy
- Unit tests para model y cast aislados
- Feature tests para integración con DB
- Tests de constraints (unique email)
- Tests de estados del Factory
- Tests de nullable fields
- Verificar auto-hashing de passwords

### Edge Cases a Testear
- Email duplicado (debe fallar)
- Email null (debe fallar)
- **Name null (debe fallar)**
- **Name con múltiples espacios (debe normalizarse)**
- **Name muy corto (<2 chars) (debe fallar)**
- **Name muy largo (>100 chars) (debe fallar)**
- **Name con tipo inválido (debe fallar)**
- Password sin hashear (Factory debe hashear)
- Sessions con user_id null (guest sessions)
- Email con uppercase (debe normalizarse)
- **Persistencia roundtrip: normalización debe mantenerse**

### Configuración Required
Asegurarse de que `.env` tiene:
```env
SESSION_DRIVER=database
SESSION_LIFETIME=120
```

---

**Status:** ✅ Ready to Implement  
**Fase:** 2 - Persistencia  
**Bloqueante para:** Task 002 (Actions), Task 004 (HTTP/Filament)  
**Depende de:** Task 001 (Contracts)
