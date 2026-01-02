# Slice 2 — Notificaciones (Email + WhatsApp wa.me)

## 1. Objetivo

Centralizar notificaciones reutilizables para los e-commerces consumidores, con templates versionados, envío asíncrono y
estados trazables (`pending`, `sent`, `failed`).

---

## 2. Actores

- Actor principal: ConsumerApp (E-commerce)
- Actores secundarios: Developer (Interno)

---

## 3. Precondiciones

- Slice 1 implementado (auth + contexto de proyecto).
- Existe una cola disponible (database o Redis) y un worker ejecutándose.
- Existe configuración SMTP válida para el entorno (si se prueba Email real).

---

## 4. Flujo principal (Happy Path)

Given que el ConsumerApp necesita notificar un evento de negocio
When llama al endpoint de notificaciones con un template y parámetros válidos
Then el BaaS crea un registro `pending` y encola un job para enviar.

Given que el worker procesa el job
When el envío es exitoso
Then el registro pasa a `sent` y queda un log mínimo de auditoría.

Given que el canal es WhatsApp
When se procesa la notificación
Then se genera un link `wa.me` (con mensaje prearmado) y se registra como resultado.

---

## 5. Flujos alternativos / errores relevantes

- Given parámetros inválidos para el template
  Then se rechaza con 422 y un mensaje claro.

- Given fallo transient de SMTP
  Then se reintenta hasta N veces y, si persiste, se marca `failed`.

- Given que el payload excede un tamaño permitido
  Then se rechaza para evitar abuso.

---

## 6. Reglas de negocio

- Todas las notificaciones se asocian a un `project_id`.
- Estados permitidos: `pending`, `sent`, `failed`.
- Templates se resuelven por `name` + `version` (versionado explícito).

---

## 7. Estados involucrados (si aplica)

- Notification: `pending` → `sent` | `failed`.

---

## 8. Validaciones clave

- `channel` debe ser uno de: `email`, `whatsapp`.
- `template.name` requerido y alfanumérico con guiones/underscores.
- `template.version` requerido (semver o entero; definir criterio mínimo).
- `payload` debe ser JSON válido y con tamaño máximo.

---

## 9. Autorización y seguridad

- Acceso: Protegido por API key.
- Seguridad:
    - No loggear contenido sensible del payload.
    - Sanitizar y validar parámetros de template para evitar inyección.
    - Rate limiting por proyecto.

---

## 10. Límites del slice (out of scope)

- No incluye SMS ni push.
- No incluye proveedor WhatsApp oficial (solo `wa.me`).
- No incluye UI para gestionar templates.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Tabla `notifications`: `project_id`, `channel`, `payload` (json), `status`, timestamps.
- Templates versionados:
    - opción A: tabla `notification_templates` (name, version, channel, body, subject)
    - opción B: almacenamiento en filesystem + manifest
    - elegir una, priorizando simpleza del MVP.
- Jobs por canal: `SendEmailNotificationJob`, `GenerateWhatsappLinkJob`.

---

## 12. Criterio de finalización

- Endpoint crea notificaciones y encola jobs.
- Worker procesa y actualiza estados.
- Email se envía correctamente en entorno de prueba.
- WhatsApp genera un link `wa.me` correcto.
- Logs útiles sin datos sensibles.
