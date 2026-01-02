# Project Definition — E-commerce BaaS (MVP)

## 1. Objetivo

Diseñar y desarrollar un **BaaS orientado a e-commerce**, inicialmente de uso interno, con foco en:

* Acelerar el desarrollo de proyectos e-commerce.
* Centralizar servicios transversales repetidos.
* Sentar bases técnicas para monetización por suscripción.

El MVP debe ser **usable desde el primer proyecto real**, sin depender de clientes externos.

---

## 2. Alcance del MVP

### Incluye

* API centralizada (Laravel 12, API-only).
* Autenticación por **API Key por proyecto**.
* Servicios core reutilizables.
* Preparación para planes y feature flags.

### Excluye

* Multi-tenant complejo (subdominios, cuentas finales).
* Dashboard para clientes externos.
* Billing automático.
* Infraestructura avanzada (K8s, colas distribuidas, etc).

---

## 3. Usuario objetivo (MVP)

* **Usuario primario:** desarrollador (vos).
* **Casos de uso:** múltiples e-commerce custom o semi-custom.
* **Consumo:** vía HTTP API o SDK Laravel.

---

## 4. Servicios del MVP

### 4.1 Notificaciones

**Descripción**
Servicio unificado para envío de notificaciones desde los e-commerce.

**Canales**

* Email (SMTP / provider simple).
* WhatsApp (wa.me inicialmente).

**Funcionalidades**

* Templates versionados.
* Envío asíncrono.
* Estados (`pending`, `sent`, `failed`).

---

### 4.2 Events & Webhooks

**Descripción**
Sistema de eventos para desacoplar lógica del core e-commerce.

**Eventos iniciales**

* `order.created`
* `order.paid`

**Funcionalidades**

* Webhooks salientes por proyecto.
* Reintentos automáticos.
* Firma HMAC.
* Logs básicos.

---

### 4.3 Pagos (Integración Abstracta)

**Descripción**
Capa de integración unificada para proveedores de pago.

**Proveedor MVP**

* Mercado Pago.

**Funcionalidades**

* Creación de pago.
* Recepción de webhooks.
* Normalización de estados (`pending`, `paid`, `failed`).

---

## 5. Arquitectura

### 5.1 Stack

* Backend: Laravel 12 (API-only).
* Base de datos: PostgreSQL.
* Queue: Database / Redis (simple).
* Auth: API Key.

---

### 5.2 Comunicación

* Cada request incluye `X-API-KEY`.
* Resolución de contexto por `project_id`.
* Rate limit por proyecto.

---

## 6. Modelo de datos (inicial)

### projects

* id
* name
* domain
* api_key_hash
* plan
* is_active
* created_at

### notifications

* id
* project_id
* channel
* payload
* status
* created_at

### payment_transactions

* id
* project_id
* provider
* external_id
* status
* amount
* created_at

### webhook_logs

* id
* project_id
* event
* target_url
* status
* response_code
* created_at

---

## 7. Gestión de proyectos (Project Management)

### 7.1 Alta de proyecto (tienda)

**Objetivo**
Permitir registrar una nueva tienda/e-commerce que consumirá el BaaS.

**Forma (MVP)**

* Alta manual vía:

    * Seeder
    * Comando Artisan
    * Endpoint protegido (solo uso interno)

**Datos mínimos**

* Nombre del proyecto
* Dominio principal (informativo / validación futura)
* Plan inicial (`free`)

---

### 7.2 Generación de API Key

**Flujo**

1. Al crear el proyecto se genera una API Key aleatoria.
2. La API Key se muestra **una sola vez**.
3. Se almacena únicamente el **hash** en base de datos.

**Uso**

* Header: `X-API-KEY`
* Resolución de contexto por proyecto.

---

### 7.3 Rotación y revocación

* Regenerar API Key invalida la anterior.
* Posibilidad de desactivar un proyecto (`is_active = false`).

---

### 7.4 Autorización

* Cada request valida:

    * API Key válida
    * Proyecto activo
* Scope implícito por proyecto (no cross-project access).

---

## 8. Planes y monetización (preparado, no activo)

(preparado, no activo)

* Planes: `free`, `pro`.
* Feature flags por proyecto.
* Límites configurables por servicio.

> Implementación sugerida: Laravel Pennant.

---

## 8. Métricas de validación

* Tiempo de desarrollo ahorrado por proyecto.
* Cantidad de integraciones reutilizadas.
* Reducción de lógica duplicada.
* Facilidad de incorporar un nuevo e-commerce.

---

## 9. Seguridad (obligatorio)

* API Keys almacenadas **hasheadas**.
* Webhooks con firma HMAC.
* Rate limiting por proyecto.
* Logs sin datos sensibles.

---

## 10. Definition of Done — MVP terminado

El MVP se considera **terminado** cuando se cumplen **todos** los siguientes criterios:

### 10.1 Funcionales

* Existe al menos **1 proyecto activo** consumiendo el BaaS en un e-commerce real.
* El servicio de **Notificaciones**:

    * Envía emails correctamente desde el e-commerce.
    * Envía mensajes vía WhatsApp (wa.me).
    * Maneja estados (`pending`, `sent`, `failed`).
* El sistema de **Events & Webhooks**:

    * Dispara eventos `order.created` y `order.paid`.
    * Firma los webhooks con HMAC.
    * Reintenta fallos y registra logs.
* La integración de **Pagos (Mercado Pago)**:

    * Crea pagos desde el e-commerce.
    * Recibe webhooks del provider.
    * Normaliza estados internos.

---

### 10.2 Técnicos

* API protegida por **API Key hasheada** por proyecto.
* Rate limiting activo por proyecto.
* Todas las operaciones críticas son **asíncronas** (queues).
* Logs claros y consultables para:

    * Notificaciones
    * Webhooks
    * Pagos
* Código desacoplado del e-commerce (no dependencias directas).

---

### 10.3 Reutilización

* Un segundo e-commerce puede integrarse:

    * Sin copiar lógica de negocio.
    * Solo configurando API Key y endpoints.
* Existe al menos **un SDK Laravel** o cliente HTTP reutilizable.

---

### 10.4 Operativos

* Deploy reproducible (script o pipeline simple).
* Variables de entorno documentadas.
* Errores críticos monitoreables (logs o alerts básicos).

---

### 10.5 Negocio / Validación

* Mediste al menos **un caso concreto** de tiempo ahorrado.
* Confirmaste que el BaaS:

    * Reduce cambios en el e-commerce.
    * Centraliza integraciones repetidas.
* El código está preparado para:

    * Planes (`free`, `pro`).
    * Feature flags.

---

## 11. Roadmap posterior al MVP

* Dashboard básico.
* Más proveedores de pago.
* SMS / Push notifications.
* Billing automático.
* Multi-tenant avanzado.
