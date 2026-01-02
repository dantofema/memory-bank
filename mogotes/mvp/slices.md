# Vertical Slices

Slice 0 — Bootstrap del BaaS (API + DB + health)
Valor: Tener una API operable y deployable desde el día 1 para construir encima.

Slice 1 — Gestión de proyectos + autenticación por API Key
Valor: Asegurar aislamiento por proyecto y un mecanismo simple de autenticación para consumir el BaaS.

Slice 2 — Notificaciones (Email + WhatsApp wa.me)
Valor: Centralizar notificaciones reutilizables con envío asíncrono y estados auditable.

Slice 3 — Events & Webhooks (HMAC + reintentos + logs)
Valor: Desacoplar integraciones del e-commerce disparando webhooks firmados y reintentables.

Slice 4 — Pagos (Mercado Pago) con normalización de estados
Valor: Proveer una capa unificada de pagos reutilizable, con webhooks de provider e idempotencia.

Slice 5 — SDK Laravel / cliente HTTP reutilizable
Valor: Reducir fricción y evitar lógica duplicada en e-commerces consumidores.

Slice 6 — Operatividad mínima (límites, observabilidad y documentación)
Valor: Hacer el MVP operable y desplegable de forma reproducible con seguridad básica.
