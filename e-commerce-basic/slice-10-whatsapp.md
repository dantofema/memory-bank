# Slice — Envío de pedido por WhatsApp (Customer → Admin)

## 1. Contexto

Cuando un customer **no autenticado** confirma una orden (checkout como invitado), el administrador debe ser notificado por WhatsApp y poder continuar la conversación directamente con ese customer.

Restricción clave del dominio:

* WhatsApp **solo permite conversaciones libres si el mensaje es enviado por un usuario real**.

Por lo tanto, el sistema **no envía el mensaje automáticamente**, sino que **prepara el mensaje y el link** para que el customer lo envíe desde su propio WhatsApp.

No existe sesión autenticada ni usuario persistido para el customer.

---

## 2. Objetivo del slice

Permitir que, al confirmar una orden:

* el sistema genere un mensaje con los datos del pedido
* el customer lo envíe por WhatsApp al administrador
* el admin reciba el mensaje y continúe la conversación

Valor entregado:

> El admin recibe pedidos por WhatsApp con contexto completo y puede responder sin fricción.

---

## 3. Vertical Slice Definition

**Slice name:** Enviar pedido por WhatsApp (guest checkout)

**Actor principal:** Customer no autenticado

**Actor secundario:** Administrador

**Inicio:** Orden creada correctamente con datos del customer

**Fin:** Mensaje enviado por WhatsApp desde el teléfono del customer

---

## 4. Expected Behavior (Given / When / Then)

### Caso feliz

- Given un customer no autenticado
- And completa un formulario de pedido válido
- When el sistema muestra la confirmación del pedido
- Then se muestra un botón "Enviar pedido por WhatsApp"


- When el customer hace click en el botón
- Then se abre WhatsApp con un mensaje prellenado
- And el mensaje contiene datos del pedido y datos de contacto del customer

---

### Errores / validaciones

- Given una orden inexistente
- Then se retorna 404


- Given la creación de la orden falla
- Then no se muestra la opción de WhatsApp

---

## 5. Contenido del mensaje

Formato sugerido:

```
Hola, acabo de realizar un pedido.


Nombre: {customer_name}
Dirección: {customer_address}
Localidad: {customer_city}
Teléfono: {customer_phone}

Productos:
- {product_1_name} x {quantity} = {subtotal}
- {product_2_name} x {quantity} = {subtotal}
... (lista completa)

Total: {order_total}
Pedido: #{order_id}

Gracias.
```

Notas:

* Texto plano
* Datos provienen del formulario de pedido
* Sin links firmados
* Sin datos sensibles (tokens, información de pago)

---

## 6. Tech Notes (Just in Time)

### Configuración

`.env`

```
WHATSAPP_ADMIN_PHONE=54911XXXXXXXX
```

`config/services.php`

```
'whatsapp' => [
    'admin_phone' => env('WHATSAPP_ADMIN_PHONE'),
],
```

---

### Servicio de dominio

Responsabilidad:

* Construir el mensaje
* Generar el link `wa.me`

Contrato implícito:

* Input: Order válida
* Output: URL externa a WhatsApp

---

### UI / Livewire

* Mostrar botón solo cuando se solicita confirmar la orden

---

## 7. Seguridad

* El número de WhatsApp del admin:

    * nunca hardcodeado
    * siempre desde `.env`

* Validaciones obligatorias:

    * campos requeridos del customer: nombre, dirección, localidad, teléfono
    * teléfono normalizado (solo números)

* La URL de WhatsApp:

    * se genera **solo después** de persistir la orden

* Logs:

    * no loguear contenido completo del mensaje
    * no loguear teléfonos

---

## 8. No incluido en este slice

* WhatsApp Business API
* Envío automático desde backend
* Templates aprobados por Meta
* Historial de conversaciones

Estos puntos quedan fuera por costo y complejidad.

---

## 9. Criterio de finalización

El slice se considera terminado cuando:

* la orden se crea correctamente
* el customer puede enviar el pedido por WhatsApp
* el admin recibe el mensaje con contexto suficiente
* el flujo puede demo-earse end-to-end

---

## 10. Regla de oro

> El sistema arma el contexto.
> El customer inicia la conversación.
> WhatsApp hace el resto.
