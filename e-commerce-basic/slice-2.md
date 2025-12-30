# Slice 2 — Notificación WhatsApp

## 1. Objetivo
El administrador recibe una notificación automática vía WhatsApp cada vez que se confirma un nuevo pedido, permitiendo una gestión rápida y directa con el cliente.

---

## 2. Actores
- Actor principal: Sistema (Proceso automático)
- Actores secundarios: Administrador (Receptor de la notificación)

---

## 3. Precondiciones
- Slice 1 completado: El sistema es capaz de generar pedidos (`Order`).
- El número de teléfono del administrador está configurado en las variables de entorno (`.env`).
- El servicio de API de WhatsApp (terceros) está configurado y operativo.

---

## 4. Flujo principal (Happy Path)
Given que un Customer ha finalizado con éxito un pedido (Slice 1)
When el sistema detecta la creación del pedido
Then dispara un Job asíncrono para enviar la notificación.

When el Job se ejecuta
Then construye un mensaje con el detalle del pedido (número, cliente, total) y un enlace al panel administrativo, y lo envía al WhatsApp del administrador.

---

## 5. Flujos alternativos / errores relevantes
- Given que la API de WhatsApp de terceros devuelve un error
  Then el sistema registra el error en los logs y reintenta el Job según la política de reintentos configurada en Laravel.

- Given que el número de teléfono del administrador no está configurado
  Then el Job falla silenciosamente (o registra un error crítico) para no interrumpir la experiencia del usuario final, pero deja rastro en logs.

---

## 6. Reglas de negocio
- La notificación debe ser asíncrona (Queue/Job) para no bloquear la respuesta al cliente.
- El mensaje debe incluir: Número de pedido, Nombre del cliente, Total de la compra y Link directo al pedido en el Admin.
- Se debe usar una API de terceros (según requerimiento no funcional) para evitar la complejidad de WhatsApp Business API oficial en el MVP.

---

## 7. Estados involucrados (si aplica)
- Order: No cambia de estado por la notificación, pero el proceso depende del evento de creación.

---

## 8. Validaciones clave
- Configuración ENV: El número de teléfono debe tener el formato internacional correcto (ej: 549...).
- Integridad del mensaje: El link al panel admin debe generarse correctamente usando rutas nombradas.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo el sistema de forma automática.
- Seguridad: No enviar datos sensibles (como dirección completa si es muy específica) si el canal no se considera 100% seguro, aunque para el MVP se prioriza la información de gestión.
- Protección: Las credenciales de la API de WhatsApp deben estar protegidas en el servidor (`.env`).

---

## 10. Límites del slice (out of scope)
- No incluye notificaciones al cliente.
- No incluye integración oficial con WhatsApp Business (se usa API de terceros).
- No incluye la posibilidad de responder al mensaje para ejecutar acciones en el sistema (es unidireccional).

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Mecanismo: Laravel Queues (usando el driver configurado, ej: database o redis).
- Eventos: Utilizar un `Observer` de Eloquent o un `Event` (OrderCreated) para disparar el Job.
- Configuración: `config/services.php` para almacenar las keys de la API de WhatsApp.
- Formato del mensaje: Template de texto plano con placeholders.

---

## 12. Criterio de finalización
- Al realizar un pedido de prueba, el Job de notificación se encola correctamente.
- El administrador recibe el mensaje en su WhatsApp con los datos reales del pedido.
- Los logs confirman el envío exitoso o el manejo de errores en caso de falla de la API.
