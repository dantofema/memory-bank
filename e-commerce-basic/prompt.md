# E-commerce Básico - Definición de Proyecto

## Rol

Actuá como Arquitecto de Software senior, con enfoque en diseño de productos digitales y definición de alcance para MVPs.

## Objetivo

Generar en esta carpeta el artefacto `project-definition.md` para un proyecto de tienda online, claro, conciso y apto para iniciar desarrollo sin ambigüedades. Utilizando como guia el archivo `proceso-de-desarrollo-lean-vsf.md`.

## Contexto Funcional del Sistema

El sistema es una tienda online B2C.

### Usuarios (Customers)

Los usuarios (customers) pueden:

- Ver productos públicos
- Filtrar productos por categorías
- Agregar productos a un carrito
- Realizar un pedido

Para completar un pedido, el customer debe cargar:

- Dirección de entrega
- Localidad
- Teléfono

### Administrador (Admin)

El administrador (admin) puede:

- Gestionar productos (CRUD)
- Gestionar categorías
- Visualizar todos los pedidos
- Actualizar el estado de los pedidos
- Visualizar un Dashboard con información básica

### Pedidos

Los pedidos tienen dos dimensiones de estado independientes:

#### Estado del Pedido
- **Pendiente**: recién creado, sin coordinar
- **Entregado**: entregado al customer
- **Cancelado**: cancelado por admin o customer

#### Estado de Pago
- **Sin Pagar**: el customer aún no pagó
- **Pagado**: el customer ya pagó (se coordina manualmente)

**Importante**: El pago y la entrega son independientes. Un pedido puede estar "Entregado" pero "Sin Pagar", o "Pendiente" pero "Pagado".

#### Coordinación Manual

La entrega y la forma de pago se coordinan manualmente:

- El admin contacta al customer por WhatsApp personal (sin WhatsApp Business ni integraciones)
- El acuerdo de entrega y pago ocurre fuera del sistema
- El admin refleja el resultado actualizando ambos estados del pedido

#### Notificaciones

**Notificación automática al crear pedido:**

- Cuando un customer crea un pedido, el sistema envía automáticamente un mensaje de WhatsApp al admin
- El mensaje debe contener información clave del pedido:
  - Número de pedido
  - Descripción de los productos y cantidades
  - Nombre del customer
  - Total del pedido
  - Link directo al detalle del pedido en el panel admin

**Implementación:**

- Usar API de WhatsApp (ej: Twilio, Ultramsg, o similar)
- El número de WhatsApp del admin se configura en variables de entorno
- La notificación se envía de forma asíncrona (cola/job) para no bloquear la creación del pedido
- Si falla el envío, el pedido se crea igual (la notificación no es crítica)


## Restricciones Explícitas

- No se maneja stock
- No hay pagos integrados
- No hay envíos automatizados
- No hay integración de WhatsApp Business (se usa API de terceros para notificaciones)