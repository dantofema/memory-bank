---
name: "E-Commerce WhatsApp + Mercado Libre"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-16"
purpose: "Single-tenant e-commerce platform for entrepreneurs with WhatsApp integration"
tools:
  - sail
  - pest
  - phpstan
  - rector
  - pint
  - filament
  - livewire
  - tailwind
phpstan_level: 6
context:
  project_type: "Laravel 12 with Filament v4, Livewire v3, Pest v4"
  team_size: 1
  priority: "MVP - speed and simplicity"
  quality_gates: "PHPStan level 6+, Pint, Rector"
  architecture: "Monolith with Laravel Modules"
---

# Project Definition

## 1. Arquitectura y enfoque

**Single-tenant | Monolito | Un merchant por instancia**

Esta aplicación está diseñada como un sistema monolítico para un único comercio (emprendedor). No es un marketplace ni
multi-tenant. Cada instancia pertenece a un solo merchant, lo que simplifica la arquitectura, el modelo de datos y la
lógica de negocio.

**Principio Lean**: priorizar funcionalidad mínima viable, evitar sobre-diseño, iterar en base a feedback real.

---

## 2. Problema y solución

### Problema

Muchos emprendedores no cuentan con herramientas digitales para gestionar pedidos o les resulta complejo acceder a
ellas. El proceso actual suele ser manual, desordenado y dependiente exclusivamente de WhatsApp, generando errores,
pérdida de oportunidades y mala experiencia para el cliente.

### Solución

Plataforma web simple que permite:

- Catálogo de productos público con carrito de compras.
- Generación de pedidos sin necesidad de cuenta.
- Envío automático de pedido vía WhatsApp al merchant.
- Backoffice completo para gestión de productos, pedidos y reportes.
- Integración de pago con Mercado Pago o efectivo/transferencia.

### Propuesta de valor

- **Para emprendedores**: profesionalizar la gestión de pedidos, reducir errores, ganar tiempo, llegar a más clientes.
- **Para clientes**: experiencia de compra clara, ordenada y rápida sin registros innecesarios.

---

## 3. Scope

### Dentro del alcance

- Catálogo de productos público con buscador y filtros.
- Carrito de compras sin login.
- Generación de pedidos y envío automático por WhatsApp.
- Checkout con datos mínimos (nombre, teléfono, dirección, observaciones).
- Backoffice (Filament) para gestión completa.
- Dos métodos de pago: Mercado Pago (link externo) y Efectivo/Transferencia.

### Fuera del alcance

- Gestión de logística y entregas (tracking externo).
- Seguimiento de conversaciones de WhatsApp.
- Marketplace o multi-merchant.
- Cuentas de usuarios finales.

---

## 4. Stakeholders y roles

### Stakeholders

- **Emprendedores / comercios**: dueños del negocio.
- **Clientes finales**: compradores sin cuenta.

### Roles del sistema

- **User**: visitante no autenticado que navega, agrega productos al carrito y crea pedidos.
- **Merchant**: usuario autenticado que gestiona productos, pedidos, reportes y configuración.

### Casos de uso principales

- User navega catálogo, agrega productos al carrito y genera pedido.
- Sistema envía pedido por WhatsApp al merchant.
- Merchant gestiona estados del pedido, edita datos y administra el catálogo.

---

## 5. Funcionalidades

### Front público (User)

- Listado de productos con precios visibles.
- Buscador y filtros por categoría.
- Variantes de producto (ej: talle, color).
- Carrito de compras (agregar, quitar, modificar cantidades).
- Checkout sin registro.
- Formulario de pedido:
    - nombre (requerido)
    - teléfono (requerido, validado)
    - dirección
    - observaciones
    - método de pago (Mercado Pago o Efectivo/Transferencia)
- Confirmación de pedido con disparo automático de WhatsApp.

### Backoffice (Merchant)

- Dashboard con métricas clave.
- Gestión de productos (stock, estado activo, variantes, precios, categoría).
- Gestión de categorías (un producto pertenece solo a una categorîa).
- Gestión de pedidos:
    - **Edición manual** (ver límites en sección 6).
    - Cambio de estados (OrderStatus y PaymentStatus).
    - Notas internas.
    - Auditoría de cambios de estado.
- Gestión de promociones (descuentos porcentuales, precios fijos, vigencia por fecha).
- Reportes:
    - Ventas por período.
    - Productos más pedidos.
    - Pedidos por estado.

---

## 6. Límites de edición manual de pedidos

El merchant puede editar pedidos desde el backoffice con las siguientes **restricciones**:

### Permitido editar

- Dirección de entrega.
- Teléfono del cliente.
- Observaciones / notas.
- Cantidades de items existentes (respetando stock disponible).
- Método de pago (Mercado Pago ↔ Efectivo/Transferencia).
- Estados del pedido (OrderStatus y PaymentStatus).

### NO permitido editar

- Agregar o quitar productos (el cliente debe crear un nuevo pedido).
- Modificar precios históricos de los items (se respeta el precio al momento de creación).
- Editar pedidos en estado `delivered` o `refunded` (solo lectura).
- Cambiar datos del cliente que comprometan trazabilidad (nombre).

**Nota**: El estado de pago (`PaymentStatus`) puede modificarse manualmente o actualizarse vía webhook de Mercado Pago.

---

## 7. Anti-abuso y configuración de seguridad

### Límite de pedidos activos por teléfono

- **Valor por defecto**: 2 pedidos activos (configurable entre 2 y 5).
- Se considera "activo" un pedido en estado `new`, `confirmed` o `in_delivery`.
- Validación por número de teléfono normalizado.
- Configurable en `.env`: `MAX_ACTIVE_ORDERS_PER_PHONE=2`.

### Rate limiting en creación de pedidos

- **Por IP**: 5 intentos por hora (configurable).
- **Por teléfono**: 3 pedidos por hora (configurable).
- Configurable en `.env`:
    - `ORDER_RATE_LIMIT_IP=5` (intentos/hora)
    - `ORDER_RATE_LIMIT_PHONE=3` (pedidos/hora)

### Otras medidas

- Captcha invisible (hCaptcha o reCAPTCHA v3).
- Validación estricta de teléfono (formato, longitud).
- Protección CSRF en todos los formularios.
- Honeypot en checkout.
- Consentimiento explícito para contacto por WhatsApp.

---

## 8. Modelo de dominio

### Entidades principales

- **Product**: producto con precio base, stock, estado activo, categoría.
- **ProductVariant**: variaciones de un producto (ej: talle, color) con precio y stock propio.
- **Category**: categoría de productos (uno a uno).
- **Order**: pedido con datos del cliente, dirección, método de pago, estados.
- **OrderItem**: línea de pedido con producto/variante, cantidad, precio histórico.
- **Address**: dirección de entrega asociada al pedido.
- **Promotion**: promoción con tipo (porcentaje o precio fijo), vigencia, productos aplicables.

### Value Objects y Casts

- **Uso de Value Objects**: propiedades complejas de modelos se mapean mediante Casts de Eloquent a Value Objects.
- **Organización**: `app/ValueObjects/{ModelName}/` (ej: `Product/PriceValueObject.php`).
- **Compartidos**: Value Objects reutilizables (ej: `Money`, `Quantity`, `PhoneNumber`) van en `app/ValueObjects/`.
- **Wireable**: todos los Value Objects implementan `Wireable` para compatibilidad con Livewire.

### Reglas de negocio

- Un producto pertenece a una sola categoría.
- Un pedido tiene una única dirección.
- El stock se descuenta al crear el pedido (transaccional).
- Se guarda siempre el precio histórico del momento de compra.
- Las promociones se aplican según vigencia y no se acumulan.

---

## 9. Flujo crítico: creación de pedido

1. User navega catálogo y agrega productos al carrito.
2. User accede al checkout y completa formulario (nombre, teléfono, dirección, observaciones, método de pago).
3. User confirma pedido.
4. Sistema valida stock, aplica promociones y crea el pedido con estado `new` y pago `pending`.
5. Sistema descuenta stock de forma transaccional.
6. Sistema genera link de WhatsApp con mensaje prearmado y lo envía al merchant.
7. Si el envío de WhatsApp falla, el pedido queda registrado igualmente (no bloquea la operación).

---

## 10. Estados del pedido

### OrderStatus (estado del pedido)

- `new`: pedido creado, esperando confirmación del merchant.
- `confirmed`: merchant confirmó el pedido.
- `in_delivery`: pedido en camino.
- `delivered`: entregado al cliente.
- `rejected`: merchant rechazó el pedido.
- `cancelled`: cliente o merchant canceló.

### PaymentStatus (estado del pago)

- `pending`: pago pendiente.
- `paid`: pago confirmado (manual o vía webhook de Mercado Pago).
- `refunded`: reembolsado.

Ambos estados son independientes y editables manualmente por el merchant.

---

## 11. Métodos de pago

### Mercado Pago

- Link externo generado por el sistema.
- Actualización de estado vía webhook (automático) o manual.

### Efectivo / Transferencia

- Pago coordinado fuera del sistema.
- Merchant actualiza manualmente el estado a `paid` una vez confirmado.

El merchant puede cambiar el método de pago de un pedido en cualquier momento (ej: cliente decide pagar en efectivo en
lugar de Mercado Pago).

---

## 12. Promociones

Soportadas en MVP:

- **Descuento porcentual**: % de descuento sobre precio original.
- **Precio fijo promocional**: precio especial para un producto.
- **Vigencia**: rango de fechas (inicio/fin).

No incluye:

- Cupones de descuento.
- Reglas combinadas (2x1, compra mínima, etc.).
- Descuentos por categoría o carritos completos.

---

## 13. Riesgos y mitigaciones

| Riesgo                           | Mitigación                                                     |
|----------------------------------|----------------------------------------------------------------|
| Pedidos falsos o abusivos        | Límite de pedidos activos por teléfono, rate limiting, captcha |
| Saturación del canal de WhatsApp | Control de envíos, queue asíncrona                             |
| Inconsistencias de stock         | Control transaccional con locks pessimistas                    |
| Fallo en envío de WhatsApp       | Pedido queda registrado, merchant lo ve en backoffice          |
| Falta de pago                    | Estados independientes, seguimiento manual                     |

---

## 14. Stack técnico

### Backend / Frontend

- **PHP**: 8.5+
- **Laravel**: 12
- **Laravel Modules**: organización modular con módulos auto-contenidos
- **Livewire**: 3 + Volt (sintaxis simplificada)
- **Filament**: 4 (backoffice)
- **TailwindCSS**: 4
- **Alpine.js**: 3.x
- **PostgreSQL**: 15+
- **Redis**: cache y queues

### Testing y calidad

- **Pest**: 4 (testing framework)
- **Pint**: code style (PSR-12 estricto)
- **Rector**: refactoring automatizado
- **Larastan**: análisis estático (nivel max)

### Infraestructura

- **Docker**: desarrollo local (Laravel Sail)
- **Comandos**: todos ejecutados vía `./vendor/bin/sail` (migrate, test, pint, phpstan, rector)
- **Digital Ocean**: producción (droplets + managed DB)
- **GitHub Actions**: CI/CD

### Integraciones

- **WhatsApp**: MVP con `wa.me`, preparado para migración a WhatsApp Business API.
- **Mercado Pago**: links de pago externos, webhooks para actualización de estado.

---

## 15. Convenciones técnicas

### Calidad de código (obligatorio)

- **Tipado fuerte**: todo el código con type hints completos (parámetros, returns, propiedades).
- **PHPStan level 6+**: análisis estático obligatorio antes de commit.
- **Pint**: formateo automático (PSR-12 estricto).
- **Rector**: refactoring automatizado cuando aplique.
- **Cobertura de tests**: prácticamente 100% con Pest 4.

### Arquitectura y diseño

- **Clases `final`**: todas las clases deben ser `final`, sin métodos `protected`.
- **Single responsibility**: Controllers y Actions con un solo método público.
- **Sin arrays públicos**: métodos públicos solo reciben/devuelven Value Objects o Spatie Laravel Data.
- **Value Objects**: implementar tempranamente, organizar en `app/ValueObjects/{ModelName}/`.
    - Usar `Wireable` para compatibilidad con Livewire.
    - Mapear con Casts de Eloquent.
    - Compartidos van en `app/ValueObjects/` (ej: `Money`, `Quantity`).
- **Validación con excepciones**: throw en lugar de null o valores por defecto.
- **Modelos con Factory**: siempre incluir Factory para cada modelo.

### Estructura de tests

- **Smoke tests obligatorios**: todas las páginas UI en `tests/Browser/SmokeTest.php`.
- **Estructura espejo**: `tests/Feature/` replica estructura de `app/`.
- **Cobertura completa**: Feature tests preferidos sobre Unit tests.

### APIs REST

- **Versionado**: rutas siempre versionadas (`api/v1/*`).
- **Resources**: endpoints siempre retornan `Resource` o `ResourceCollection`.
- **Testing**: incluir archivos `.http` y `.bru` (Bruno) para pruebas manuales.

### Base de datos

- **Índices**: incluir siempre para optimizar consultas.
- **Migraciones sensibles**: etiquetar con `migration-affects-production`.
- **Enums PHP**: preferir enums de PHP sobre enums de base de datos.

### Comandos comunes

```zsh
./vendor/bin/sail up -d
./vendor/bin/sail artisan migrate:fresh --seed
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail composer run rector
./vendor/bin/sail composer run phpstan
./vendor/bin/sail test
```

---

## 16. Auditoría y trazabilidad

### Qué se audita

- **Cambios de estado de pedidos** (OrderStatus y PaymentStatus):
    - Quién realizó el cambio (user_id del merchant).
    - Cuándo (timestamp).
    - Estado anterior y nuevo.

### Qué NO se audita en MVP

- Ediciones de campos individuales del pedido (dirección, teléfono, cantidades).
- Cambios en productos o categorías.
- Accesos al backoffice.

### Testing y validación

- **Smoke tests**: obligatorios para todas las páginas con Pest 4.
- **Archivos `.http` y `.bru`**: para testing manual/automatizado de APIs.
- **Cobertura**: prácticamente 100% de funcionalidad crítica.

Implementación: tabla `order_status_logs` con:

- `order_id`
- `user_id`
- `field` (order_status | payment_status)
- `old_value`
- `new_value`
- `created_at`

---

## 17. Notas de seguridad

- **Endpoints públicos**: protección con rate limiting específico por endpoint, CSRF, validaciones estrictas.
- **Validaciones con excepciones**: throw en lugar de null, FormRequest o clases con `Illuminate\Validation`.
- **Datos personales**: consentimiento explícito para contacto vía WhatsApp, cumplimiento GDPR básico.
- **Credenciales**: nunca exponer API keys en frontend, usar variables de entorno.
- **SQL Injection**: uso exclusivo de Eloquent ORM y query builder.
- **XSS**: escapado automático de Blade, validación de inputs.
- **Sesiones**: configuración segura (httponly, secure en producción, SameSite=Lax).
- **Migraciones sensibles**: etiquetar con `migration-affects-production` para revisión manual.

---

## Resumen ejecutivo

Sistema e-commerce monolítico single-tenant para emprendedores, con catálogo público, carrito sin registro, checkout
simple, envío automático por WhatsApp y backoffice completo. Stack: Laravel 12 + Livewire 3 + Filament 4 + PostgreSQL.
Foco en simplicidad, seguridad y mantenibilidad. MVP con funcionalidad esencial, preparado para escalar funcionalidades
según feedback real.

