# Slice 4 — Pagos (Mercado Pago) con normalización de estados

## 1. Objetivo

Proveer una capa estándar de pagos: crear transacciones de pago y recibir webhooks de Mercado Pago para normalizar
estados internos (`pending`, `paid`, `failed`) de forma idempotente.

---

## 2. Actores

- Actor principal: ConsumerApp (E-commerce)
- Actores secundarios: PaymentProvider (Mercado Pago)

---

## 3. Precondiciones

- Slice 1 implementado.
- Existe configuración del provider por proyecto (tokens/credenciales) almacenada de forma segura.
- Idealmente Slice 3 disponible para emitir `order.paid` como webhook saliente (opcional pero recomendado).

---

## 4. Flujo principal (Happy Path)

Given que el ConsumerApp necesita iniciar un cobro
When llama al endpoint de creación de pago con monto y referencia
Then el BaaS crea una `payment_transaction` interna en estado `pending` y devuelve los datos necesarios para continuar
el flujo (preferencia/checkout/redirect, según MP).

Given que Mercado Pago notifica un cambio de estado
When llama al webhook del BaaS
Then el BaaS valida el webhook, actualiza la transacción de forma idempotente y normaliza el estado interno.

Given que el estado pasa a `paid`
Then el BaaS puede emitir un evento interno equivalente a `order.paid` para integraciones (si Slice 3 está presente).

---

## 5. Flujos alternativos / errores relevantes

- Given el ConsumerApp envía datos inválidos de pago
  Then se rechaza 422.

- Given Mercado Pago reintenta el mismo webhook
  Then el BaaS procesa idempotentemente (sin duplicar efectos).

- Given el webhook no se puede validar
  Then se rechaza 401/403 y se registra evento de seguridad.

---

## 6. Reglas de negocio

- Estados internos normalizados: `pending`, `paid`, `failed`.
- Toda transacción se asocia a `project_id`.
- `external_id` identifica el recurso del provider y debe ser único por provider+proyecto.

---

## 7. Estados involucrados (si aplica)

- PaymentTransaction: `pending` → `paid` | `failed`.

---

## 8. Validaciones clave

- `amount` debe ser > 0 y con precisión válida.
- `currency` (si aplica) debe ser consistente por proyecto.
- Webhook debe mapearse a una transacción interna (por `external_id` o metadata).

---

## 9. Autorización y seguridad

- Endpoint de creación: protegido por API key.
- Endpoint de webhook del provider: público (sin API key) pero validado con mecanismo del provider.
- Seguridad:
    - No exponer tokens del provider en logs.
    - Validación estricta del webhook para evitar fraude.
    - Idempotencia obligatoria.

---

## 10. Límites del slice (out of scope)

- No incluye múltiples providers.
- No incluye reconciliación contable.
- No incluye billing por suscripción del BaaS.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Tabla `payment_transactions`: `project_id`, `provider`, `external_id`, `status`, `amount`, timestamps.
- Diseño “provider abstraction”: interfaz `PaymentProvider` con métodos `createPayment()` y `handleWebhook()`.
- Webhook validation: seguir el mecanismo oficial de Mercado Pago para firmas/secret.

---

## 12. Criterio de finalización

- Se puede crear un pago y persiste la transacción.
- Webhooks actualizan estado de forma idempotente.
- Estado normalizado consistente.
- Logs útiles y seguros.
