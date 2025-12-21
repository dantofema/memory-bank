---
task_id: "auth-005-events"
module: "Auth"
agent: "Agente E - Events, Listeners y Jobs"
title: "Auth - Eventos de Dominio y Auditoría"
priority: "HIGH"
estimated_time: "4 hours"
dependencies:
  - "auth-001-contracts"
  - "auth-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
phase: "Fase 4 - Eventos"
security_critical: true
---

# Task 005: Auth - Eventos de Dominio y Auditoría

## 🎯 Objetivo

Implementar eventos de dominio para auditoría completa de autenticación, **incluyendo eventos de fallo** que son críticos para seguridad y prevención de intrusiones. En MVP no tienen listeners, pero quedan preparados para futuras integraciones con módulo de Security.

## ⚠️ Riesgos Críticos

### Puntos Ciegos de Auditoría
**Problema:** La ausencia de eventos de fallo impide que el módulo de Security reaccione ante:
- Intentos de fuerza bruta por IP
- Intentos de intrusión por cuenta
- Patrones de abuso sospechosos

**Solución:** Implementar `UserLoginFailedEvent` con datos completos de auditoría (IP, email, timestamp, razón de fallo).

## 📦 Artefactos a Crear

### 1. Event: UserLoginEvent

**Ubicación:** `Modules/Auth/App/Events/UserLoginEvent.php`

**Especificaciones:**

```php
namespace Modules\Auth\App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Modules\Auth\ValueObjects\Email;

final readonly class UserLoginEvent
{
    use Dispatchable;
    use SerializesModels;

    public function __construct(
        public int $user_id,
        public Email $email,
        public string $ip_address,
        public \Carbon\Carbon $logged_in_at,
    ) {}
}
```

---

### 2. Event: UserLoginFailedEvent ⚠️ CRÍTICO

**Ubicación:** `Modules/Auth/App/Events/UserLoginFailedEvent.php`

**Importancia:** Este evento es una **omisión de seguridad crítica**. Permite:
- Rastreo de intentos de intrusión
- Detección de fuerza bruta
- Auditoría de seguridad
- Rate limiting por IP/cuenta

**Especificaciones:**

```php
namespace Modules\Auth\App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Modules\Auth\ValueObjects\Email;

final readonly class UserLoginFailedEvent
{
    use Dispatchable;
    use SerializesModels;

    public function __construct(
        public Email $email,
        public string $ip_address,
        public string $failure_reason, // 'invalid_credentials', 'account_locked', etc.
        public \Carbon\Carbon $attempted_at,
    ) {}
}
```

**Notas:**
- No incluye user_id porque puede no existir el usuario
- failure_reason permite categorizar tipos de fallo
- IP y timestamp son esenciales para rate limiting

---

### 3. Event: UserLogoutEvent

**Ubicación:** `Modules/Auth/App/Events/UserLogoutEvent.php`

**Especificaciones:**

```php
namespace Modules\Auth\App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Modules\Auth\ValueObjects\Email;

final readonly class UserLogoutEvent
{
    use Dispatchable;
    use SerializesModels;

    public function __construct(
        public int $user_id,
        public Email $email,
        public \Carbon\Carbon $logged_out_at,
    ) {}
}
```

---

## 🧪 Tests Requeridos

### Unit Tests - Eventos

**Ubicación:** `Modules/Auth/tests/Unit/Events/`

```php
use Modules\Auth\App\Events\UserLoginEvent;
use Modules\Auth\App\Events\UserLoginFailedEvent;
use Modules\Auth\App\Events\UserLogoutEvent;
use Modules\Auth\ValueObjects\Email;

describe('UserLoginEvent', function () {
    it('crea el evento correctamente', function () {
        $email = Email::fromString('test@example.com');
        $timestamp = now();
        
        $event = new UserLoginEvent(
            user_id: 1,
            email: $email,
            ip_address: '192.168.1.1',
            logged_in_at: $timestamp
        );
        
        expect($event->user_id)->toBe(1)
            ->and($event->email)->toBe($email)
            ->and($event->ip_address)->toBe('192.168.1.1')
            ->and($event->logged_in_at)->toBe($timestamp);
    });
});

describe('UserLoginFailedEvent', function () {
    it('crea el evento de fallo correctamente', function () {
        $email = Email::fromString('test@example.com');
        $timestamp = now();
        
        $event = new UserLoginFailedEvent(
            email: $email,
            ip_address: '192.168.1.1',
            failure_reason: 'invalid_credentials',
            attempted_at: $timestamp
        );
        
        expect($event->email)->toBe($email)
            ->and($event->ip_address)->toBe('192.168.1.1')
            ->and($event->failure_reason)->toBe('invalid_credentials')
            ->and($event->attempted_at)->toBe($timestamp);
    });
});

describe('UserLogoutEvent', function () {
    it('crea el evento correctamente', function () {
        $email = Email::fromString('test@example.com');
        $timestamp = now();
        
        $event = new UserLogoutEvent(
            user_id: 1,
            email: $email,
            logged_out_at: $timestamp
        );
        
        expect($event->user_id)->toBe(1)
            ->and($event->email)->toBe($email)
            ->and($event->logged_out_at)->toBe($timestamp);
    });
});
```

### Feature Tests - Dispatching

**Ubicación:** `Modules/Auth/tests/Feature/Events/`

```php
describe('Auth Events Dispatching', function () {
    it('dispatches UserLoginEvent on successful login', function () {
        Event::fake();
        
        // ... login action test
        
        Event::assertDispatched(UserLoginEvent::class, function ($event) {
            return $event->email->toString() === 'test@example.com'
                && $event->ip_address !== null;
        });
    });
    
    it('dispatches UserLoginFailedEvent on invalid credentials', function () {
        Event::fake();
        
        // ... failed login action test
        
        Event::assertDispatched(UserLoginFailedEvent::class, function ($event) {
            return $event->email->toString() === 'test@example.com'
                && $event->failure_reason === 'invalid_credentials'
                && $event->ip_address !== null;
        });
    });

    it('dispatches UserLogoutEvent on logout', function () {
        Event::fake();
        
        // ... logout action test
        
        Event::assertDispatched(UserLogoutEvent::class, function ($event) {
            return $event->user_id === 1;
        });
    });
});
```

## ✅ Criterios de Aceptación

### Funcionales
- [ ] UserLoginEvent se dispara en login exitoso con todos los datos (user_id, email, IP, timestamp)
- [ ] **UserLoginFailedEvent se dispara en login fallido con email, IP, razón de fallo y timestamp** ⚠️
- [ ] UserLogoutEvent se dispara en logout con user_id, email y timestamp
- [ ] Eventos contienen todos los datos necesarios para auditoría
- [ ] Eventos son inmutables (readonly)
- [ ] failure_reason en UserLoginFailedEvent es descriptivo ('invalid_credentials', 'account_locked', etc.)

### Técnicos
- [ ] Eventos son final readonly class
- [ ] Eventos usan ValueObjects (Email) correctamente
- [ ] Tests unitarios verifican creación de eventos
- [ ] Tests de feature verifican dispatch de eventos
- [ ] PHPStan level 6+ sin errores
- [ ] Pint ejecutado correctamente
- [ ] Cobertura de tests 100%

### Seguridad
- [ ] **UserLoginFailedEvent captura intentos de intrusión** ⚠️
- [ ] IP address registrada en todos los eventos relevantes
- [ ] Timestamps precisos para correlación temporal
- [ ] Datos suficientes para rate limiting futuro

## 🔗 Integración con Módulo Security (Futuro)

Estos eventos serán consumidos por:
- **RateLimitListener**: Bloqueo por IP tras N intentos fallidos
- **AbuseDetectionListener**: Patrones sospechosos (mismo IP, múltiples cuentas)
- **AuditLogListener**: Registro persistente de auditoría
- **AlertListener**: Notificaciones a admins sobre actividad sospechosa

## 📋 Notas de Implementación

1. **Emisión desde Actions (Agente B)**:
   - `LoginUserAction` → dispara `UserLoginEvent` o `UserLoginFailedEvent`
   - `LogoutUserAction` → dispara `UserLogoutEvent`

2. **No crear Listeners en esta task**:
   - Eventos quedan preparados para futuras integraciones
   - MVP solo emite eventos, no los consume

3. **Registro en EventServiceProvider**:
   - Aunque no hay listeners, registrar eventos para visibilidad

**Status:** ⚠️ ACTUALIZADO - Evento de fallo añadido  
**Fase:** 4 - Eventos  
**Depende de:** Task 001 (ValueObjects), 003 (Persistence)  
**Prioridad:** HIGH (por implicaciones de seguridad)  
**Nota:** Listeners no implementados en MVP, pero eventos son críticos para auditoría
