# Slice 3 — Logística Argentina: Andreani

> Formato optimizado para ejecución por AI / agente técnico  
> Stack objetivo: Laravel 12 · PostgreSQL · Redis · Livewire / Volt · FilamentPHP - Sail

---

## Objetivo del Slice

Implementar **integración completa con Andreani**: cotización de envíos en tiempo real, selección de método (domicilio/sucursal) y generación de etiquetas.
Este slice **agrega el diferencial logístico local** y completa el flujo de compra real en Argentina.

---

## Definition of Done (global)

- Usuario puede cotizar costo de envío con su código postal
- Usuario puede elegir entre envío a domicilio o retiro en sucursal
- Sistema genera etiqueta de envío Andreani post-pago
- Costos de envío se suman correctamente al total
- Tracking code guardado en la orden
- Todo el flujo testeable con mocks de Andreani API

---

## Tareas técnicas

### 3.1 Configuración de Andreani API

**Descripción**
Preparar credenciales y cliente HTTP para comunicación con Andreani.

**Tareas**

- Agregar credenciales Andreani a `.env`: `ANDREANI_USERNAME`, `ANDREANI_PASSWORD`, `ANDREANI_CONTRACT`
- Crear config file `config/andreani.php`
- Implementar `AndreaniClient` service con Guzzle
- Métodos base: `authenticate()`, `getToken()`, `refreshToken()`
- Manejo de expiración de token (cache en Redis)
- Configurar endpoint sandbox vs producción

**DoD**

- Cliente HTTP funcional y autenticado
- Token cacheado correctamente (60 min)
- Retry automático en falla de autenticación
- Tests con mock de API

**Fuera de scope**

- Múltiples contratos Andreani
- Fallback a otros carriers
- Pool de conexiones

**Seguridad**

- Credenciales solo en `.env`
- Token nunca logueado
- HTTPS obligatorio para API calls
- Rate limiting interno

---

### 3.2 Modelo de Dirección

**Descripción**
Entidad para almacenar direcciones de envío.

**Tareas**

- Crear migración `addresses` (id, user_id nullable, street, number, floor, apartment, city, state, postal_code, alias, is_default, timestamps)
- Crear modelo `Address` con validaciones
- Relaciones: User hasMany Addresses
- Factory para tests con datos argentinos
- Índices en `user_id` y `postal_code`
- Soft deletes

**DoD**

- Direcciones persistibles correctamente
- Validación de código postal argentino (4 dígitos)
- Relaciones funcionando
- Factory genera direcciones válidas

**Fuera de scope**

- Geolocalización automática
- Validación con API de ARCA
- Normalización de direcciones
- Múltiples países

**Seguridad**

- Validar ownership de direcciones
- No exponer direcciones de otros usuarios
- Sanitizar inputs de texto

---

### 3.3 CRUD de direcciones en perfil

**Descripción**
Interfaz para que usuario gestione sus direcciones.

**Tareas**

- Crear ruta `/profile/addresses`
- Componente Livewire para listar direcciones
- Formulario inline para agregar/editar
- Validación de campos obligatorios
- Marcar dirección como predeterminada
- Eliminar dirección (soft delete)
- Límite de 5 direcciones por usuario

**DoD**

- Usuario puede CRUD sus direcciones
- Dirección predeterminada destacada
- Validación en español clara
- UX simple y responsive

**Fuera de scope**

- Importar direcciones desde archivo
- Compartir direcciones entre usuarios
- Historial de cambios

**Seguridad**

- Policy para verificar ownership
- Validación server-side estricta
- XSS prevenido en campos texto
- Rate limiting en creación

---

### 3.4 Servicio de cotización Andreani

**Descripción**
Integrar endpoint de cotización de tarifas.

**Tareas**

- Crear service `AndreaniShippingService`
- Método `quoteDomicilio(postalCode, weight, volume)`
- Método `quoteSucursal(postalCode, weight, volume)`
- Método `getSucursales(city, postalCode)`
- Parsear respuesta JSON de Andreani
- Manejo de errores (zona no cubierta, peso excedido)
- Cache de cotizaciones (15 min por CP + peso)
- Log de requests para debug

**DoD**

- Cotizaciones retornan precio correcto
- Errores manejados claramente
- Performance aceptable (<2s)
- Tests unitarios con mocks

**Fuera de scope**

- Múltiples opciones de velocidad
- Seguro de envío opcional
- Retiro en correo

**Seguridad**

- Validar inputs antes de API call
- No exponer credenciales en logs
- Rate limiting (max 10/min por sesión)
- Timeout en requests (5s)

---

### 3.5 Cálculo de peso y volumen de orden

**Descripción**
Determinar peso/volumen total para cotización.

**Tareas**

- Agregar campos a `products`: `weight_grams`, `length_cm`, `width_cm`, `height_cm`
- Migración para campos nuevos
- Método `Order::getTotalWeight()`
- Método `Order::getTotalVolume()`
- Valores por defecto sensatos (ej: 500g, 20x20x10cm)
- Validación de valores positivos

**DoD**

- Peso y volumen calculables por orden
- Defaults aplicados si producto sin datos
- Cálculos correctos en tests
- UI en admin para configurar por producto

**Fuera de scope**

- Cálculo volumétrico complejo
- Peso variable por variante
- Optimización de empaque

**Seguridad**

- Validar que valores sean positivos
- No permitir manipulación cliente-side

---

### 3.6 Integración de cotización en checkout

**Descripción**
Mostrar opciones de envío durante checkout.

**Tareas**

- Agregar paso "Envío" antes de pago en checkout
- Input de código postal (si no tiene dirección guardada)
- Selector de dirección guardada (si usuario logueado)
- Llamada a `AndreaniShippingService` al ingresar CP
- Mostrar opciones: Domicilio ($X) | Sucursal ($Y)
- Cargar sucursales cercanas si elige sucursal
- Validar selección antes de continuar a pago
- Mostrar total actualizado (productos + envío)

**DoD**

- Cotización aparece en <3 segundos
- Opciones claras con precios
- Total se actualiza reactivamente
- Validación completa antes de pago

**Fuera de scope**

- Envío gratis por monto mínimo (slice futuro)
- Múltiples paquetes
- Selección de horario de entrega

**Seguridad**

- Validar CP server-side
- No confiar en precio del cliente
- Re-cotizar server-side antes de orden

---

### 3.7 Modelo de Envío (Shipment)

**Descripción**
Entidad para persistir datos de envío.

**Tareas**

- Crear migración `shipments` (id, order_id, carrier='andreani', method enum(domicilio, sucursal), cost, tracking_number, address_id nullable, sucursal_id nullable, status enum, estimated_delivery_date, timestamps)
- Crear modelo `Shipment` con relaciones
- Order hasOne Shipment
- Shipment belongsTo Address (si domicilio)
- Factory para tests
- Índices en `order_id` y `tracking_number`

**DoD**

- Envíos persistibles correctamente
- Relaciones OK con Order y Address
- Status manejados con Enum
- Tests de modelo pasando

**Fuera de scope**

- Múltiples carriers
- Envíos parciales
- Devoluciones

**Seguridad**

- Validar que shipment pertenece a orden del usuario
- No exponer tracking de otros

---

### 3.8 Generación de etiqueta Andreani

**Descripción**
Crear orden de envío en Andreani post-pago exitoso.

**Tareas**

- Crear service `AndreaniLabelService`
- Método `createShipment(order, shipmentData)`
- Enviar datos de paquete (peso, dimensiones, destino)
- Recibir tracking number de Andreani
- Descargar etiqueta PDF desde Andreani
- Guardar PDF en storage (`storage/app/labels/{order_id}.pdf`)
- Actualizar Shipment con tracking_number
- Trigger vía evento `OrderPaid`

**DoD**

- Etiqueta se genera automáticamente post-pago
- PDF guardado correctamente
- Tracking number en DB
- Error manejado sin romper flujo de pago

**Fuera de scope**

- Impresión automática
- Re-generación manual de etiqueta
- Cancelación de envío

**Seguridad**

- Validar que orden está paga
- Storage no público directo
- Solo admin/usuario pueden descargar etiqueta
- Validar formato de respuesta API

---

### 3.9 Visualización de datos de envío

**Descripción**
Mostrar información de envío en confirmación y detalle de orden.

**Tareas**

- Agregar sección "Envío" en confirmación de orden
- Mostrar: método, costo, dirección/sucursal, tracking number
- Link para descargar etiqueta (solo admin por ahora)
- Mostrar en detalle de orden en perfil
- Estado del envío visible
- Fecha estimada de entrega

**DoD**

- Datos de envío claros y completos
- Tracking number copiable
- UI coherente con resto del sitio
- Mobile responsive

**Fuera de scope**

- Tracking en tiempo real
- Notificaciones de cambio de estado
- Mapa de seguimiento

---

### 3.10 Sucursales Andreani

**Descripción**
Permitir selección de sucursal para retiro.

**Tareas**

- Crear migración `andreani_sucursales` (cache local)
- Método `AndreaniClient::getSucursales(city, province)`
- Cache de sucursales por localidad (24 horas)
- Componente Livewire para selector de sucursal
- Mostrar: nombre, dirección, horarios
- Filtro por proximidad (código postal)
- Guardar sucursal_id en Shipment

**DoD**

- Usuario puede buscar sucursales cercanas
- Datos claros de cada sucursal
- Selección persistida correctamente
- Performance OK (cache efectivo)

**Fuera de scope**

- Mapa interactivo
- Disponibilidad en tiempo real
- Horarios especiales

**Seguridad**

- Cache invalidable manualmente
- Validar que sucursal existe
- No permitir sucursales inventadas

---

### 3.11 Actualización de total en Order

**Descripción**
Incluir costo de envío en total de la orden.

**Tareas**

- Modificar `Order` para separar: `subtotal`, `shipping_cost`, `total`
- Migración para agregar campos
- Calcular total = subtotal + shipping_cost
- Actualizar lógica de creación de orden
- Validar coherencia en payment
- Reflejar en confirmación y emails

**DoD**

- Total incluye envío correctamente
- Subtotal vs total distinguibles
- Payment valida total correcto
- Tests actualizados

**Fuera de scope**

- Descuentos
- Impuestos
- Múltiples monedas

**Seguridad**

- Validar que shipping_cost coincide con cotización
- No confiar en valor del cliente
- Re-calcular server-side

---

### 3.12 Manejo de errores de Andreani

**Descripción**
Gestionar fallos de API sin romper checkout.

**Tareas**

- Detectar errores: zona no cubierta, peso excedido, servicio caído
- Mensaje claro al usuario según tipo de error
- Fallback: permitir continuar sin calcular envío (admin resuelve)
- Log detallado de errores para debug
- Retry automático con backoff exponencial
- Timeout configurado (5s)
- Circuit breaker básico

**DoD**

- Errores no bloquean compra totalmente
- Usuario informado claramente
- Admin puede resolver manual
- Logs suficientes para debug

**Fuera de scope**

- Fallback a otro carrier
- Compensación automática
- Dashboard de salud de API

**Seguridad**

- No exponer detalles técnicos al usuario
- Rate limiting ante errores
- No loguear credenciales en errores

---

### 3.13 Tests end-to-end del slice

**Descripción**
Test automatizado del flujo completo con envío.

**Tareas**

- Test PEST: agregar dirección → checkout con cotización → pagar → etiqueta generada
- Mock de Andreani API (happy path)
- Test: error de API no rompe checkout
- Test: cotización cachea correctamente
- Test: total incluye envío
- Test: solo usuario puede ver su tracking
- Verificar integridad de Shipment

**DoD**

- Tests pasan en pipeline
- Cobertura > 80% del slice
- Mocks bien configurados
- Tests de seguridad incluidos

---

### 3.14 Background jobs para etiquetas

**Descripción**
Generar etiquetas de forma asíncrona.

**Tareas**

- Crear Job `GenerateAndreaniLabel`
- Dispatch desde listener de `OrderPaid`
- Configurar queue `shipments`
- Retry policy: 3 intentos con backoff
- Notificar a admin si falla definitivamente
- Log de job execution

**DoD**

- Etiquetas se generan async
- No bloquea checkout
- Retry automático funcional
- Logs claros de ejecución

**Fuera de scope**

- Priorización de jobs
- Jobs programados
- Dead letter queue manual

**Seguridad**

- Job solo procesa órdenes pagas
- Validar datos antes de API call
- No exponer datos en job failed logs

---

### 3.15 Panel admin para envíos (Filament básico)

**Descripción**
Vista de envíos en backoffice.

**Tareas**

- Crear FilamentResource `ShipmentResource`
- Listar envíos con filtros: estado, método, fecha
- Ver tracking number y descargar etiqueta
- Acción manual: regenerar etiqueta
- Acción: marcar como enviado
- Relación visible con Order

**DoD**

- Admin puede gestionar envíos
- Acciones funcionan correctamente
- Filtros operativos
- Download de PDF funcional

**Fuera de scope**

- Tracking automático (slice futuro)
- Impresión masiva
- Estadísticas de envíos

**Seguridad**

- Solo admin accede
- Policy verificada
- Actions validadas

---

### 3.16 Métricas de logística

**Descripción**
Instrumentar puntos clave de envíos.

**Tareas**

- Log de cotizaciones realizadas
- Log de errores de API Andreani
- Contador de métodos elegidos (domicilio vs sucursal)
- Tiempo promedio de generación de etiqueta
- Rate de éxito de API calls

**DoD**

- Métricas logueadas correctamente
- Información útil para optimización
- Sin datos sensibles

**Fuera de scope**

- Dashboard de analytics
- Alertas en tiempo real
- SLA tracking

---

## Restricciones

- No implementar tracking en tiempo real (slice futuro)
- No implementar múltiples carriers
- No manejar devoluciones
- No optimizar empaque automático
- No implementar envío gratis condicional

---

## Regla de corte

Si una tarea:

- No es parte del flujo crítico de envío
- Introduce múltiples carriers
- Depende de slices futuros (ML sync, admin avanzado)

→ **queda fuera del Slice 3**

---

## Nota de seguridad (obligatoria)

- Credenciales Andreani solo en `.env`
- Token nunca expuesto en logs o respuestas
- HTTPS obligatorio para API calls
- Validar ownership de direcciones y envíos
- Re-cotizar server-side antes de crear orden
- No confiar en precios del cliente
- Rate limiting en cotizaciones
- Timeout en API calls (5s)
- Validar formato de respuestas API
- Storage de etiquetas no público
- Policy estricta para descargar etiquetas
- Logs sin datos sensibles (direcciones completas, tracking)

---

## Orden de ejecución sugerido

1. Configuración Andreani API (3.1)
2. Modelo Dirección (3.2)
3. CRUD direcciones (3.3)
4. Cálculo peso/volumen (3.5)
5. Servicio cotización (3.4)
6. Modelo Shipment (3.7)
7. Actualización total Order (3.11)
8. Integración checkout (3.6)
9. Sucursales (3.10)
10. Generación etiqueta (3.8)
11. Background jobs (3.14)
12. Visualización datos envío (3.9)
13. Manejo errores (3.12)
14. Panel admin (3.15)
15. Tests e2e (3.13)
16. Métricas (3.16)

---

## Comandos útiles (vía Sail)

```bash
# Crear migraciones
./vendor/bin/sail artisan make:migration create_addresses_table
./vendor/bin/sail artisan make:migration create_shipments_table
./vendor/bin/sail artisan make:migration add_shipping_fields_to_orders
./vendor/bin/sail artisan make:migration add_dimensions_to_products

# Correr migraciones
./vendor/bin/sail artisan migrate

# Crear modelos
./vendor/bin/sail artisan make:model Address
./vendor/bin/sail artisan make:model Shipment

# Crear servicios
./vendor/bin/sail artisan make:class Services/AndreaniClient
./vendor/bin/sail artisan make:class Services/AndreaniShippingService
./vendor/bin/sail artisan make:class Services/AndreaniLabelService

# Crear componentes Livewire
./vendor/bin/sail artisan make:livewire Profile/Addresses
./vendor/bin/sail artisan make:livewire Checkout/ShippingSelector
./vendor/bin/sail artisan make:livewire Checkout/SucursalPicker

# Crear Jobs
./vendor/bin/sail artisan make:job GenerateAndreaniLabel

# Crear Filament Resource
./vendor/bin/sail artisan make:filament-resource Shipment

# Crear Policies
./vendor/bin/sail artisan make:policy AddressPolicy --model=Address
./vendor/bin/sail artisan make:policy ShipmentPolicy --model=Shipment

# Config
./vendor/bin/sail artisan config:publish andreani

# Tests
./vendor/bin/sail test --filter=ShippingTest
./vendor/bin/sail test --filter=AndreaniTest
./vendor/bin/sail test --coverage

# Queue
./vendor/bin/sail artisan queue:work --queue=shipments
./vendor/bin/sail artisan horizon
```

---

## Valor entregado

Al finalizar este slice:

- **Envíos reales calculados** → completitud del flujo de compra
- **Diferencial local** → integración con carrier argentino líder
- **Opciones al usuario** → domicilio vs sucursal aumenta conversión
- **Automatización logística** → etiquetas generadas automáticamente
- **Riesgo técnico mitigado** → integración compleja resuelta temprano

---

## Integración con slices anteriores

Este slice **extiende** Slice 1 y 2:

- Checkout recibe paso adicional de envío
- Total de orden ahora incluye shipping_cost
- Usuarios logueados pueden guardar direcciones
- Guest checkout permite ingresar dirección una vez
- Tests de slices anteriores deben seguir pasando

---

## Dependencias técnicas

**Requiere de Slice 1:**
- Modelo Order
- Modelo Product
- Flujo de checkout

**Requiere de Slice 2:**
- Usuarios registrados (opcional para direcciones guardadas)
- Guest checkout sigue funcionando

**Habilita para futuros slices:**
- Tracking automático de envíos
- Notificaciones de estado
- Reportes logísticos
- Optimización de costos

---

## Próximo slice recomendado

**Slice 4 — Backoffice operativo (Filament)** para gestión completa de productos, órdenes y configuraciones sin tocar código.
