Perfecto. Acá va el **Slice 2 adaptado correctamente a `wa.me`**, manteniendo el enfoque lean, sin APIs externas y con límites claros.

---

# Slice 2 — Notificación WhatsApp (wa.me)

## 1. Objetivo

El administrador recibe una **notificación accionable vía WhatsApp** cada vez que se confirma un nuevo pedido, mediante un **mensaje con link `wa.me`** que permite abrir el pedido directamente en el panel admin desde el celular.

---

## 2. Actores

* **Actor principal:** Sistema (proceso automático)
* **Actor secundario:** Administrador (receptor del mensaje)

---

## 3. Precondiciones

* Slice 1 completado: el sistema crea pedidos (`Order`).
* El pedido posee una **referencia pública no secuencial** (`public_reference`).
* El número de WhatsApp del administrador está configurado en `.env`.
* El panel admin es accesible desde mobile y requiere autenticación.

---

## 4. Flujo principal (Happy Path)

**Given** que un Customer (autenticado o guest) finaliza un pedido
**When** el sistema persiste el pedido
**Then** se dispara un evento `OrderCreated`

**When** el listener asíncrono procesa el evento
**Then** construye un mensaje de texto con:

* referencia pública del pedido
* total
* link directo al pedido en el admin

**And** genera un link `https://wa.me/...` con el mensaje precompletado
**And** deja el link disponible para el administrador (visualización / apertura manual)

---

## 5. Flujos alternativos / errores relevantes

* **Given** que el número de WhatsApp no está configurado
  **Then** el listener registra error en logs y finaliza sin afectar el pedido.

* **Given** que el link al admin no puede generarse
  **Then** el mensaje se genera sin link y se registra advertencia.

* **Given** que el Job falla
  **Then** Laravel reintenta según política configurada.

---

## 6. Reglas de negocio

* La notificación es **asíncrona** (Queue / Job).
* El mensaje es **solo texto plano**, sin envío automático.
* El mensaje debe incluir:

  * Referencia pública del pedido
  * Total
  * Link al panel admin
* **No se usa API de WhatsApp ni terceros**.
* WhatsApp es un **canal pasivo**: el admin decide abrir el link.

---

## 7. Estados involucrados

* `Order`: no cambia de estado por la notificación.
* La notificación no es parte del lifecycle del pedido.

---

## 8. Validaciones clave

* El número de WhatsApp debe estar en formato internacional (`549...`).
* `public_reference` debe ser:

  * no secuencial
  * no guessable
* El link admin debe usar rutas nombradas.

---

## 9. Autorización y seguridad

* El flujo solo lo ejecuta el sistema.
* El link al pedido:

  * requiere autenticación admin
  * no expone IDs internos
* El mensaje **no incluye datos sensibles** (email, dirección completa, DNI).
* El número de WhatsApp se gestiona por `.env`, no hardcodeado.

---

## 10. Límites del slice (out of scope)

* No incluye respuesta desde WhatsApp.
* No incluye bots ni automatización.
* No incluye WhatsApp Business API.
* No incluye notificaciones al cliente.

---

## 11. Decisiones técnicas mínimas (Just in Time)

* **Disparo:** `Event` (`OrderCreated`)
* **Ejecución:** `Listener` que implementa `ShouldQueue`
* **Infra:** Laravel Queues (`database` o `redis`)
* **Configuración:**

  * `WHATSAPP_ADMIN_PHONE` en `.env`
* **Formato:**

  * mensaje de texto
  * link `wa.me` con `urlencode()`

---

## 12. Criterio de finalización

* Al crear un pedido:

  * el evento se dispara
  * el listener se encola
* El mensaje generado contiene:

  * referencia correcta
  * total correcto
  * link válido al admin
* El admin puede:

  * abrir WhatsApp
  * tocar el link
  * ver el pedido en mobile

---

### Nota clave de arquitectura

> WhatsApp **no es un sistema de entrega**, es solo un **transportador de contexto**.
> El sistema sigue siendo el backend.

