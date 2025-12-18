# Diagrama de Módulos del Sistema

```mermaid
graph TB
    subgraph Auth["🔐 Auth (Autenticación) - TRANSVERSAL"]
        AUTH_LOGIN[Login Merchant]
        AUTH_SESSION[Gestión de Sesiones]
        AUTH_PERMS[Control de Acceso]
    end

    subgraph Security["🔒 Security (Anti-abuso) - TRANSVERSAL"]
        SEC_RATE[Rate Limiting]
        SEC_CAPTCHA[Captcha]
        SEC_VALID[Validaciones]
    end

    subgraph Catalog["🏪 Catalog (Catálogo)"]
        CAT_PROD[Products]
        CAT_CAT[Categories]
        CAT_VAR[Variants]
        CAT_PROM[Promotions]
    end

    subgraph Cart["🛒 Cart (Carrito)"]
        CART_ITEMS[Items Management]
        CART_CHECKOUT[Checkout]
    end

    subgraph Orders["📋 Orders (Pedidos) - CORE"]
        ORD_CREATE[Creación]
        ORD_MANAGE[Gestión]
        ORD_STATUS[Estados]
        ORD_ADDR[Address]
        ORD_AUDIT[Auditoría]
    end

    subgraph Payments["💳 Payments (Pagos)"]
        PAY_MP[Mercado Pago]
        PAY_STATUS[Payment Status]
        PAY_WEBHOOK[Webhooks]
    end

    subgraph WhatsApp["💬 WhatsApp (Notificaciones)"]
        WA_SEND[Envío de Mensajes]
        WA_QUEUE[Cola de Envío]
    end

    subgraph Reports["📊 Reports (Reportes)"]
        REP_SALES[Ventas]
        REP_PRODUCTS[Productos]
        REP_ORDERS[Estados Pedidos]
    end

%% Relaciones entre módulos - Flujo principal
    Catalog -->|Consulta productos y precios| Cart
    Catalog -->|Aplica promociones| Cart
    Cart -->|Genera pedido| Orders
    Catalog -->|Valida stock y promociones| Orders
    Payments -->|Métodos disponibles| Cart
    Orders -->|Solicita pago| Payments
    Payments -->|Confirma/actualiza pago| Orders
    Orders -->|Notifica creación y estados| WhatsApp
    Payments -->|Notifica confirmación| WhatsApp
%% Auth controla acceso al backoffice
    Auth -.->|Protege| Catalog
    Auth -.->|Protege| Orders
    Auth -.->|Protege| Payments
    Auth -.->|Protege| Reports
%% Security protege endpoints públicos y privados
    Security -.->|Protege| Catalog
    Security -.->|Protege| Cart
    Security -.->|Protege| Orders
    Security -.->|Protege| Payments
    Security -.->|Protege| WhatsApp
    Security -.->|Protege| Reports
%% Reports consume datos (solo lectura)
    Orders -.->|Datos| Reports
    Payments -.->|Datos| Reports
    Catalog -.->|Datos| Reports
%% Estilos
    classDef coreModule fill: #ff6b6b, stroke: #c92a2a, stroke-width: 3px, color: #fff
    classDef transversalModule fill: #4ecdc4, stroke: #0d9488, stroke-width: 3px, color: #fff
    classDef standardModule fill: #95a5a6, stroke: #5d6d7e, stroke-width: 2px, color: #fff
    class Orders coreModule
    class Auth, Security transversalModule
    class Catalog, Cart, Payments, WhatsApp, Reports standardModule
```

## Descripción de Módulos

### 🔐 Auth (Autenticación) - TRANSVERSAL

**Responsabilidad:** Autenticación y control de acceso al backoffice (Filament).

- Login de merchants
- Gestión de sesiones
- Control de permisos para gestión de productos, pedidos y reportes
- **Nota:** El frontend público (catálogo/carrito) NO requiere autenticación

### 🏪 Catalog (Catálogo)

**Responsabilidad:** Gestión de productos, categorías, variantes y promociones.

- **Products:** gestión de productos con precio base, stock, estado activo
- **Categories:** categorización de productos (uno a uno)
- **Variants:** variaciones (talle, color) con precio y stock propio
- **Promotions:** descuentos porcentuales, precios fijos, vigencia por fecha

### 🛒 Cart (Carrito)

**Responsabilidad:** Carrito de compras sin autenticación.

- Gestión de items (agregar, quitar, modificar cantidades)
- Checkout con formulario mínimo (nombre, teléfono, dirección, observaciones)
- Selección de método de pago

### 📋 Orders (Pedidos) - CORE

**Responsabilidad:** Creación, gestión y edición de pedidos con restricciones.

- Módulo central del sistema
- **Creación:** validación de stock, aplicación de promociones, descuento transaccional
- **Gestión:** cambios de estado (OrderStatus y PaymentStatus)
- **Address:** dirección de entrega única por pedido
- **Auditoría:** trazabilidad de cambios de estado (quién, cuándo, qué cambió)
- **Restricciones:** límites de edición según estado del pedido

### 🔒 Security (Anti-abuso) - TRANSVERSAL

**Responsabilidad:** Rate limiting, captcha, validaciones.

- Protege transversalmente todos los módulos
- **Rate Limiting:** por IP y por teléfono (configurable)
- **Captcha:** invisible (hCaptcha o reCAPTCHA v3)
- **Validaciones:** teléfono, formularios, honeypot, CSRF
- **Límites:** pedidos activos por teléfono (2-5 configurable)

### 💬 WhatsApp (Notificaciones)

**Responsabilidad:** Envío de notificaciones por WhatsApp.

- Notificación de creación de pedido al merchant
- Notificaciones de cambio de estado
- Confirmaciones de pago
- Cola asíncrona para control de envíos
- **MVP:** implementación con `wa.me`, preparado para migración a WhatsApp Business API

### 💳 Payments (Pagos)

**Responsabilidad:** Integración con Mercado Pago y gestión de estados de pago.

- **Mercado Pago:** generación de links de pago externos
- **Payment Status:** gestión de estados (pending, paid, refunded)
- **Webhooks:** actualización automática de estado vía Mercado Pago
- **Manual:** merchant puede actualizar estado manualmente (efectivo/transferencia)

### 📊 Reports (Reportes)

**Responsabilidad:** Métricas y análisis (solo lectura).

- Consume datos de otros módulos
- Ventas por período
- Productos más pedidos
- Pedidos por estado
- **Acceso:** solo para merchants autenticados

## Leyenda

- **Líneas sólidas (→):** Dependencias directas y flujo de datos principal
- **Líneas punteadas (-.->):** Relaciones transversales o de solo lectura
- **Color rojo:** Módulo CORE del sistema
- **Color turquesa:** Módulos TRANSVERSALES
- **Color gris:** Módulos estándar

