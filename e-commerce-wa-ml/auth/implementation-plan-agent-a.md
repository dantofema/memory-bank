---
name: "Plan de Implementación - Agente A - Módulo Auth"
version: "1.0"
author: "Alejandro Leone"
created: "2025-12-18"
module: "Auth"
agent: "Agente A"
purpose: "Plan detallado para implementar la frontera pública del módulo Auth"
dependencies:
  - auth-class-diagram.md
  - ../../laravel/agents/agente-a.md
  - ../../laravel/conventions/conventions.md
status: "pending"
---

# Plan de Implementación - Agente A - Módulo Auth

## Resumen Ejecutivo

Este documento define el plan paso a paso para implementar la **frontera pública del módulo Auth** según las
responsabilidades del **Agente A**.

**Alcance del Agente A:**

- ✅ Value Objects (Email, Name, Password)
- ✅ Enums (MerchantRoleEnum)
- ✅ Data Objects (LoginData, RequestPasswordResetData, ResetPasswordData, AuthenticationResult)
- ✅ Contracts (si aplican para comunicación inter-módulos)
- ✅ Tests unitarios (100% cobertura)

**Fuera del alcance (otros agentes):**

- ❌ Actions (AuthenticateMerchantAction, etc.)
- ❌ Models (User, PasswordResetToken)
- ❌ Repositories
- ❌ Custom Filament Pages
- ❌ Commands
- ❌ Jobs
- ❌ Migrations

---

## Precondiciones Globales

Antes de comenzar cualquier tarea, verificar:

1. **Tests baseline pasan:**
   ```bash
   ./vendor/bin/sail test
   ```

---

## Estructura de Tareas

### 📦 Tarea 1: Value Objects Compartidos

**Prioridad:** ALTA (todo depende de esto)  
**Estimación:** 2-3 horas  
**Archivos a crear:** 6 archivos

#### Precondiciones

- Directorio `app/ValueObjects/` existe
- Directorio `app/Casts/` existe
- Directorio `tests/Unit/ValueObjects/` existe

#### Archivos a Crear

1. **`app/ValueObjects/EmailValueObject.php`**
    - Validación de formato email con `filter_var`
    - Implementa `Wireable` para Livewire
    - Constructor valida y lanza `InvalidArgumentException`
    - Métodos: `toString()`, `toLivewire()`, `fromLivewire()`

2. **`app/Casts/EmailCast.php`**
    - Extiende `Illuminate\Contracts\Database\Eloquent\CastsAttributes`
    - `get()`: string → EmailValueObject
    - `set()`: EmailValueObject → string

3. **`app/ValueObjects/NameValueObject.php`**
    - Normalización: trim, capitalizar
    - Validación: min 2, max 255 caracteres
    - Implementa `Wireable`
    - Métodos: `toString()`, `toUpperCase()`, `toLivewire()`, `fromLivewire()`

4. **`app/Casts/NameCast.php`**
    - `get()`: string → NameValueObject
    - `set()`: NameValueObject → string

5. **`app/ValueObjects/PasswordValueObject.php`**
    - Constructor recibe hash (desde DB)
    - `static fromPlain(string)`: hashea con `Hash::make()`
    - `verify(string): bool`: verifica con `Hash::check()`
    - Implementa `Wireable`
    - Métodos: `toString()`, `toLivewire()`, `fromLivewire()`

6. **`app/Casts/PasswordCast.php`**
    - `get()`: string (hash) → PasswordValueObject
    - `set()`: PasswordValueObject → string (hash)

#### Tests a Crear

**`tests/Unit/ValueObjects/EmailValueObjectTest.php`**

- ✅ Crea email válido
- ✅ Lanza excepción con email inválido (sin @)
- ✅ Lanza excepción con email vacío
- ✅ Lanza excepción con email muy largo (>255)
- ✅ Método `toString()` retorna el email
- ✅ Serialización Livewire (`toLivewire()` / `fromLivewire()`)

**`tests/Unit/ValueObjects/NameValueObjectTest.php`**

- ✅ Crea nombre válido
- ✅ Normaliza espacios múltiples
- ✅ Capitaliza primera letra de cada palabra
- ✅ Lanza excepción con nombre vacío
- ✅ Lanza excepción con nombre muy corto (<2)
- ✅ Lanza excepción con nombre muy largo (>255)
- ✅ Método `toUpperCase()` retorna mayúsculas
- ✅ Serialización Livewire

**`tests/Unit/ValueObjects/PasswordValueObjectTest.php`**

- ✅ Constructor acepta hash válido
- ✅ `fromPlain()` hashea correctamente
- ✅ `verify()` valida password correcto
- ✅ `verify()` rechaza password incorrecto
- ✅ Dos `fromPlain()` con mismo password generan hashes diferentes (bcrypt salt)
- ✅ `toString()` retorna el hash
- ✅ Serialización Livewire

#### Validaciones Postcondición

```bash
# Formatear código
./vendor/bin/sail bin pint app/ValueObjects/ app/Casts/

# Análisis estático
./vendor/bin/sail composer run phpstan -- --level=6 app/ValueObjects/ app/Casts/

# Tests unitarios
./vendor/bin/sail test tests/Unit/ValueObjects/

# Cobertura debe ser 100%
./vendor/bin/sail test --coverage --min=100 tests/Unit/ValueObjects/
```

#### Criterios de Completitud

- [ ] 3 Value Objects creados y funcionan
- [ ] 3 Casts creados y mapean correctamente
- [ ] 3 archivos de tests con cobertura 100%
- [ ] PHPStan level 6 sin errores
- [ ] Pint sin cambios pendientes

---

### 📦 Tarea 2: Enum MerchantRoleEnum

**Prioridad:** ALTA  
**Estimación:** 1 hora  
**Archivos a crear:** 2 archivos

#### Precondiciones

- Directorio `app/Enums/` existe
- Directorio `tests/Unit/Enums/` existe

#### Archivos a Crear

1. **`app/Enums/MerchantRoleEnum.php`**
    - `enum MerchantRoleEnum: string`
    - Casos: `OWNER = 'owner'`, `ADMIN = 'admin'`
    - Método `label(): string` (Propietario, Administrador)
    - Método `permissions(): array` (preparado para futuro)
    - Método `canManageUsers(): bool` (solo OWNER = true)

#### Tests a Crear

**`tests/Unit/Enums/MerchantRoleEnumTest.php`**

- ✅ Casos existen (OWNER, ADMIN)
- ✅ Valores son strings correctos
- ✅ Labels en español
- ✅ `canManageUsers()` solo true para OWNER
- ✅ `permissions()` retorna array (aunque vacío en MVP)

#### Validaciones Postcondición

```bash
./vendor/bin/sail bin pint app/Enums/
./vendor/bin/sail composer run phpstan -- --level=6 app/Enums/
./vendor/bin/sail test tests/Unit/Enums/
```

#### Criterios de Completitud

- [ ] Enum MerchantRoleEnum creado con 2 casos
- [ ] 3 métodos helper implementados
- [ ] Tests con cobertura 100%
- [ ] PHPStan level 6 sin errores

---

### 📦 Tarea 3: Data Objects del Módulo Auth

**Prioridad:** MEDIA (depende de Tarea 1 y 2)  
**Estimación:** 2 horas  
**Archivos a crear:** 5 archivos

#### Precondiciones

- ✅ Tarea 1 completada (Value Objects funcionan)
- ✅ Tarea 2 completada (Enum funciona)
- Módulo Auth existe: `Modules/Auth/`
- Directorio `Modules/Auth/app/Data/` existe
- Directorio `Modules/Auth/tests/Unit/Data/` existe

#### Archivos a Crear

1. **`Modules/Auth/app/Data/LoginData.php`**
   ```php
   final readonly class LoginData extends Data
   {
       public function __construct(
           public EmailValueObject $email,
           public string $password,  // ← plano, NO PasswordValueObject
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

2. **`Modules/Auth/app/Data/RequestPasswordResetData.php`**
   ```php
   final readonly class RequestPasswordResetData extends Data
   {
       public function __construct(
           public EmailValueObject $email,
       ) {}
   
       public static function rules(): array
       {
           return [
               'email' => ['required', 'email', 'max:255'],
           ];
       }
   }
   ```

3. **`Modules/Auth/app/Data/ResetPasswordData.php`**
   ```php
   final readonly class ResetPasswordData extends Data
   {
       public function __construct(
           public EmailValueObject $email,
           public string $token,
           public string $password,  // ← plano, NO PasswordValueObject
       ) {}
   
       public static function rules(): array
       {
           return [
               'email' => ['required', 'email', 'max:255'],
               'token' => ['required', 'string', 'size:60'],
               'password' => ['required', 'string', 'min:8', 'confirmed'],
           ];
       }
   }
   ```

4. **`Modules/Auth/app/Data/AuthenticationResult.php`**
   ```php
   final readonly class AuthenticationResult extends Data
   {
       public function __construct(
           public bool $success,
           public ?User $user = null,
           public ?string $error = null,
       ) {}
   }
   ```

#### Tests a Crear

**`Modules/Auth/tests/Unit/Data/LoginDataTest.php`**

- ✅ Crea LoginData válido con EmailValueObject
- ✅ Valida reglas de email
- ✅ Valida reglas de password (min 8)
- ✅ Remember por defecto es false
- ✅ Serialización con `from()` y `toArray()`

**`Modules/Auth/tests/Unit/Data/RequestPasswordResetDataTest.php`**

- ✅ Crea RequestPasswordResetData válido
- ✅ Valida reglas de email
- ✅ Serialización

**`Modules/Auth/tests/Unit/Data/ResetPasswordDataTest.php`**

- ✅ Crea ResetPasswordData válido
- ✅ Valida reglas de email, token, password
- ✅ Valida longitud de token (60 caracteres)
- ✅ Serialización

**`Modules/Auth/tests/Unit/Data/AuthenticationResultTest.php`**

- ✅ Crea AuthenticationResult exitoso (success=true, user set)
- ✅ Crea AuthenticationResult fallido (success=false, error set)
- ✅ User es nullable
- ✅ Error es nullable

#### Validaciones Postcondición

```bash
./vendor/bin/sail bin pint Modules/Auth/app/Data/
./vendor/bin/sail composer run phpstan -- --level=6 Modules/Auth/app/Data/
./vendor/bin/sail test Modules/Auth/tests/Unit/Data/
```

#### Criterios de Completitud

- [ ] 4 Data Objects creados
- [ ] Reglas de validación implementadas
- [ ] Tests con cobertura 100%
- [ ] PHPStan level 6 sin errores

---

### 📦 Tarea 4: Contracts (Interfaces para Comunicación Inter-Módulos)

**Prioridad:** BAJA (opcional en Agente A)  
**Estimación:** 1 hora  
**Archivos a crear:** 0-3 archivos

#### Análisis de Necesidad

Según `auth-class-diagram.md`, el módulo Auth **NO expone contratos públicos a otros módulos** en el MVP. Las Actions se
usan internamente desde Custom Filament Pages.

**Decisión:** ⏭️ **SKIP esta tarea en Agente A**

Si en futuro se necesita comunicación inter-módulos (ej: módulo Order necesita validar permisos), se crearían:

- `Modules/Auth/Contracts/Commands/AuthenticateMerchantInterface.php`
- `Modules/Auth/Contracts/Queries/GetMerchantInterface.php`

Por ahora, las Actions serán **internas del módulo**.

---

### 📦 Tarea 5: Documentación y Validación Final

**Prioridad:** ALTA  
**Estimación:** 30 minutos

#### Checklist Final

**Código:**

- [ ] Todos los archivos tienen `declare(strict_types=1);`
- [ ] Todas las clases son `final`
- [ ] Todos los Value Objects son `readonly`
- [ ] Todos los Data Objects son `readonly`
- [ ] No hay `mixed`, `array` genéricos en signatures públicas

**Tests:**

- [ ] Cobertura 100% en Value Objects
- [ ] Cobertura 100% en Enums
- [ ] Cobertura 100% en Data Objects
- [ ] Todos los tests pasan

**Calidad:**

- [ ] PHPStan level 6 sin errores
- [ ] Pint sin cambios pendientes
- [ ] Rector ejecutado (si aplica)

**Documentación:**

- [ ] Docblocks en métodos públicos
- [ ] `@throws` documentados en Value Objects
- [ ] README.md del módulo actualizado (si existe)

#### Comandos de Validación Final

```bash
# 1. Formatear todo el código
./vendor/bin/sail bin pint --dirty

# 2. Análisis estático completo
./vendor/bin/sail composer run phpstan

# 3. Ejecutar todos los tests del proyecto
./vendor/bin/sail test

# 4. Ejecutar solo tests del Agente A
./vendor/bin/sail test tests/Unit/ValueObjects/
./vendor/bin/sail test tests/Unit/Enums/
./vendor/bin/sail test Modules/Auth/tests/Unit/Data/

# 5. Verificar cobertura
./vendor/bin/sail test --coverage --min=100 tests/Unit/ValueObjects/
./vendor/bin/sail test --coverage --min=100 tests/Unit/Enums/
./vendor/bin/sail test --coverage --min=100 Modules/Auth/tests/Unit/Data/

# 6. Rector (si hay reglas configuradas)
./vendor/bin/sail composer run rector --dry-run
```

#### Entregables del Agente A

Al finalizar, se habrán creado:

**Archivos de Código:** 11 archivos

- 3 Value Objects (`app/ValueObjects/`)
- 3 Casts (`app/Casts/`)
- 1 Enum (`app/Enums/`)
- 4 Data Objects (`Modules/Auth/app/Data/`)

**Tests:** 7 archivos

- 3 tests de Value Objects (`tests/Unit/ValueObjects/`)
- 1 test de Enum (`tests/Unit/Enums/`)
- 4 tests de Data Objects (`Modules/Auth/tests/Unit/Data/`) (opcional, si hay lógica compleja)

**Total:** ~18 archivos

---

## Orden de Ejecución Recomendado

```
Tarea 1 (Value Objects) → obligatoria primero
    ↓
Tarea 2 (Enum) → puede ir en paralelo con Tarea 1
    ↓
Tarea 3 (Data Objects) → depende de Tarea 1 y 2
    ↓
Tarea 4 (Contracts) → SKIP en MVP
    ↓
Tarea 5 (Validación Final) → obligatoria al final
```

**Tiempo estimado total:** 6-7 horas

---

## Riesgos y Mitigaciones

### Riesgo 1: Proyecto Laravel no existe o no está configurado

**Probabilidad:** Media  
**Impacto:** Alto (bloqueante)  
**Mitigación:** Validar precondiciones globales ANTES de comenzar Tarea 1

### Riesgo 2: Laravel Modules usa estructura diferente

**Probabilidad:** Media  
**Impacto:** Medio  
**Mitigación:** Verificar documentación de `nwidart/laravel-modules` y ajustar rutas

### Riesgo 3: Spatie Laravel Data no instalado o versión incompatible

**Probabilidad:** Baja  
**Impacto:** Alto (bloqueante para Tarea 3)  
**Mitigación:** Instalar antes de Tarea 3: `composer require spatie/laravel-data`

### Riesgo 4: Tests fallan por dependencias circulares

**Probabilidad:** Baja  
**Impacto:** Medio  
**Mitigación:** Value Objects NO deben depender entre sí, solo de clases de Laravel

### Riesgo 5: PHPStan level 6 demasiado estricto

**Probabilidad:** Media  
**Impacto:** Medio  
**Mitigación:** Comenzar con level 5, subir a 6 gradualmente. Agregar baseline si necesario.

---

## Próximos Pasos (Fuera del Agente A)

Después de completar el Agente A, otros agentes se encargarán de:

1. **Agente B - Modelos y Persistencia:**
    - Migración `create_users_table`
    - Migración `create_password_reset_tokens_table`
    - Model `User` con Casts
    - Model `PasswordResetToken`
    - Factories
    - Seeders

2. **Agente C - Lógica de Negocio:**
    - Actions (Authenticate, RequestPasswordReset, ResetPassword)
    - Repository (PasswordResetTokenRepository)
    - Exceptions custom

3. **Agente D - Infraestructura UI:**
    - Custom Filament Pages (Login, RequestPasswordReset, ResetPassword)
    - Configuración de AdminPanelServiceProvider
    - Commands (CreateOwnerCommand)
    - Jobs (CleanExpiredPasswordResetTokensJob)

4. **Agente E - Testing de Integración:**
    - Feature tests de flujos completos
    - Tests de integración con Filament
    - Smoke tests

---

## Notas Finales

- Este plan asume que el proyecto Laravel ya existe y está configurado
- Si no existe, crear un plan previo de "Setup Inicial del Proyecto"
- Cada tarea es independiente y puede ser implementada por un agente diferente (humano o IA)
- La validación después de cada tarea es OBLIGATORIA antes de continuar

---

**Estado:** 📋 Pendiente de ejecución  
**Aprobado por:** Alejandro Leone  
**Fecha de creación:** 2025-12-18
