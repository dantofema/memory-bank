# Slice 1 — Carrito y Checkout

## 1. Objetivo
El cliente puede seleccionar productos del catálogo, gestionar su carrito de compras y enviar un pedido proporcionando sus datos de contacto básicos.

---

## 2. Actores
- Actor principal: Customer (Anónimo)
- Actores secundarios: N/A

---

## 3. Precondiciones
- Slice 0 completado: El catálogo de productos es funcional y visible.
- El sistema tiene una sesión activa para manejar la persistencia del carrito.

---

## 4. Flujo principal (Happy Path)
Given que el Customer está navegando el catálogo
When selecciona "Agregar al carrito" en uno o varios productos
Then los productos se añaden al carrito y se actualiza el total.

When el Customer accede a la vista del Carrito
Then puede visualizar los productos seleccionados, ajustar cantidades o eliminar ítems.

When el Customer procede al Checkout
Then completa un formulario con: Nombre, Dirección, Localidad y Teléfono.

When el Customer confirma el pedido
Then el sistema crea el Pedido (Order) y sus detalles (OrderItems), vacía el carrito y muestra una pantalla de éxito con el número de pedido.

---

## 5. Flujos alternativos / errores relevantes
- Given que el carrito está vacío
  Then la opción de "Proceder al Checkout" está deshabilitada o redirige al catálogo con un mensaje informativo.

- Given que el Customer intenta agregar una cantidad inválida (ej: 0 o negativa)
  Then el sistema ignora la acción o muestra un mensaje de error de validación.

- Given que faltan datos obligatorios en el Checkout
  Then el sistema resalta los campos faltantes y no permite enviar el pedido.

---

## 6. Reglas de negocio
- El carrito es persistente por sesión (no requiere login).
- Los precios de los productos en el pedido se congelan al momento de la compra (copiados a `OrderItem`).
- Un pedido se crea inicialmente con estado de pago "Sin Pagar" y estado de entrega "Pendiente".
- No se realiza validación de stock (según scope).

---

## 7. Estados involucrados (si aplica)
- Order:
    - Estado Pago: Sin Pagar, Pagado.
    - Estado Pedido: Pendiente, Entregado, Cancelado.

---

## 8. Validaciones clave
- Campos obligatorios en Checkout: Nombre, Dirección, Localidad, Teléfono.
- Formato de teléfono: Validación básica de caracteres numéricos.
- Cantidad de ítems: Mínimo 1 por producto seleccionado.

---

## 9. Autorización y seguridad
- Acceso: Público y anónimo.
- Seguridad: Los datos del cliente solo se utilizan para la gestión del pedido.
- Integridad: El total del pedido debe recalcularse en el servidor, no confiando solo en el total enviado por el cliente.

---

## 10. Límites del slice (out of scope)
- No incluye integración con pasarelas de pago (Slice 1 es manual).
- No incluye notificaciones WhatsApp (Slice 2).
- No incluye registro o perfil de usuario (Customer es siempre anónimo).
- No incluye cálculo automatizado de costos de envío.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Entidades nuevas: 
    - `Order` (id, number, total, customer_name, customer_address, customer_city, customer_phone, order_status, payment_status).
    - `OrderItem` (id, order_id, product_id, quantity, unit_price, subtotal).
- Relaciones: `Order` hasMany `OrderItem`, `OrderItem` belongsTo `Product`.
- Generación de número de pedido: Formato simple (ej: ORD-YYYYMMDD-ID).

---

## 12. Criterio de finalización
- Se puede completar el flujo desde agregar un producto hasta ver la pantalla de éxito del pedido.
- El pedido y sus ítems quedan correctamente registrados en la base de datos.
- El carrito se limpia tras una compra exitosa.
- Cumple con las validaciones de datos del cliente.
