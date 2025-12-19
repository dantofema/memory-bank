---
task_id: "agente-e-task-03"
module: "Auth"
agent: "Agente E"
title: "Job de Limpieza de Tokens Expirados"
priority: "medium"
estimated_time: "1h"
dependencies:
  - "Agente C: PasswordResetTokenRepository implementado"
status: "pending"
---

# Task: Job de Limpieza de Tokens Expirados

## Contexto

Crear un Job asíncrono que elimine tokens de password reset expirados de forma programada. Este Job se ejecuta diariamente mediante Laravel Scheduler y mantiene la base de datos limpia de tokens antiguos.

**Objetivo**: Limpieza automática de tokens expirados (> 60 minutos) sin intervención manual.

---

## Objetivos

1. ✅ Crear Job `CleanExpiredPasswordResetTokensJob`
2. ✅ Implementar lógica de limpieza delegando a Repository
3. ✅ Configurar ejecución programada en Scheduler
4. ✅ Tests del Job con cola fake
5. ✅ Validar PHPStan level 6

---

## Job a Crear

### CleanExpiredPasswordResetTokensJob

**Ubicación**: `Modules/Auth/app/Jobs/CleanExpiredPasswordResetTokensJob.php`

**Descripción**: Elimina tokens de password reset expirados (creados hace más de 60 minutos).

**Código**:

```php
<?php

declare(strict_types=1);

namespace Modules\Auth\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Modules\Auth\Repositories\PasswordResetTokenRepository;

/**
 * Job que elimina tokens de password reset expirados.
 * 
 * Se ejecuta diariamente mediante Laravel Scheduler.
 */
final class CleanExpiredPasswordResetTokensJob implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    /**
     * Timeout en segundos.
     */
    public int $timeout = 60;

    /**
     * Número de reintentos.
     */
    public int $tries = 3;

    /**
     * Backoff en segundos entre reintentos.
     */
    public array $backoff = [10, 30, 60];

    public function __construct()
    {
        // Sin parámetros: no depende de datos específicos
    }

    public function handle(PasswordResetTokenRepository $repository): void
    {
        Log::info('Starting cleanup of expired password reset tokens');

        $deletedCount = $repository->deleteExpired();

        // Alerta si se elimina un número anormalmente alto de tokens
        if ($deletedCount > 1000) {
            Log::warning('High number of expired tokens deleted', [
                'deleted_count' => $deletedCount,
                'context' => 'security_audit',
            ]);
        }

        Log::info('Finished cleanup of expired password reset tokens', [
            'deleted_count' => $deletedCount,
        ]);
    }

    /**
     * Maneja un fallo del job.
     */
    public function failed(\Throwable $exception): void
    {
        Log::error('Failed to clean expired password reset tokens', [
            'error' => $exception->getMessage(),
            'trace' => $exception->getTraceAsString(),
        ]);
    }
}
```

**Características**:
- ✅ Implementa `ShouldQueue` (asíncrono)
- ✅ Traits obligatorios de Laravel Queues
- ✅ Timeout y reintentos configurados con backoff exponencial
- ✅ Delega lógica a Repository
- ✅ Logging de inicio, fin y errores
- ✅ Alerta si se eliminan > 1000 tokens (posible ataque)
- ✅ Método `failed()` para manejar fallos
- ✅ Timeout configurable desde config/auth.php

---

## Actualización del Repository

### Método `deleteExpired()` en PasswordResetTokenRepository

**Ubicación**: `Modules/Auth/app/Repositories/PasswordResetTokenRepository.php`

**Agregar método** (si no existe):

```php
/**
 * Elimina todos los tokens expirados (creados hace más de X minutos).
 * 
 * El timeout se configura en config/auth.php con la clave 'password_timeout'.
 * Por defecto: 60 minutos.
 * 
 * @return int Cantidad de tokens eliminados
 */
public function deleteExpired(): int
{
    $timeoutMinutes = config('auth.password_timeout', 60);
    
    return PasswordResetToken::where(
        'created_at',
        '<',
        now()->subMinutes($timeoutMinutes)
    )->delete();
}
```

**Nota**: Si este método ya existe del Agente C, no hace falta crearlo nuevamente.

---

## Configuración del Scheduler

### Programar Ejecución Diaria

**Ubicación**: `app/Console/Kernel.php` o `routes/console.php` (Laravel 11+)

#### Opción 1: Laravel 11+ con `routes/console.php`

```php
<?php

use Illuminate\Support\Facades\Schedule;
use Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob;

Schedule::job(CleanExpiredPasswordResetTokensJob::class)
    ->daily()
    ->at('02:00')
    ->name('auth:clean-expired-tokens')
    ->onOneServer()
    ->withoutOverlapping();
```

#### Opción 2: Laravel 10 con `app/Console/Kernel.php`

```php
<?php

declare(strict_types=1);

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;
use Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob;

final class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $schedule->job(CleanExpiredPasswordResetTokensJob::class)
            ->daily()
            ->at('02:00')
            ->name('auth:clean-expired-tokens')
            ->onOneServer()
            ->withoutOverlapping();
    }

    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}
```

**Configuración**:
- ✅ Ejecución diaria a las 02:00 AM
- ✅ `onOneServer()`: Solo se ejecuta en un servidor (importante en clusters)
- ✅ `withoutOverlapping()`: No se ejecuta si ya hay una instancia corriendo
- ✅ `name()`: Identificador para logging y debugging

---

## Tests del Job

### Test: CleanExpiredPasswordResetTokensJobTest

**Ubicación**: `Modules/Auth/tests/Feature/Jobs/CleanExpiredPasswordResetTokensJobTest.php`

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Queue;
use Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob;
use Modules\Auth\Models\PasswordResetToken;
use Modules\Auth\Repositories\PasswordResetTokenRepository;

beforeEach(function () {
    $this->repository = new PasswordResetTokenRepository();
});

it('elimina tokens expirados correctamente', function () {
    // Crear tokens: 2 expirados, 1 válido
    $expiredToken1 = PasswordResetToken::factory()->create([
        'created_at' => now()->subHours(2), // Expirado
    ]);

    $expiredToken2 = PasswordResetToken::factory()->create([
        'created_at' => now()->subMinutes(90), // Expirado
    ]);

    $validToken = PasswordResetToken::factory()->create([
        'created_at' => now()->subMinutes(30), // Válido
    ]);

    expect(PasswordResetToken::count())->toBe(3);

    // Ejecutar Job
    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);

    // Verificar que solo se eliminaron los expirados
    expect(PasswordResetToken::count())->toBe(1)
        ->and(PasswordResetToken::where('email', $validToken->email->toString())->exists())->toBeTrue()
        ->and(PasswordResetToken::where('email', $expiredToken1->email->toString())->exists())->toBeFalse()
        ->and(PasswordResetToken::where('email', $expiredToken2->email->toString())->exists())->toBeFalse();
});

it('registra en logs el inicio y fin de la limpieza', function () {
    Log::spy();

    PasswordResetToken::factory()->create([
        'created_at' => now()->subHours(2),
    ]);

    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Starting cleanup of expired password reset tokens');

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Finished cleanup of expired password reset tokens', \Mockery::on(function ($context) {
            return isset($context['deleted_count']) && $context['deleted_count'] === 1;
        }));
});

it('no elimina tokens válidos', function () {
    $validToken = PasswordResetToken::factory()->create([
        'created_at' => now()->subMinutes(30),
    ]);

    expect(PasswordResetToken::count())->toBe(1);

    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);

    expect(PasswordResetToken::count())->toBe(1)
        ->and(PasswordResetToken::where('email', $validToken->email->toString())->exists())->toBeTrue();
});

it('maneja correctamente cuando no hay tokens expirados', function () {
    Log::spy();

    expect(PasswordResetToken::count())->toBe(0);

    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);

    expect(PasswordResetToken::count())->toBe(0);

    Log::shouldHaveReceived('info')
        ->once()
        ->with('Finished cleanup of expired password reset tokens', \Mockery::on(function ($context) {
            return $context['deleted_count'] === 0;
        }));
});

it('se puede despachar a la cola correctamente', function () {
    Queue::fake();

    CleanExpiredPasswordResetTokensJob::dispatch();

    Queue::assertPushed(CleanExpiredPasswordResetTokensJob::class);
});

it('registra warning cuando se eliminan más de 1000 tokens', function () {
    Log::spy();

    // Crear 1500 tokens expirados
    PasswordResetToken::factory()->count(1500)->create([
        'created_at' => now()->subHours(2),
    ]);

    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);

    // Verificar warning
    Log::shouldHaveReceived('warning')
        ->once()
        ->with('High number of expired tokens deleted', \Mockery::on(function ($context) {
            return $context['deleted_count'] === 1500
                && $context['context'] === 'security_audit';
        }));
});

it('elimina tokens eficientemente con 10k registros', function () {
    // Test de performance
    PasswordResetToken::factory()->count(10000)->create([
        'created_at' => now()->subHours(2),
    ]);

    expect(PasswordResetToken::count())->toBe(10000);

    $startTime = microtime(true);
    
    $job = new CleanExpiredPasswordResetTokensJob();
    $job->handle($this->repository);
    
    $executionTime = microtime(true) - $startTime;

    expect(PasswordResetToken::count())->toBe(0)
        ->and($executionTime)->toBeLessThan(5); // Debe completar en menos de 5 segundos
});

it('registra errores si falla', function () {
    Log::spy();

    $job = new CleanExpiredPasswordResetTokensJob();
    $exception = new \Exception('Test error');

    $job->failed($exception);

    Log::shouldHaveReceived('error')
        ->once()
        ->with('Failed to clean expired password reset tokens', \Mockery::on(function ($context) {
            return $context['error'] === 'Test error';
        }));
});
```

---

## Comandos Útiles

### Ejecutar Job Manualmente

```bash
# Despachar a la cola
./vendor/bin/sail artisan tinker
>>> dispatch(new \Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob());

# O ejecutar síncronamente
>>> (new \Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob())->handle(app(\Modules\Auth\Repositories\PasswordResetTokenRepository::class));
```

### Ejecutar Worker de Cola

```bash
./vendor/bin/sail artisan queue:work --once
```

### Ver Tareas Programadas

```bash
./vendor/bin/sail artisan schedule:list
```

**Salida esperada**:
```
0 2 * * * Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob ... Next Due: ...
```

### Ejecutar Scheduler Manualmente (para testing)

```bash
./vendor/bin/sail artisan schedule:run
```

---

## Validación de Calidad

### Checklist Obligatorio

- [ ] **PHPStan level 6** sin errores
  ```bash
  ./vendor/bin/sail composer run phpstan
  ```

- [ ] **Pint** ejecutado
  ```bash
  ./vendor/bin/sail bin pint --dirty
  ```

- [ ] **Tests del Job verdes**
  ```bash
  ./vendor/bin/sail test Modules/Auth/Tests/Feature/Jobs
  ```

- [ ] **Job despachable a la cola**
  ```bash
  ./vendor/bin/sail artisan tinker
  >>> dispatch(new \Modules\Auth\Jobs\CleanExpiredPasswordResetTokensJob());
  ```

- [ ] **Scheduler configurado correctamente**
  ```bash
  ./vendor/bin/sail artisan schedule:list | grep auth:clean-expired-tokens
  ```

---

## Estructura de Archivos Final

```
Modules/Auth/app/Jobs/
└── CleanExpiredPasswordResetTokensJob.php ✅

Modules/Auth/tests/Feature/Jobs/
└── CleanExpiredPasswordResetTokensJobTest.php ✅

routes/console.php (o app/Console/Kernel.php)
└── Schedule configurado ✅
```

---

## Criterios de Aceptación

La tarea está completa cuando:

1. ✅ Job `CleanExpiredPasswordResetTokensJob` creado
2. ✅ Implementa `ShouldQueue` con traits obligatorios
3. ✅ Delega lógica a `PasswordResetTokenRepository`
4. ✅ Logging de inicio, fin y errores
5. ✅ Scheduler configurado (daily a las 02:00)
6. ✅ Tests del Job verdes (100% cobertura)
7. ✅ PHPStan level 6 sin errores
8. ✅ `schedule:list` muestra el Job programado

---

## Notas Importantes

### 🎯 Job solo limpia, no decide

- No contiene lógica de negocio
- No valida nada
- Solo delega a Repository
- Repository decide qué es "expirado"

### ⏰ Ejecución en Producción

En producción, el Scheduler requiere un cron job:

```bash
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

### 🔄 Idempotencia

El Job es **idempotente**: ejecutarlo múltiples veces con los mismos datos produce el mismo resultado (no hay efectos secundarios indeseados).

### 📊 Monitoring

El campo `deleted_count` en logs permite:
- Monitorear cuántos tokens se limpian diariamente
- Detectar patrones (muchos tokens expirados = posible ataque)
- Alertas automáticas si el número supera 1000 tokens

**Configuración de alertas recomendada**:
```bash
# En sistema de monitoreo (CloudWatch, ELK, etc.)
# Crear alerta si deleted_count > 1000 en 24h
# Esto puede indicar:
# - Ataque de fuerza bruta
# - Bot generando tokens masivamente
# - Problema en la aplicación
```

**Configuración del timeout**:
```php
// config/auth.php
'password_timeout' => env('AUTH_PASSWORD_TIMEOUT', 60), // minutos
```

---

## Referencias

- **Agente E (metodología)**: `laravel/agents/agente-e.md`
- **Agente C**: PasswordResetTokenRepository
- **Laravel Queues**: https://laravel.com/docs/queues
- **Laravel Scheduler**: https://laravel.com/docs/scheduling

---

**Última actualización**: 2025-12-18  
**Autor**: Alejandro Leone
