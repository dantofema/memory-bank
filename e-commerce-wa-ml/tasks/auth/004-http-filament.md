---
task_id: "auth-004-http-filament"
module: "Auth"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Auth - Integración con Filament y Tests Feature"
priority: "HIGH"
estimated_time: "6 hours"
dependencies:
  - "auth-001-contracts"
  - "auth-002-actions"
  - "auth-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
phase: "Fase 3 - Presentación"
---

# Task 004: Auth - Integración con Filament y Tests Feature

## 🎯 Objetivo

Integrar el módulo Auth con Filament para proporcionar autenticación segura en el backoffice. Implementar rate limiting, protección CSRF, y tests feature completos.

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Login funciona con credenciales válidas
- [ ] Login falla con credenciales inválidas
- [ ] Remember me genera token correctamente
- [ ] Rate limiting bloquea después de 5 intentos
- [ ] Logout invalida sesión y revoca token
- [ ] Dashboard solo accesible con autenticación
- [ ] CSRF protection funciona en formularios

### Técnicos
- [ ] Tests con Pest 4
- [ ] Cobertura 100% de flujos principales
- [ ] PHPStan level 6+ sin errores
- [ ] Session security configurada
- [ ] Rate limiting configurable vía env
- [ ] Middleware de autenticación expuesto como contrato público
- [ ] Rate limiting (5 intentos/minuto) implementado estrictamente

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Presentación  
**Depende de:** Task 001, 002, 003

---

## 📋 Implementación Detallada

### 1. Middleware de Autenticación como Contrato Público

**Ubicación:** `Modules/Auth/Http/Middleware/`

Crear middleware que será consumido por otros módulos de backoffice:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Modules\Auth\Contracts\Queries\GetAuthenticatedUserInterface;
use Symfony\Component\HttpFoundation\Response;

final readonly class AuthenticateBackoffice
{
    public function __construct(
        private GetAuthenticatedUserInterface $getAuthenticatedUser,
    ) {}

    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->session()->has('auth_user_id')) {
            return redirect()->route('auth.login')
                ->with('error', 'Debe iniciar sesión para acceder');
        }

        try {
            $userId = $request->session()->get('auth_user_id');
            $user = $this->getAuthenticatedUser->execute($userId);
            
            // Compartir usuario autenticado en el request
            $request->merge(['authenticated_user' => $user]);
            
            return $next($request);
            
        } catch (\Exception $e) {
            $request->session()->flush();
            
            return redirect()->route('auth.login')
                ->with('error', 'Sesión inválida o expirada');
        }
    }
}
```

**Registro del middleware como alias público:**

En `Modules/Auth/Providers/AuthServiceProvider.php`:

```php
protected array $middlewareAliases = [
    'auth.backoffice' => \Modules\Auth\Http\Middleware\AuthenticateBackoffice::class,
];
```

**Documentación para consumo por otros módulos:**

Crear `Modules/Auth/docs/middleware.md`:

```markdown
# Middleware de Autenticación Auth Module

## Uso en otros módulos

### En rutas web:
```php
Route::middleware(['auth.backoffice'])->group(function () {
    // Rutas protegidas
});
```

### En controllers:
```php
public function __construct()
{
    $this->middleware('auth.backoffice');
}
```

### Obtener usuario autenticado:
```php
$user = $request->get('authenticated_user');
```
```

---

### 2. Rate Limiting Estricto (Seguridad Crítica)

**⚠️ RIESGO CRÍTICO:** Si el rate limiting no se implementa correctamente, el sistema queda expuesto a ataques de fuerza bruta.

#### 2.1 Configuración de Rate Limiting

**Archivo:** `config/auth.php` (extender)

```php
return [
    'rate_limiting' => [
        'login' => [
            'max_attempts' => env('AUTH_MAX_LOGIN_ATTEMPTS', 5),
            'decay_minutes' => env('AUTH_LOGIN_DECAY_MINUTES', 1),
            'lockout_duration' => env('AUTH_LOCKOUT_DURATION', 15), // minutos
        ],
    ],
];
```

**.env:**
```env
AUTH_MAX_LOGIN_ATTEMPTS=5
AUTH_LOGIN_DECAY_MINUTES=1
AUTH_LOCKOUT_DURATION=15
```

#### 2.2 Middleware de Rate Limiting para Login

**Archivo:** `Modules/Auth/Http/Middleware/ThrottleLogin.php`

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Http\Middleware;

use Closure;
use Illuminate\Cache\RateLimiter;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final readonly class ThrottleLogin
{
    public function __construct(
        private RateLimiter $limiter,
    ) {}

    public function handle(Request $request, Closure $next): Response
    {
        $key = $this->throttleKey($request);
        $maxAttempts = config('auth.rate_limiting.login.max_attempts', 5);
        $decayMinutes = config('auth.rate_limiting.login.decay_minutes', 1);

        if ($this->limiter->tooManyAttempts($key, $maxAttempts)) {
            $seconds = $this->limiter->availableIn($key);
            
            return response()->json([
                'message' => 'Demasiados intentos de inicio de sesión',
                'retry_after' => $seconds,
            ], 429);
        }

        $this->limiter->hit($key, $decayMinutes * 60);

        $response = $next($request);

        // Si el login fue exitoso, limpiar el contador
        if ($response->isSuccessful()) {
            $this->limiter->clear($key);
        }

        return $response;
    }

    private function throttleKey(Request $request): string
    {
        return 'login:' . $request->input('email') . ':' . $request->ip();
    }
}
```

#### 2.3 Aplicar Rate Limiting en Rutas

**Archivo:** `Modules/Auth/Routes/web.php`

```php
<?php

use Illuminate\Support\Facades\Route;
use Modules\Auth\Http\Controllers\Web\LoginController;

Route::prefix('auth')
    ->name('auth.')
    ->group(function (): void {
        // Rutas públicas con rate limiting estricto
        Route::middleware(['web', 'throttle.login'])
            ->group(function (): void {
                Route::post('login', [LoginController::class, 'store'])
                    ->name('login.store');
            });

        Route::middleware(['web'])
            ->group(function (): void {
                Route::get('login', [LoginController::class, 'create'])
                    ->name('login');
            });

        // Rutas protegidas
        Route::middleware(['web', 'auth.backoffice'])
            ->group(function (): void {
                Route::post('logout', [LoginController::class, 'destroy'])
                    ->name('logout');
            });
    });
```

#### 2.4 Tests de Rate Limiting (Obligatorios)

**Archivo:** `Modules/Auth/tests/Feature/Http/RateLimitingTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\RateLimiter;

beforeEach(function (): void {
    RateLimiter::clear('login:test@example.com:127.0.0.1');
});

test('permite 5 intentos de login fallidos antes de bloquear', function (): void {
    // Arrange
    $credentials = ['email' => 'test@example.com', 'password' => 'wrong'];

    // Act & Assert - Primeros 5 intentos
    for ($i = 0; $i < 5; $i++) {
        $response = $this->post(route('auth.login.store'), $credentials);
        $response->assertStatus(302); // Redirect con error
    }

    // 6to intento debe ser bloqueado
    $response = $this->post(route('auth.login.store'), $credentials);
    $response->assertStatus(429); // Too Many Requests
});

test('limpia el rate limiter después de login exitoso', function (): void {
    // Arrange
    $user = User::factory()->create([
        'email' => 'success@example.com',
        'password' => bcrypt('correct-password'),
    ]);

    // Act - 3 intentos fallidos
    for ($i = 0; $i < 3; $i++) {
        $this->post(route('auth.login.store'), [
            'email' => 'success@example.com',
            'password' => 'wrong',
        ]);
    }

    // Login exitoso
    $response = $this->post(route('auth.login.store'), [
        'email' => 'success@example.com',
        'password' => 'correct-password',
    ]);

    // Assert
    $response->assertRedirect();
    
    // Verificar que el rate limiter se limpió
    $key = 'login:success@example.com:127.0.0.1';
    expect(RateLimiter::attempts($key))->toBe(0);
});

test('rate limiter usa combinación de email e IP', function (): void {
    // Arrange
    $credentials = ['email' => 'test@example.com', 'password' => 'wrong'];

    // Act - 5 intentos desde IP 1
    for ($i = 0; $i < 5; $i++) {
        $this->post(route('auth.login.store'), $credentials);
    }

    // Simular request desde IP diferente
    $this->withServerVariables(['REMOTE_ADDR' => '192.168.1.1'])
        ->post(route('auth.login.store'), $credentials)
        ->assertStatus(302); // No bloqueado, diferente IP
});
```

---

### 3. Tests Feature Completos

#### 3.1 Login Tests

**Archivo:** `Modules/Auth/tests/Feature/Http/Web/LoginTest.php`

```php
<?php

declare(strict_types=1);

test('usuario puede hacer login con credenciales válidas', function (): void {
    // Arrange
    $user = User::factory()->create([
        'email' => 'test@example.com',
        'password' => bcrypt('password123'),
    ]);

    // Act
    $response = $this->post(route('auth.login.store'), [
        'email' => 'test@example.com',
        'password' => 'password123',
    ]);

    // Assert
    $response->assertRedirect(route('dashboard'));
    $this->assertAuthenticatedAs($user);
});

test('login falla con credenciales inválidas', function (): void {
    // Act
    $response = $this->post(route('auth.login.store'), [
        'email' => 'wrong@example.com',
        'password' => 'wrong-password',
    ]);

    // Assert
    $response->assertRedirect();
    $response->assertSessionHasErrors(['email']);
    $this->assertGuest();
});

test('remember me genera token correctamente', function (): void {
    // Arrange
    $user = User::factory()->create([
        'email' => 'test@example.com',
        'password' => bcrypt('password123'),
    ]);

    // Act
    $response = $this->post(route('auth.login.store'), [
        'email' => 'test@example.com',
        'password' => 'password123',
        'remember' => true,
    ]);

    // Assert
    $response->assertRedirect();
    $this->assertNotNull($user->fresh()->remember_token);
});
```

#### 3.2 Logout Tests

```php
test('usuario autenticado puede hacer logout', function (): void {
    // Arrange
    $user = User::factory()->create();
    $this->actingAs($user);

    // Act
    $response = $this->post(route('auth.logout'));

    // Assert
    $response->assertRedirect(route('auth.login'));
    $this->assertGuest();
});
```

#### 3.3 CSRF Protection Tests

```php
test('formulario de login requiere token CSRF', function (): void {
    // Act
    $response = $this->withoutMiddleware(\Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class)
        ->post(route('auth.login.store'), [
            'email' => 'test@example.com',
            'password' => 'password',
        ]);

    // Assert - Sin CSRF middleware, la request procede
    $response->assertStatus(302);
    
    // Con CSRF middleware (comportamiento por defecto)
    $response = $this->post(route('auth.login.store'), [
        'email' => 'test@example.com',
        'password' => 'password',
    ]);
    
    // Sin token CSRF válido debe fallar
    $response->assertStatus(419);
});
```

---

## 🔒 Checklist de Seguridad

- [ ] Rate limiting configurado (5 intentos/minuto)
- [ ] Rate limiting usa combinación email + IP
- [ ] Rate limiter se limpia en login exitoso
- [ ] CSRF protection activa en todas las rutas POST
- [ ] Session timeout configurado (default 2 horas)
- [ ] Remember token usa hash seguro
- [ ] Passwords hasheados con bcrypt
- [ ] No se exponen detalles de error en producción
- [ ] Logs de intentos fallidos activados
- [ ] Middleware de autenticación expuesto como contrato

---

## 🧪 Tests Obligatorios

### Feature Tests
- [ ] Login exitoso con credenciales válidas
- [ ] Login fallido con credenciales inválidas
- [ ] Remember me funciona correctamente
- [ ] Logout invalida sesión
- [ ] Rate limiting bloquea después de 5 intentos
- [ ] Rate limiting se limpia en login exitoso
- [ ] CSRF protection funciona
- [ ] Middleware auth.backoffice redirige si no autenticado

### Smoke Tests
- [ ] Página de login carga sin errores
- [ ] Página de dashboard carga para usuario autenticado
- [ ] Dashboard redirige a login para usuarios no autenticados

---

## 📦 Entregables

1. **Middleware:**
   - `AuthenticateBackoffice.php` (contrato público)
   - `ThrottleLogin.php` (rate limiting)

2. **Controllers:**
   - `LoginController.php` (web)

3. **Rutas:**
   - `Routes/web.php` con rate limiting aplicado

4. **Tests:**
   - `LoginTest.php`
   - `RateLimitingTest.php`
   - `MiddlewareTest.php`
   - `SmokeTest.php`

5. **Configuración:**
   - Variables de entorno para rate limiting
   - Documentación de middleware en `docs/middleware.md`

---

## ⚠️ Riesgos Críticos Mitigados

### 1. Vulnerabilidad de Fuerza Bruta
**Mitigación:** Rate limiting estricto (5 intentos/minuto) implementado en capa de middleware, no bypasseable.

### 2. Session Hijacking
**Mitigación:** Session timeout, regeneración de ID en login, invalidación en logout.

### 3. CSRF Attacks
**Mitigación:** Token CSRF obligatorio en todas las rutas POST.

### 4. Falta de Contrato para Otros Módulos
**Mitigación:** Middleware expuesto como alias público `auth.backoffice` con documentación clara.
