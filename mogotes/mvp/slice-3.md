# Slice 3 — Events & Webhooks (HMAC + reintentos + logs)

## 1. Objetivo

Desacoplar integraciones del e-commerce permitiendo registrar eventos (`order.created`, `order.paid`) y disparar
webhooks salientes por proyecto con firma HMAC, reintentos y logs básicos.

---

## 2. Actores

- Actor principal: ConsumerApp (E-commerce)
- Actores secundarios: WebhookReceiver (Sistema externo propiedad del mismo Developer)

---

## 3. Precondiciones

- Slice 1 implementado.
- Cola/worker disponible.

---

## 4. Flujo principal (Happy Path)

Given que el ConsumerApp necesita informar un evento
When envía `event_name` y `data` a un endpoint del BaaS
Then el BaaS valida, persiste lo mínimo necesario y encola entregas de webhook.

Given que el proyecto tiene webhooks configurados
When el worker ejecuta la entrega
Then hace una request HTTP al `target_url` con firma HMAC y registra el resultado.

Given que el receptor responde 2xx
Then la entrega queda marcada como `sent` (o equivalente) y el log registra `response_code`.

---

## 5. Flujos alternativos / errores relevantes

- Given `event_name` no permitido
  Then se rechaza con 422.

- Given fallo de red o respuesta 5xx
  Then se reintenta con backoff hasta N veces y se registra el último estado.

- Given respuesta 4xx
  Then no se reintenta (o se reintenta solo un número limitado), y se marca como failed.

---

## 6. Reglas de negocio

- Eventos permitidos MVP: `order.created`, `order.paid`.
- Cada evento se procesa en contexto de `project_id`.
- Se firma con HMAC para que el receptor valide integridad y autenticidad.

---

## 7. Estados involucrados (si aplica)

- Webhook delivery: `pending` → `sent` | `failed`.

---

## 8. Validaciones clave

- `event_name` debe ser uno de los eventos permitidos.
- `target_url` debe ser https (o http solo en local) y validarse para evitar SSRF.
- `data` debe tener tamaño máximo y campos permitidos.

---

## 9. Autorización y seguridad

- Acceso: Endpoint de ingest protegido por API key.
- Seguridad:
    - Firma HMAC con secreto por proyecto (no reusar API key).
    - Evitar SSRF: bloquear IPs privadas/localhost si se permite configuración por DB.
    - No loggear payload completo si contiene datos sensibles.

---

## 10. Límites del slice (out of scope)

- No incluye UI para gestionar webhooks.
- No incluye múltiples tipos de eventos ni esquema versionado complejo.
- No incluye colas distribuidas.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Tabla `webhook_logs`: `project_id`, `event`, `target_url`, `status`, `response_code`, timestamps.
- Configuración de webhooks por proyecto:
    - opción A: tabla `project_webhooks` (project_id, event, target_url, is_active)
    - opción B: config estática por entorno
    - preferir A para no recompilar/deployar.
- Firma HMAC:
    - Input: cuerpo raw del request.
    - Header: `X-Signature` (por ejemplo `sha256=...`).

---

## 12. Criterio de finalización

- Crear evento dispara entregas webhook.
- Firma HMAC documentada y verificable.
- Reintentos funcionando con backoff.
- Logs consultables (mínimos) por proyecto.
