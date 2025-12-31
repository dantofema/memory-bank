# Project Definition - E-commerce Básico

## Problema
Los pequeños comercios B2C necesitan una plataforma sencilla para exponer sus productos y recibir pedidos sin la complejidad de sistemas de inventario o pasarelas de pago integradas, permitiendo una gestión humana y directa del cierre de ventas.

## Objetivos
- Proveer un catálogo público de productos filtrable.
- Facilitar la creación de pedidos con datos de contacto mínimos.
- Notificar al administrador de forma automática vía WhatsApp sobre nuevos pedidos.
- Permitir al administrador gestionar el catálogo y los estados de pago/entrega de forma manual.

## Scope (Alcance)
### MVP
- Catálogo de productos con categorías.
- Carrito de compras y proceso de checkout (sin pago).
- Panel de administración para gestión de productos, categorías y pedidos.
- Dashboard básico para el administrador.
- Notificaciones asíncronas vía WhatsApp al admin.

### Fuera de Alcance
- Gestión de stock/inventario.
- Integración de pasarelas de pago.
- Cálculo o gestión de envíos automatizados.
- Integración oficial de WhatsApp Business.
- Registro de usuarios (customers).

## Roles
- **Customer (Anónimo):** Puede ver productos, filtrar por categoría, armar carrito y realizar pedidos.
- **Admin:** Gestiona catálogo, visualiza y actualiza pedidos, y monitorea el dashboard.

## Casos de Uso Principales
- **UC01: Realizar Pedido:** El customer selecciona productos, carga datos de contacto (dirección, localidad, teléfono) y confirma el pedido.
- **UC02: Notificar Pedido:** El sistema envía un WhatsApp al admin con el detalle del pedido y link al panel.
- **UC03: Gestionar Catálogo:** El admin crea, edita o elimina productos y categorías.
- **UC04: Actualizar Estado de Pedido:** El admin cambia el estado de entrega (Pendiente, Entregado, Cancelado) o el de pago (Sin Pagar, Pagado) tras la coordinación manual.

## Requerimientos
- **Funcionales:** Filtrado por categorías, carrito persistente (sesión), notificación asíncrona (Queue/Job).
- **No Funcionales:** Configuración del número de admin vía ENV, uso de API de terceros para WhatsApp, escalabilidad básica para el MVP.

## Dominio (Entidades)
- **Product:** nombre, descripción, precio, imagen, categoría.
- **Category:** nombre, descripción.
- **Order:** número, total, datos_cliente (nombre, dirección, localidad, teléfono), estado_pedido, estado_pago.
- **OrderItem:** producto, cantidad, precio_unitario, subtotal.

## Riesgos
- **Falla en Notificación:** Si la API de WhatsApp falla, el admin podría no enterarse inmediatamente (mitigado por persistencia del pedido).
- **Coordinación Manual:** Dependencia del contacto externo por WhatsApp para el cierre real de la venta.
- **Seguridad del Panel Admin:** Necesidad de proteger el acceso a los datos de los pedidos.

---
*Si no entra en 1 vista -> volver atrás.*
