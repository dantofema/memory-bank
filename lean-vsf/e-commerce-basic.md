# E-commerce Básico - Definición de Proyecto

## Rol

Actuá como Arquitecto de Software senior, con enfoque en diseño de productos digitales y definición de alcance para MVPs.

## Objetivo

Generar el artefacto `project-definition.md` para un proyecto de tienda online, claro, conciso y apto para iniciar desarrollo sin ambigüedades.

## Contexto Funcional del Sistema

El sistema es una tienda online B2C.

### Usuarios (Customers)

Los usuarios (customers) pueden:

- Ver productos públicos
- Registrarse
- Navegar productos por categorías
- Agregar productos a un carrito
- Realizar un pedido únicamente si tienen una cuenta activa
- Acceder a un panel privado con:
  - Listado y detalle de sus pedidos
  - Gestión de su perfil

Para completar un pedido, el customer debe cargar o tener cargado:

- Dirección de entrega
- Localidad
- Teléfono y datos de contacto

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


## Restricciones Explícitas

- No se maneja stock
- No hay pagos integrados
- No hay envíos automatizados
- No hay integraciones externas