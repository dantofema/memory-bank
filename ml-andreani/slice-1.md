# Slice 1 — Core Value: Compra simple end-to-end

> Formato optimizado para ejecución por AI / agente técnico  
> Stack objetivo: Laravel 12 · PostgreSQL · Redis · Livewire / Volt · FilamentPHP - Sail

---

## Objetivo del Slice

Implementar el **flujo de compra mínimo viable**: usuario compra un producto único y recibe confirmación.
Este slice **demuestra que el negocio funciona** y reduce el mayor riesgo técnico temprano.

---

## Definition of Done (global)

- Usuario puede ver productos disponibles
- Usuario puede agregar producto al carrito
- Usuario puede completar checkout sin envío
- Pago exitoso genera orden persistida
- Sistema muestra confirmación clara
- Todo el flujo testeable end-to-end

---

## Tareas técnicas

### 1.1 Modelo de Producto

**Descripción**
Crear entidad Producto con atributos mínimos para venta.

**Tareas**

- Crear migración `products` (id, name, description, price, stock, active, timestamps)
- Crear modelo `Product` con validación
- Definir VO y Casts para price (decimal) y stock (integer)
- Definir Factory para tests
- Implementar soft deletes
- Índices en `active` y `stock`

**DoD**

- Productos creables vía Factory
- Validación server-side (price > 0, stock >= 0)
- Tests unitarios pasando

**Fuera de scope**

- Variantes de producto
- Imágenes múltiples
- Categorías complejas

**Seguridad**

- Validar precio y stock numéricamente
- Sanitizar inputs de texto

---

### 1.2 Listado de productos

**Descripción**
Mostrar productos disponibles en un catálogo simple.

**Tareas**

- Crear ruta `/products`
- Crear componente Livewire/Volt para listado
- Mostrar: nombre, precio, stock disponible
- Paginación básica (10 items)
- Filtrar solo productos activos con stock > 0

**DoD**

- Listado renderiza correctamente
- Productos sin stock no se muestran
- Performance aceptable (N+1 controlado)

**Fuera de scope**

- Búsqueda avanzada
- Filtros por categoría
- Ordenamiento complejo

**Seguridad**

- No exponer productos inactivos
- XSS prevenido en descripciones

---

### 1.3 Modelo de Carrito (session-based)

**Descripción**
Implementar carrito de compras temporal en sesión.

**Tareas**

- Crear servicio `CartService`
- Métodos: `add()`, `remove()`, `clear()`, `total()`
- Persistir en sesión Laravel
- Validar stock al agregar
- Calcular subtotal automáticamente

**DoD**

- Carrito persiste entre requests
- Stock validado en cada operación
- Tests de servicio completos

**Fuera de scope**

- Carrito persistido en DB
- Carritos recuperables
- Cupones de descuento

**Seguridad**

- Validar que producto existe
- Validar stock antes de agregar
- Prevenir manipulación de precios

---

### 1.4 UI del Carrito

**Descripción**
Mostrar contenido del carrito con opciones básicas.

**Tareas**

- Crear componente Livewire para carrito
- Mostrar: producto, cantidad, precio unitario, subtotal
- Botón eliminar item
- Botón vaciar carrito
- Mostrar total general

**DoD**

- Carrito reactivo (actualización sin reload)
- Feedback visual en acciones
- Mobile responsive básico

**Fuera de scope**

- Editar cantidad inline
- Guardar para después
- Wishlist

---

### 1.5 Modelo de Orden

**Descripción**
Entidad que persiste la compra confirmada.

**Tareas**

- Crear migración `orders` (id, user_id nullable, status, total, payment_status, timestamps)
- Crear migración `order_items` (id, order_id, product_id, quantity, price, subtotal)
- Modelos con relaciones correctas
- Enum para estados (`pending`, `paid`, `cancelled`)
- Factory para tests

**DoD**

- Relaciones funcionando (Order hasMany OrderItems)
- Estados validados vía Enum
- Integridad referencial (FK)

**Fuera de scope**

- Estados complejos de envío
- Historial de cambios
- Facturación

**Seguridad**

- Validar que total coincide con items
- user_id nullable permite guest checkout

---

### 1.6 Checkout sin envío

**Descripción**
Pantalla de checkout que genera orden sin costo de envío.

**Tareas**

- Crear ruta `/checkout`
- Componente Livewire para checkout
- Mostrar resumen del carrito
- Input básico: email (si guest), nombre, teléfono
- Botón "Confirmar compra"
- Validar stock antes de crear orden

**DoD**

- Formulario valida correctamente
- Stock verificado en momento de compra
- Orden creada con datos correctos
- Carrito se limpia post-compra

**Fuera de scope**

- Cálculo de envío
- Cupones
- Múltiples direcciones

**Seguridad**

- Validación server-side de todos los campos
- Re-validar stock antes de confirmar
- CSRF protection activo

---

### 1.7 Integración de pago (happy path)

**Descripción**
Integrar pasarela de pago local (ej: Mercado Pago) solo flujo exitoso.

**Tareas**

- Configurar credenciales Mercado Pago en `.env`
- Crear servicio `PaymentService`
- Generar preferencia de pago
- Redireccionar a checkout de MP
- Webhook para confirmar pago exitoso
- Actualizar `payment_status` de orden

**DoD**

- Pago exitoso actualiza orden
- Webhook validado correctamente
- Tests con mocks de MP

**Fuera de scope**

- Manejo de pagos rechazados
- Pagos en cuotas
- Múltiples métodos de pago
- Reembolsos

**Seguridad**

- Webhook firmado y verificado
- Secrets de MP fuera del repo
- Validar que monto coincide
- Logs no exponen tokens

---

### 1.8 Confirmación de orden

**Descripción**
Pantalla que muestra orden exitosa al usuario.

**Tareas**

- Crear ruta `/orders/{order}/confirmation`
- Mostrar: número de orden, items, total, estado
- Mensaje claro de confirmación
- Link para volver al catálogo

**DoD**

- Orden accesible solo por quien la creó
- Datos claros y completos
- No exponer órdenes de otros

**Fuera de scope**

- Descarga de factura
- Tracking de envío
- Cancelación de orden

**Seguridad**

- Validar ownership de la orden
- No permitir acceso no autorizado

---

### 1.9 Email transaccional básico

**Descripción**
Enviar confirmación de compra por email.

**Tareas**

- Crear Mailable `OrderConfirmed`
- Template simple con: número de orden, items, total
- Enviar post-pago exitoso
- Queue para envío asíncrono

**DoD**

- Email se envía correctamente
- Cola configurada
- Template en español, claro

**Fuera de scope**

- Templates complejos con branding
- Emails de seguimiento
- Notificaciones SMS

**Seguridad**

- No exponer info sensible en email
- Rate limiting en envíos

---

### 1.10 Tests end-to-end del flujo completo

**Descripción**
Test automatizado del happy path completo.

**Tareas**

- Test PEST: visitar listado → agregar al carrito → checkout → pago mock → orden creada
- Verificar stock descontado
- Verificar email enviado
- Verificar orden persistida correctamente

**DoD**

- Test pasa en pipeline
- Cobertura > 80% del slice
- Sin falsos positivos

---

### 1.11 Descuento de stock post-compra

**Descripción**
Actualizar stock disponible al confirmar pago.

**Tareas**

- Listener que escucha evento `OrderPaid`
- Descontar stock de cada producto en la orden
- Validar que stock no quede negativo
- Rollback si hay inconsistencia

**DoD**

- Stock se actualiza correctamente
- Manejo de concurrencia básico (locks)
- Tests de integridad

**Fuera de scope**

- Stock reservado al agregar al carrito
- Notificaciones de bajo stock

**Seguridad**

- Transacciones DB para integridad
- Logs de cambios de stock

---

### 1.12 Métricas básicas

**Descripción**
Instrumentar puntos clave para monitoreo.

**Tareas**

- Log de tiempo entre agregar al carrito y orden creada
- Log de errores de pago
- Counter simple de órdenes creadas

**DoD**

- Métricas logueadas correctamente
- Información suficiente para debug

**Fuera de scope**

- Dashboard completo
- Alertas automáticas
- Analytics avanzado

---

## Restricciones

- No implementar envío (eso es Slice 3)
- No implementar usuarios registrados (eso es Slice 2)
- No manejar errores de pago (solo happy path)
- No optimizaciones de performance complejas

---

## Regla de corte

Si una tarea:

- No es parte del flujo crítico de compra
- Introduce complejidad prematura
- Depende de slices futuros

→ **queda fuera del Slice 1**

---

## Nota de seguridad (obligatoria)

- Validación server-side de precio y stock en cada paso
- Webhook de pago firmado y verificado
- Secrets de pasarela fuera del repo
- No exponer órdenes de otros usuarios
- CSRF protection en todos los formularios
- XSS prevenido en campos de texto
- Logs sin datos sensibles (tarjetas, tokens)

---

## Orden de ejecución sugerido

1. Modelo Producto (1.1)
2. Listado productos (1.2)
3. Modelo Carrito (1.3)
4. UI Carrito (1.4)
5. Modelo Orden (1.5)
6. Checkout (1.6)
7. Descuento stock (1.11)
8. Integración pago (1.7)
9. Confirmación (1.8)
10. Email (1.9)
11. Tests e2e (1.10)
12. Métricas (1.12)

---

## Comandos útiles (vía Sail)

```bash
# Crear migraciones
./vendor/bin/sail artisan make:migration create_products_table
./vendor/bin/sail artisan make:migration create_orders_table

# Correr migraciones
./vendor/bin/sail artisan migrate

# Crear modelos
./vendor/bin/sail artisan make:model Product
./vendor/bin/sail artisan make:model Order

# Crear componentes Livewire
./vendor/bin/sail artisan make:livewire ProductList
./vendor/bin/sail artisan make:livewire Cart
./vendor/bin/sail artisan make:livewire Checkout

# Tests
./vendor/bin/sail test
./vendor/bin/sail test --coverage

# Análisis estático
./vendor/bin/sail composer pint
./vendor/bin/sail composer rector
./vendor/bin/sail artisan vendor:publish --tag=larastan
```

---

## Valor entregado

Al finalizar este slice:

- **Usuario puede comprar** → validación de negocio
- **Riesgo técnico reducido** → integración de pago funcionando
- **Base para slices futuros** → estructura de producto/orden lista
- **Deploy posible** → flujo completo testeable

---

## Próximo slice recomendado

**Slice 2 — Usuarios reales** para asociar órdenes a cuentas persistentes y habilitar historial de compras.
