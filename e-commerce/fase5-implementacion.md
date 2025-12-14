# Fase 5: Webhooks y Estados - Implementación Completa

## Fecha de implementación
14 de diciembre de 2025

## Resumen

Se ha completado exitosamente la implementación de la Fase 5 del MVP, que incluye:

1. **Sistema de Webhooks de Stripe**
2. **Gestión de Estados de Órdenes y Pagos**
3. **Procesamiento Idempotente de Eventos**
4. **Comando de Expiración de Órdenes**
5. **Tests Completos**

## Componentes Implementados

### 1. Controlador de Webhooks

**Archivo**: `app/Http/Controllers/Webhooks/StripeWebhookController.php`

- Maneja webhooks entrantes de Stripe
- Verifica firma HMAC para seguridad
- Delega procesamiento al gateway
- Maneja errores y responde correctamente

### 2. Configuración de Rutas

**Archivo**: `routes/web.php`

- Ruta POST `/webhooks/stripe` registrada
- Excluida del middleware CSRF en `bootstrap/app.php`

### 3. Eventos Soportados

El `StripeGateway` ya manejaba estos eventos:

- `checkout.session.completed` → Confirma pago exitoso
- `checkout.session.expired` → Marca sesión como expirada
- `payment_intent.payment_failed` → Marca pago como fallido

### 4. Modelo WebhookEvent

**Archivo**: `app/Models/WebhookEvent.php`

Registra eventos para idempotencia:
- `provider` (stripe)
- `event_id` (único de Stripe)
- `event_type`
- `payload` (JSON)
- `processed_at`

### 5. Estados de Orden

**Archivo**: `app/OrderStatus.php`

Enum con los siguientes estados:
- `Placed` → Orden creada
- `Paid` → Pago confirmado
- `PaymentFailed` → Pago fallido
- `Failed` → Fallo general
- `Expired` → Orden expirada por timeout

### 6. Estados de Pago

**Archivo**: `app/PaymentStatus.php`

Enum con estados:
- `Pending` → Pago iniciado
- `Processing` → En proceso (Stripe Checkout abierto)
- `Succeeded` → Pago exitoso
- `Failed` → Pago fallido

### 7. Comando de Expiración

**Archivo**: `app/Console/Commands/ExpirePendingOrders.php`

- Comando: `orders:expire-pending`
- Procesa órdenes antiguas (>30 min por defecto)
- Libera stock reservado
- Marca pagos como fallidos

### 8. Tests Implementados

#### `tests/Feature/StripeWebhookTest.php`
- ✅ Webhook procesa checkout.session.completed
- ✅ Webhook procesa checkout.session.expired
- ✅ Rechaza firma inválida
- ✅ Rechaza solicitudes sin firma
- ✅ Maneja idempotencia correctamente
- ✅ Ignora eventos no soportados

#### `tests/Feature/ExpirePendingOrdersCommandTest.php`
- ✅ Comando expira órdenes antiguas
- ✅ No afecta órdenes ya pagadas
- ✅ Libera stock de órdenes expiradas

#### `tests/Feature/WebhookIntegrationTest.php`
- ✅ Flujo completo: webhook confirma pago
- ✅ Flujo completo: webhook expira sesión
- ✅ Flujo completo: orden antigua expira automáticamente

### 9. Documentación

**Archivo**: `docs/WEBHOOKS.md`

Documentación completa que incluye:
- Configuración local con Stripe CLI
- Configuración en producción
- Eventos soportados y sus efectos
- Seguridad y verificación de firmas
- Idempotencia
- Monitoreo y troubleshooting
- Comandos útiles

### 10. Configuración

**Archivo**: `config/checkout.php`
- `expiration_minutes` → tiempo antes de expirar orden (default: 30)

**Variables de entorno** (`.env.example` actualizado):
```env
STRIPE_KEY=
STRIPE_SECRET=
STRIPE_WEBHOOK_SECRET=
CHECKOUT_EXPIRATION_MINUTES=30
```

## Flujos de Trabajo

### Flujo de Pago Exitoso

1. Usuario completa checkout en Stripe
2. Stripe envía webhook `checkout.session.completed`
3. Sistema verifica firma
4. Registra evento en `webhook_events` (idempotencia)
5. Actualiza pago a `succeeded`
6. Actualiza orden a `paid`
7. Confirma descuento de stock
8. Vacía carrito del cliente

### Flujo de Pago Fallido/Expirado

1. Sesión expira o pago falla
2. Stripe envía webhook correspondiente
3. Sistema verifica firma
4. Registra evento
5. Actualiza pago a `failed`
6. Actualiza orden a `payment_failed` o `expired`
7. Libera stock reservado

### Flujo de Expiración Automática

1. Comando `orders:expire-pending` se ejecuta (manualmente o programado)
2. Busca órdenes pendientes > 30 minutos
3. Para cada orden expirada:
   - Marca orden como `expired`
   - Marca pago como `failed`
   - Libera stock reservado
4. Registra en logs

## Seguridad

✅ Verificación de firma HMAC en todos los webhooks
✅ Ruta excluida de CSRF (necesario para webhooks externos)
✅ Validación de payload antes de procesar
✅ Logging completo de eventos y errores
✅ Idempotencia para evitar procesamiento duplicado

## Idempotencia Garantizada

El sistema garantiza que un evento webhook nunca se procese dos veces:

1. Antes de procesar, se verifica si `event_id` existe en `webhook_events`
2. Si existe, se retorna sin procesar
3. Si no existe, se procesa y luego se registra

Esto protege contra:
- Reintentos de Stripe
- Duplicación accidental
- Errores de red

## Monitoreo y Logs

Todos los eventos importantes se registran con contexto completo:
- IDs de orden, pago, evento
- Tipos de evento
- Montos y estado
- IPs y errores

Comando útil:
```bash
php artisan pail --filter="webhook"
```

## Testing

Se pueden ejecutar todos los tests con:

```bash
php artisan test --filter=Webhook
php artisan test --filter=ExpirePendingOrders
```

Cobertura de tests:
- ✅ Webhooks válidos e inválidos
- ✅ Verificación de firma
- ✅ Idempotencia
- ✅ Efectos secundarios (stock, estados)
- ✅ Comando de expiración
- ✅ Flujos de integración completos

## Próximos Pasos (Fase 6)

La Fase 5 está completa y lista. Los siguientes pasos sugeridos:

1. **Fase 6**: UI de Checkout y Carrito
   - Implementar páginas Volt para carrito
   - Integrar Stripe Elements
   - Mostrar resumen de orden

2. **Fase 7**: Scheduled Jobs
   - Programar `orders:expire-pending` en scheduler
   - Configurar cola para procesamiento asíncrono
   - Implementar notificaciones por email

3. **Testing en desarrollo**
   - Instalar Stripe CLI
   - Probar webhooks locales
   - Validar flujos completos

## Notas Técnicas

- Todos los archivos siguen las convenciones de Laravel 12
- Código formateado con Pint
- Strict types habilitado
- Type hints completos
- PHPDoc donde corresponde
- Tests con Pest 4
- Logging estructurado

## Estado de la Fase 5

✅ **COMPLETADA AL 100%**

Todos los componentes están implementados, testeados y documentados.
