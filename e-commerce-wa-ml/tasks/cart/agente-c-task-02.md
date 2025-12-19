---
task_id: "agente-c-task-02"
module: "Auth"
agent: "Agente C"
title: "Migración de tabla users"
priority: "high"
estimated_time: "1h"
dependencies:
  - "agente-c-task-01: Modelo User y Casts"
status: "pending"
---

# Task: Migración de tabla users

## Contexto

Crear la migración para la tabla `users` con todos los campos necesarios, índices y tipos correctos para soportar los Value Objects y el enum `MerchantRoleEnum`.

**Ubicación**: `database/migrations/2025_12_17_000001_create_users_table.php`

---

## Objetivos

1. ✅ Crear migración de tabla `users`
2. ✅ Incluir índices para optimizar consultas
3. ✅ Tipos de datos correctos
4. ✅ Validar migración ejecuta sin errores

---

## Especificaciones de la Migración

### Campos de la tabla

**Ubicación**: `database/migrations/2025_12_17_000001_create_users_table.php`

**Estructura**:

```php
Schema::create('users', function (Blueprint $table): void {
    // ID autoincremental
    $table->id();

    // Campos obligatorios (mapeados a Value Objects)
    $table->string('name', 255);
    $table->string('email', 255)->unique();
    $table->string('password', 255);

    // Role (Enum)
    $table->string('role', 50)->default('owner');

    // Remember token para "Recordarme"
    $table->rememberToken();

    // Timestamps
    $table->timestamps();

    // Índices para optimizar consultas
    $table->index('email');
    $table->index('role');
    $table->index('created_at');
});
```

---

## Características Obligatorias

- ✅ `declare(strict_types=1)`
- ✅ Tipar `up()` y `down()` con `: void`
- ✅ Incluir índice en `email` (además del unique)
- ✅ Incluir índice en `role` (para futuras queries)
- ✅ Incluir índice en `created_at` (para ordenamiento temporal)
- ✅ Valor por defecto en `role` = `'owner'`
- ✅ Método `down()` con `Schema::dropIfExists('users')`

**Referencia**: Ver ejemplo en `agente-c.md` líneas 421-473

---

## Mapeo de Campos a Value Objects

| Campo DB      | Tipo DB         | Value Object        | Cast                 |
|---------------|-----------------|---------------------|----------------------|
| `id`          | `bigIncrements` | `int`               | (nativo)             |
| `name`        | `string(255)`   | `NameValueObject`   | `NameCast`           |
| `email`       | `string(255)`   | `EmailValueObject`  | `EmailCast`          |
| `password`    | `string(255)`   | `PasswordValueObject` | `PasswordCast`     |
| `role`        | `string(50)`    | `MerchantRoleEnum`  | (Cast nativo enum)   |
| `remember_token` | `string(100)` | `string` (nullable) | (nativo)          |

---

## Tests de Migración

**Ubicación**: `tests/Feature/Database/UserMigrationTest.php`

**Tests obligatorios**:

```php
it('ejecuta la migración sin errores', function () {
    Artisan::call('migrate:fresh');
    
    expect(Schema::hasTable('users'))->toBeTrue();
});

it('tiene todas las columnas requeridas', function () {
    expect(Schema::hasColumns('users', [
        'id',
        'name',
        'email',
        'password',
        'role',
        'remember_token',
        'created_at',
        'updated_at',
    ]))->toBeTrue();
});

it('tiene índice único en email', function () {
    $indexes = Schema::getIndexes('users');
    $emailIndex = collect($indexes)->first(fn($idx) => in_array('email', $idx['columns']));
    
    expect($emailIndex)->not->toBeNull()
        ->and($emailIndex['unique'])->toBeTrue();
});

it('tiene índices en role y created_at', function () {
    $indexes = Schema::getIndexes('users');
    $indexColumns = collect($indexes)->pluck('columns')->flatten();
    
    expect($indexColumns)->toContain('role')
        ->and($indexColumns)->toContain('created_at');
});

it('el campo role tiene valor por defecto owner', function () {
    $table = Schema::getColumns('users');
    $roleColumn = collect($table)->firstWhere('name', 'role');
    
    expect($roleColumn['default'])->toBe('owner');
});
```

---

## Validación de Calidad

Antes de considerar completa la tarea:

1. ✅ **Migración ejecuta sin errores**
   ```bash
   ./vendor/bin/sail artisan migrate:fresh
   ```

2. ✅ **Tabla creada correctamente**
   ```bash
   ./vendor/bin/sail artisan tinker
   >>> Schema::hasTable('users')
   => true
   ```

3. ✅ **Índices creados correctamente**
   ```bash
   ./vendor/bin/sail artisan db:show
   ./vendor/bin/sail artisan db:table users
   ```

4. ✅ **PHPStan level 6** sin errores
   ```bash
   ./vendor/bin/sail composer run phpstan
   ```

5. ✅ **Pint** ejecutado
   ```bash
   ./vendor/bin/sail bin pint --dirty
   ```

6. ✅ **Tests verdes**
   ```bash
   ./vendor/bin/sail test tests/Feature/Database/UserMigrationTest.php
   ```

---

## Seeders (Opcional para Testing)

**Ubicación**: `database/seeders/UserSeeder.php`

**Especificaciones**:

- ✅ Crear usuario owner para desarrollo
- ✅ Crear usuarios admin para testing

```php
final class UserSeeder extends Seeder
{
    public function run(): void
    {
        // Owner para desarrollo
        User::factory()->owner()->create([
            'email' => new EmailValueObject('admin@example.com'),
            'password' => PasswordValueObject::fromPlain('password'),
        ]);

        // Admins para testing
        User::factory()->admin()->count(3)->create();
    }
}
```

**Nota**: Solo crear si se necesita para testing local.

---

## Restricciones

❌ **NO crear**:
- Lógica de negocio en la migración
- Validaciones de negocio
- Seeds con datos de producción

✅ **Solo crear**:
- Migración de tabla
- Índices
- Tests de migración

---

## Referencias

- **Diagrama de clases Auth**: `e-commerce-wa-ml/auth/auth-class-diagram.md` líneas 488-527
- **Guía Agente C**: `laravel/agents/agente-c.md` líneas 421-473
- **Convenciones del proyecto**: `e-commerce-wa-ml/project_definition.md` líneas 368-382
- **Laravel Migrations**: https://laravel.com/docs/migrations

---

## Notas

- La tabla `users` es parte del core de Laravel, NO del módulo Auth
- Los índices son obligatorios para columnas que se usan en WHERE, ORDER BY
- El valor por defecto de `role` es `'owner'` (string, no enum case)
- El campo `remember_token` es manejado automáticamente por Laravel
