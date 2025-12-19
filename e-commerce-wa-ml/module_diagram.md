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

#### Relación con el Flujo de Compra sin Login

Auth tiene un **rol unidireccional** en el sistema:

- **NO interviene en el flujo público de compra**: Users (clientes finales) navegan el catálogo, agregan productos al carrito y generan pedidos **sin ninguna autenticación**. El sistema no valida credenciales, no crea cuentas de usuario, y no requiere login para checkout.
  
- **Protege el backoffice exclusivamente**: Auth solo controla el acceso de los **merchants** al panel administrativo de Filament. Merchants autenticados pueden:
  - Gestionar productos, categorías, variantes y promociones (módulo Catalog)
  - Visualizar, editar y cambiar estados de pedidos (módulo Orders)
  - Actualizar manualmente estados de pago o revisar webhooks (módulo Payments)
  - Consultar reportes de ventas y métricas (módulo Reports)

- **Security complementa donde Auth no aplica**: Mientras Auth protege el backoffice con sesiones y permisos, el módulo **Security** (anti-abuso) protege los endpoints públicos mediante:
  - Rate limiting por IP y por teléfono
  - Captcha invisible en checkout
  - Validación estricta de formularios
  - Límite de pedidos activos por teléfono

**Decisión de diseño**: Se eligió un flujo sin login para clientes finales para reducir fricción en la conversión, optimizar la experiencia móvil (contexto de WhatsApp) y alinearse con el modelo de negocio de emprendedores pequeños donde los clientes prefieren rapidez sobre crear cuentas.

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

---

## Orden Sugerido para el Desarrollo

### Fase 1: Fundamentos (Base del Sistema)

**1. Auth (Autenticación)**
- **Por qué primero:** Necesario para acceder al backoffice de Filament
- **Alcance mínimo:** Login básico de merchants, sesiones, middleware de autenticación
- **Validación:** Merchant puede loguearse y ver dashboard vacío de Filament

**2. Catalog (Catálogo)**
- **Por qué:** Base de datos de productos necesaria para todo el flujo
- **Alcance mínimo:**
  - Modelos: Product, Category, ProductVariant
  - CRUD completo en Filament (backoffice)
  - Gestión de stock y precios
  - Relación producto-categoría (uno a uno)
- **Validación:** Merchant puede crear productos con variantes y verlos en Filament

### Fase 2: Flujo Crítico (MVP Funcional)

**3. Cart (Carrito)**
- **Por qué:** Permite a users agregar productos y preparar pedido
- **Alcance mínimo:**
  - Vista pública del catálogo (listado, detalle)
  - Carrito en sesión (agregar, quitar, modificar cantidades)
  - Checkout básico con formulario (sin pago ni confirmación aún)
- **Dependencias:** Catalog
- **Validación:** User puede navegar catálogo, agregar al carrito y ver formulario de checkout

**4. Orders (Pedidos) - CORE**
- **Por qué:** Módulo central, conecta Cart con el negocio
- **Alcance mínimo:**
  - Creación de pedido desde Cart
  - Validación de stock transaccional
  - Estados básicos (OrderStatus: new, confirmed, cancelled)
  - Address (dirección de entrega)
  - Gestión en Filament (ver pedidos, cambiar estados)
  - Auditoría de cambios de estado
- **Dependencias:** Catalog, Cart
- **Validación:** User crea pedido, se descuenta stock, merchant ve pedido en backoffice

**5. Security (Anti-abuso)**
- **Por qué:** Proteger flujo público antes de lanzar
- **Alcance mínimo:**
  - Rate limiting por IP y teléfono
  - Captcha invisible en checkout
  - Validación de teléfono
  - Límite de pedidos activos por teléfono
- **Dependencias:** Orders
- **Validación:** Endpoints públicos protegidos, límites funcionando

### Fase 3: Integraciones (Comunicación Externa)

**6. WhatsApp (Notificaciones)**
- **Por qué:** Notificar al merchant sobre nuevos pedidos
- **Alcance mínimo:**
  - Envío de mensaje vía `wa.me` al crear pedido
  - Cola asíncrona para gestión de envíos
  - Notificación de cambios de estado
- **Dependencias:** Orders
- **Validación:** Merchant recibe mensaje de WhatsApp al crearse un pedido

**7. Payments (Pagos)**
- **Por qué:** Habilitar métodos de pago y gestión de cobros
- **Alcance mínimo:**
  - Selección de método en checkout (Mercado Pago / Efectivo-Transferencia)
  - Generación de link de Mercado Pago
  - PaymentStatus (pending, paid, refunded)
  - Webhook de Mercado Pago para actualización automática
  - Actualización manual por merchant
- **Dependencias:** Orders
- **Validación:** User puede pagar con Mercado Pago, webhook actualiza estado, merchant puede marcar como pagado manualmente

### Fase 4: Mejoras y Análisis (Post-MVP)

**8. Catalog - Promotions (Promociones)**
- **Por qué:** Agregar capacidad de descuentos
- **Alcance:**
  - Modelo Promotion (porcentaje, precio fijo, vigencia)
  - Aplicación automática en Cart y Orders
  - Gestión en Filament
- **Dependencias:** Catalog (extensión del módulo existente)
- **Validación:** Merchant crea promoción, se aplica automáticamente en checkout

**9. Reports (Reportes)**
- **Por qué:** Métricas para decisiones de negocio (solo lectura)
- **Alcance:**
  - Ventas por período
  - Productos más pedidos
  - Pedidos por estado
  - Dashboard con widgets en Filament
- **Dependencias:** Orders, Payments, Catalog
- **Validación:** Merchant visualiza reportes con datos reales

---

### Resumen del Orden

```
Fase 1 (Fundamentos):
  1. Auth
  2. Catalog

Fase 2 (MVP Funcional):
  3. Cart
  4. Orders ⭐ CORE
  5. Security

Fase 3 (Integraciones):
  6. WhatsApp
  7. Payments

Fase 4 (Post-MVP):
  8. Promotions (extensión de Catalog)
  9. Reports
```

### Criterios de Decisión

- **Dependencias técnicas:** Un módulo debe completarse antes si otro depende de él
- **Valor de negocio:** Priorizar flujo crítico (catálogo → carrito → pedido)
- **Riesgo:** Abordar integraciones externas (WhatsApp, Mercado Pago) una vez el core es estable
- **Testing incremental:** Cada fase debe poder testearse de forma aislada antes de continuar

### Validación por Fase

- **Fin Fase 1:** Merchant autenticado puede gestionar productos en Filament
- **Fin Fase 2:** User puede crear pedido completo, merchant lo ve y gestiona (MVP MÍNIMO)
- **Fin Fase 3:** Sistema notifica por WhatsApp y acepta pagos externos
- **Fin Fase 4:** Sistema completo con promociones y métricas de negocio

