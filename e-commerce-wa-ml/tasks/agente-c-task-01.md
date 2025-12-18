---
task_id: "agente-c-task-01"
module: "Auth"
agent: "Agente C"
title: "Modelo User y Value Objects Casts"
priority: "high"
estimated_time: "2h"
dependencies:
  - "Agente A: User contracts y Value Objects"
  - "Agente B: Actions implementadas"
status: "pending"
---

# Task: Modelo User y Value Objects Casts

## Contexto

Implementar el modelo `User` en la raíz del proyecto (`app/Models/User.php`) con los Casts necesarios para mapear Value Objects (`EmailValueObject`, `NameValueObject`, `PasswordValueObject`) y el enum `MerchantRoleEnum`.

**Ubicación**: `app/Models/User.php` (core de Laravel, NO en módulo Auth)

---

## Objetivos

1. ✅ Crear modelo `User` con tipado fuerte
2. ✅ Implementar Casts para Value Objects
3. ✅ Configurar enum `MerchantRoleEnum`
4. ✅ Crear Factory con estados
5. ✅ Tests unitarios del modelo y Casts

---

## Archivos a Crear

### 1. Modelo User

**Ubicación**: `app/Models/User.php`

**Especificaciones**:

- ✅ `final class User extends Authenticatable` (de `Illuminate\Foundation\Auth\User`)
- ✅ Docblock con `@property` para todas las propiedades
- ✅ Configurar Casts para Value Objects:
  - `name` → `NameCast::class`
  - `email` → `EmailCast::class`
  - `password` → `PasswordCast::class`
  - `role` → `MerchantRoleEnum::class`
- ✅ `$fillable`: `['name', 'email', 'password', 'role']`
- ✅ `$hidden`: `['password', 'remember_token']`

**Referencia**: Ver sección "1. Modelo User" en `auth-class-diagram.md` líneas 488-527

---

### 2. Casts para Value Objects

#### EmailCast

**Ubicación**: `app/Casts/EmailCast.php`

**Especificaciones**:

- ✅ `final class implements CastsAttributes`
- ✅ Docblock: `@implements CastsAttributes<EmailValueObject, EmailValueObject>`
- ✅ `get()`: string → `EmailValueObject`
- ✅ `set()`: `EmailValueObject` → string
- ✅ Validar tipos y lanzar `InvalidArgumentException` si inválido

**Referencia**: Ver sección "3. Value Object EmailValueObject" en `auth-class-diagram.md` líneas 563-602

---

#### NameCast

**Ubicación**: `app/Casts/NameCast.php`

**Especificaciones**:

- ✅ `final class implements CastsAttributes`
- ✅ Docblock: `@implements CastsAttributes<NameValueObject, NameValueObject>`
- ✅ `get()`: string → `NameValueObject`
- ✅ `set()`: `NameValueObject` → string
- ✅ Validar tipos y lanzar `InvalidArgumentException` si inválido

**Referencia**: Ver sección "3.2. Value Object NameValueObject" en `auth-class-diagram.md` líneas 672-707

---

#### PasswordCast

**Ubicación**: `app/Casts/PasswordCast.php`

**Especificaciones**:

- ✅ `final class implements CastsAttributes`
- ✅ Docblock: `@implements CastsAttributes<PasswordValueObject, PasswordValueObject>`
- ✅ `get()`: string (hash) → `PasswordValueObject`
- ✅ `set()`: `PasswordValueObject` → string (hash)
- ✅ Validar tipos y lanzar `InvalidArgumentException` si inválido

**⚠️ Importante**:
- El constructor de `PasswordValueObject` recibe el hash directamente (cuando se lee de DB)
- Usar `PasswordValueObject::fromPlain()` para crear desde password plano

**Referencia**: Ver sección "3.1. Value Object PasswordValueObject" en `auth-class-diagram.md` líneas 604-670

---

### 3. Factory de User

**Ubicación**: `database/factories/UserFactory.php`

**Especificaciones**:

- ✅ `final class extends Factory`
- ✅ Docblock: `@extends Factory<User>`
- ✅ `definition()`: generar datos fake con Value Objects
- ✅ Estado `owner()`: crea usuario con rol `OWNER`
- ✅ Estado `admin()`: crea usuario con rol `ADMIN`

**Ejemplo de definition()**:

```php
return [
    'name' => new NameValueObject($this->faker->name()),
    'email' => new EmailValueObject($this->faker->unique()->safeEmail()),
    'password' => PasswordValueObject::fromPlain('password'),
    'role' => MerchantRoleEnum::ADMIN,
];
```

**Referencia**: Ver sección "1. Modelo User" en `auth-class-diagram.md` línea 520

---

## Tests a Crear

### Unit Tests para Modelo User

**Ubicación**: `tests/Unit/Models/UserTest.php`

**Tests obligatorios**:

```php
it('castea correctamente el email a EmailValueObject')
it('castea correctamente el name a NameValueObject')
it('castea correctamente el password a PasswordValueObject')
it('castea correctamente el role a MerchantRoleEnum')
it('verifica password correctamente con PasswordValueObject')
it('oculta password en array serializado')
it('oculta remember_token en array serializado')
it('factory genera usuario válido')
it('factory estado owner crea usuario OWNER')
it('factory estado admin crea usuario ADMIN')
```

**Referencia**: Ver sección "Testing" en `auth-class-diagram.md` líneas 1159-1275

---

### Unit Tests para Casts

**Ubicación**: `tests/Unit/Casts/`

**Nota**: Los tests de Casts pueden ejecutarse en cualquier orden (son independientes entre sí).

#### EmailCastTest.php

```php
it('castea correctamente de DB a EmailValueObject')
it('castea correctamente de EmailValueObject a DB')
it('lanza excepción si el valor en get es null')
it('lanza excepción si el valor en set no es EmailValueObject')
```

#### NameCastTest.php

```php
it('castea correctamente de DB a NameValueObject')
it('castea correctamente de NameValueObject a DB')
it('lanza excepción si el valor en get es null')
it('lanza excepción si el valor en set no es NameValueObject')
```

#### PasswordCastTest.php

```php
it('castea correctamente de DB (hash) a PasswordValueObject')
it('castea correctamente de PasswordValueObject a DB (hash)')
it('lanza excepción si el valor en get es null')
it('lanza excepción si el valor en set no es PasswordValueObject')
it('no modifica el hash en el cast')
```

**Referencia**: Ver ejemplo en `agente-c.md` líneas 627-677

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

3. ✅ **Tests verdes** con cobertura del 100%
   ```bash
   ./vendor/bin/sail test tests/Unit/Models/UserTest.php
   ./vendor/bin/sail test tests/Unit/Casts/
   ```

4. ✅ **Factory funciona correctamente**
   ```bash
   ./vendor/bin/sail tinker
   >>> App\Models\User::factory()->owner()->create()
   >>> App\Models\User::factory()->admin()->count(5)->create()
   ```

---

## Dependencias

### Input del Agente A

- ✅ `EmailValueObject` en `app/ValueObjects/EmailValueObject.php`
- ✅ `NameValueObject` en `app/ValueObjects/NameValueObject.php`
- ✅ `PasswordValueObject` en `app/ValueObjects/PasswordValueObject.php`
- ✅ `MerchantRoleEnum` en `app/Enums/MerchantRoleEnum.php`

### Input del Agente B

- ✅ Actions implementadas (para entender flujo)

---

## Restricciones

❌ **NO crear**:
- Actions (ya creadas por Agente B)
- Controllers
- Filament Resources/Pages
- Lógica de negocio

✅ **Solo crear**:
- Modelo User
- Casts
- Factory
- Tests unitarios

---

## Referencias

- **Diagrama de clases Auth**: `e-commerce-wa-ml/auth/auth-class-diagram.md`
- **Guía Agente C**: `laravel/agents/agente-c.md`
- **Convenciones del proyecto**: `e-commerce-wa-ml/project_definition.md`
- **Laravel Eloquent Casts**: https://laravel.com/docs/eloquent-mutators#custom-casts

---

## Notas

- El modelo `User` vive en `app/Models/` (core de Laravel), NO en el módulo Auth
- Los Casts viven en `app/Casts/` para ser reutilizables entre módulos
- Los Value Objects ya fueron creados por el Agente A
- El Factory vive en `database/factories/` (convención Laravel)
