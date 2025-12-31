# Slice 10 — Confirmación y Envío por WhatsApp (Checkout)

## 1. Contexto

Cuando un customer confirma una orden en el checkout, el sistema debe persistir el pedido en la base de datos y, simultáneamente, facilitar que el customer envíe el detalle por WhatsApp al administrador para cerrar la venta.

Restricción clave del dominio:
* El sistema **no envía el mensaje automáticamente** (debido a inestabilidad de APIs externas y costos), sino que **prepara el mensaje y el link** para que el customer lo envíe desde su propio WhatsApp al hacer clic en "Confirmar".

---

## 2. Objetivo del slice

Integrar la acción de "Confirmar Pedido" con la apertura de WhatsApp:
1. Validar y guardar la orden en la BD.
2. Generar un mensaje detallado con los productos y datos de contacto.
3. Abrir WhatsApp en una nueva pestaña con el mensaje prellenado.
4. Redirigir la ventana principal a la página de éxito (`order-success`).

---

## 3. Vertical Slice Definition

**Slice name:** Confirmar y enviar por WhatsApp (Checkout)
**Actor principal:** Customer (Anónimo)
**Actor secundario:** Administrador
**Inicio:** Clic en el botón "Confirmar Pedido" en el formulario de Checkout.
**Fin:** Orden persistida, WhatsApp abierto con mensaje y redirección a página de éxito.

---

## 4. Expected Behavior (Given / When / Then)

### Caso feliz
- **Given** un customer con productos en el carrito.
- **And** completa el formulario de checkout con datos válidos.
- **When** hace clic en el botón de confirmación.
- **Then** se crea el registro en las tablas `orders` y `order_items`.
- **And** se limpia el carrito de compras.
- **And** se abre una nueva pestaña con la URL `wa.me` prellenada.
- **And** la pestaña original se redirige a `/order-success/{order_number}`.

---

## 5. Contenido del mensaje

Formato requerido:
```text
Hola, acabo de realizar un pedido:

Pedido: #{order_number}
Cliente: {customer_name}
Dirección: {customer_address}, {customer_city}
Teléfono: {customer_phone}

Productos:
- {product_name} x {quantity} (${subtotal})
...

Total: ${order_total}

Gracias.
```

---

## 6. Tech Notes

### Configuración de Destino
El número de destino se obtiene en este orden de prioridad:
1. `settings('contact_whatsapp')` (Gestionado en Slice 6).
2. `config('services.whatsapp.admin_phone')` (Fallback del `.env`).

### Implementación en `checkout.blade.php`
La lógica reside en la función `placeOrder` del componente Volt:
- Se debe construir el string del mensaje iterando sobre `$this->cartItems`.
- Se debe usar `urlencode()` para el contenido del mensaje.
- El componente debe emitir un evento o devolver una instrucción para que el navegador abra la ventana de WhatsApp antes o durante la redirección.

---

## 7. Criterio de finalización

* La orden se guarda correctamente en la base de datos.
* El carrito queda vacío tras la acción.
* Se dispara la apertura de WhatsApp con el mensaje correcto.
* El usuario llega a la página de éxito sin errores.
* Se desactiva el envío automático (CallMeBot) si causaba conflictos.
