---
task_id: "auth-005-events"
module: "Auth"
agent: "Agente E - Events, Listeners y Jobs"
title: "Auth - Eventos de Dominio y Auditoría"
priority: "MEDIUM"
estimated_time: "3 hours"
dependencies:
  - "auth-001-contracts"
  - "auth-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-e-events.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
phase: "Fase 4 - Eventos"
---

# Task 005: Auth - Eventos de Dominio y Auditoría

## 🎯 Objetivo

Implementar eventos de dominio para auditoría de autenticación. En MVP no tienen listeners, pero quedan preparados para futuras integraciones.

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

### 2. Event: UserLogoutEvent

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

```php
use Modules\Auth\App\Events\UserLoginEvent;
use Modules\Auth\App\Events\UserLogoutEvent;

describe('Auth Events', function () {
    it('dispatches UserLoginEvent on login', function () {
        Event::fake();
        // ... login test
        Event::assertDispatched(UserLoginEvent::class);
    });

    it('dispatches UserLogoutEvent on logout', function () {
        Event::fake();
        // ... logout test
        Event::assertDispatched(UserLogoutEvent::class);
    });
});
```

## ✅ Criterios de Aceptación

### Funcionales
- [ ] UserLoginEvent se dispara en login exitoso
- [ ] UserLogoutEvent se dispara en logout
- [ ] Eventos contienen todos los datos necesarios
- [ ] Eventos son inmutables (readonly)

### Técnicos
- [ ] Eventos son final readonly class
- [ ] Tests verifican dispatch de eventos
- [ ] PHPStan level 6+ sin errores

**Status:** ✅ Ready to Implement  
**Fase:** 4 - Eventos  
**Depende de:** Task 001, 003  
**Nota:** Listeners no implementados en MVP
