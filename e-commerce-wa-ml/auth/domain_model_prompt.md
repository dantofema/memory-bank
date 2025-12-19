# Prompt: Generar Domain Model para un Módulo

## Contexto del Proyecto

Estás trabajando en un proyecto Laravel con arquitectura modular. Este proyecto requiere documentar el modelo de dominio de cada módulo de forma exhaustiva y profesional para que sirva como referencia durante el desarrollo.

### Stack Técnico
- **Framework:** Laravel 12
- **Arquitectura:** Monolito modular con Laravel Modules (nwidart/laravel-modules)
- **Presentación:** Filament v4 (backoffice) + Livewire v3 con Volt (frontend público)
- **Testing:** Pest v4
- **Base de datos:** PostgreSQL 15+
- **Calidad:** PHPStan level 6+, Pint (PSR-12), Rector

### Filosofía del Proyecto
- **Tipo:** Single-tenant e-commerce para emprendedores
- **Prioridad:** MVP - velocidad y simplicidad
- **Equipo:** 1 desarrollador
- **Calidad:** Tipado fuerte, análisis estático obligatorio, cobertura de tests prácticamente 100%

---

## Instrucciones

Genera un documento de **Domain Model** completo y profesional para el módulo especificado. Este documento debe servir como referencia definitiva del diseño del módulo, incluyendo entidades, value objects, enums, acciones, reglas de negocio, eventos y consideraciones técnicas.

### Archivos de Referencia

**IMPORTANTE:** Debes leer y respetar la arquitectura, convenciones y restricciones definidas en los siguientes archivos:

1. **Definición del Proyecto:**
   - `@e-commerce-wa-ml/project_definition.md`
   - Contiene: alcance del proyecto, funcionalidades, stakeholders, flujos críticos, estados, métodos de pago, promociones, riesgos, stack técnico

2. **Arquitectura Modular:**
   - `@e-commerce-wa-ml/modular-architecture.md`
   - Contiene: módulos del sistema, responsabilidades, comunicación entre módulos (interfaces y eventos), diagramas de flujo, restricciones por módulo

3. **Convenciones Laravel:**
   - `@laravel/conventions/conventions.md`
   - Contiene: convenciones técnicas, arquitectura, Value Objects, Spatie Laravel Data, estructura de tests, APIs REST, base de datos

4. **Ejemplo de Referencia:**
   - `@e-commerce-wa-ml/orders/domain_model.md`
   - Referencia completa de estructura y contenido esperado

---

## Estructura del Documento a Generar

El documento debe seguir esta estructura exacta:

### 1. Module Overview

**Secciones:**
- **Name:** Nombre del módulo (español e inglés)
- **Type:** CORE | TRANSVERSAL | SOPORTE
- **Purpose:** Propósito del módulo en una frase clara
- **Responsibilities:** Lista detallada de responsabilidades (bullet points)
- **Dependencies:** Módulos de los que depende con explicación breve

**Ejemplo:**
```markdown
## Module Overview

**Name:** Orders (Pedidos)  
**Type:** CORE Module  
**Purpose:** Creación, gestión y auditoría de pedidos con control transaccional de stock y estados

**Responsibilities:**
- Creación de pedidos desde el carrito de compras
- Validación de stock y aplicación de promociones
- Gestión de estados de pedido (OrderStatus) y pago (PaymentStatus)
- Edición controlada de pedidos con restricciones según estado
- Auditoría de cambios de estado
- Gestión de direcciones de entrega
- Control transaccional de stock con locks pessimistas

**Dependencies:**
- **Catalog:** Validación de productos, stock, precios y aplicación de promociones
- **Payments:** Estado de pago y método de pago seleccionado
- **WhatsApp:** Notificaciones de creación y cambios de estado
- **Security:** Rate limiting, validación de teléfono, límite de pedidos activos
```

---

### 2. Domain Model Class Diagram

**Instrucciones:**
- Crear diagrama de clases completo en formato Mermaid
- Incluir todas las entidades del módulo con sus propiedades y métodos clave
- Incluir Value Objects con sus métodos principales
- Incluir Enumerations con sus valores
- Incluir Actions con sus métodos de ejecución
- Incluir Data Transfer Objects (DTOs)
- Mostrar relaciones entre entidades (composición, agregación, uso)
- Usar estereotipos: `<<value_object>>`, `<<enumeration>>`, `<<data>>`

**Elementos a incluir:**
- Entidades con atributos y métodos públicos principales
- Value Objects compartidos y específicos del módulo
- Enums con valores y métodos de comportamiento
- Actions (clases que ejecutan lógica de negocio)
- DTOs (Data Transfer Objects para inputs/outputs)
- Relaciones: asociaciones, composiciones, dependencias

---

### 3. Entities

**Para cada entidad del módulo:**

#### 3.1. Nombre de la Entidad

**Secciones:**

- **Description:** Descripción breve del propósito de la entidad

- **Key Attributes:** Lista de propiedades principales con tipo y descripción
  - Formato: `nombre: tipo - descripción breve`

- **Business Rules:** Lista de reglas de negocio que aplican a la entidad
  - Cuándo puede editarse
  - Restricciones sobre valores
  - Comportamientos especiales
  - Relaciones con otras entidades

- **Invariants:** Condiciones que siempre deben cumplirse
  - Formato: `campo = expresión` o `condición lógica`
  - Ejemplo: `total = subtotal - discount`

**Ejemplo completo:**
```markdown
### Order

**Description:** Pedido realizado por un cliente (usuario no autenticado)

**Key Attributes:**
- `order_number`: string - Número único de pedido (formato: ORD-YYYYMMDD-XXXXX)
- `customer_name`: string - Nombre del cliente (no editable post-entrega)
- `customer_phone`: PhoneNumber - Teléfono normalizado del cliente
- `status`: OrderStatus - Estado actual del pedido
- `payment_status`: PaymentStatus - Estado del pago
- `total`: Money - Total final a pagar

**Business Rules:**
- Un pedido solo puede editarse si no está en estado `DELIVERED` o `REFUNDED`
- El nombre del cliente no puede editarse una vez confirmado el pedido
- Los precios de los items son históricos y no cambian si cambia el precio del producto
- El stock se descuenta transaccionalmente al crear el pedido
- Un mismo teléfono puede tener máximo N pedidos activos (configurable 2-5)

**Invariants:**
- `total = subtotal - discount`
- `subtotal = sum(order_items.total)`
- `confirmed_at != null` si `status != NEW`
```

---

### 4. Value Objects

**Para cada Value Object:**

#### 4.1. Nombre del Value Object

**Secciones:**

- **Description:** Propósito y razón de existencia del VO

- **Attributes:** Propiedades internas del VO con tipo

- **Validation Rules:** Reglas que se validan en el constructor
  - Formato esperado
  - Rangos permitidos
  - Invariantes específicas

- **Methods:** Métodos públicos principales
  - Operaciones aritméticas (si aplica)
  - Conversiones de formato
  - Comparaciones
  - Métodos de dominio específicos

**Criterios para decidir usar un Value Object:**
1. **Tiene reglas de negocio propias** (validación compleja, comportamiento especializado)
2. **Se reutiliza en múltiples contextos** (varios modelos o servicios)
3. **No debe existir inválido** (constructor garantiza estado válido)
4. **Aporta semántica clara al dominio** (tipo primitivo no expresa suficiente significado)

**Ejemplo completo:**
```markdown
### Money

**Description:** Valor monetario con operaciones aritméticas seguras

**Attributes:**
- `amount`: int - Cantidad en centavos
- `currency`: string - Moneda (MVP: ARS)

**Validation Rules:**
- Amount debe ser >= 0 en la mayoría de contextos
- Currency debe ser código ISO válido

**Methods:**
- `add(Money other)`: Suma dos valores monetarios
- `subtract(Money other)`: Resta dos valores monetarios
- `multiply(float factor)`: Multiplica el monto
- `divide(float divisor)`: Divide el monto
- `format()`: Retorna formato legible (ej: $1.234,56)
- `toDecimal()`: Conversión a float para APIs
- `isZero()`: Verifica si el monto es cero
- `isPositive()`: Verifica si el monto es positivo
```

---

### 5. Enumerations

**Para cada Enum:**

#### 5.1. Nombre del Enum

**Secciones:**

- **Values:** Lista de valores permitidos con descripción

- **Valid Transitions:** (si aplica) Transiciones válidas entre estados
  - Formato diagrama de flujo simple o lista

- **Terminal States:** (si aplica) Estados terminales que no permiten transiciones

- **Methods:** (si aplica) Métodos del enum
  - `label()`: Etiqueta legible
  - `color()`: Color para UI
  - `canTransitionTo()`: Validación de transiciones

**Ejemplo completo:**
```markdown
### OrderStatus

**Values:**
- `NEW`: Pedido creado, esperando confirmación del merchant
- `CONFIRMED`: Merchant confirmó el pedido
- `IN_DELIVERY`: Pedido en camino
- `DELIVERED`: Entregado al cliente
- `REJECTED`: Merchant rechazó el pedido
- `CANCELLED`: Cliente o merchant canceló

**Valid Transitions:**
```
NEW → CONFIRMED | REJECTED | CANCELLED
CONFIRMED → IN_DELIVERY | CANCELLED
IN_DELIVERY → DELIVERED | CANCELLED
DELIVERED → (terminal)
REJECTED → (terminal)
CANCELLED → (terminal)
```

**Terminal States:** DELIVERED, REJECTED, CANCELLED

**Methods:**
- `canTransitionTo(OrderStatus target)`: Valida si la transición es permitida
- `isTerminal()`: Retorna true si es un estado terminal
- `label()`: Retorna etiqueta legible en español
- `color()`: Retorna color para badges de Filament
```

---

### 6. Actions

**Para cada Action:**

#### 6.1. Nombre de la Action

**Secciones:**

- **Purpose:** Propósito y responsabilidad de la Action

- **Input:** DTOs o Value Objects que recibe como parámetros

- **Output:** Tipo de retorno (entidad, DTO, void)

- **Steps:** Lista numerada de pasos que ejecuta la Action
  1. Paso 1 con descripción clara
  2. Paso 2...

- **Validations:** Validaciones que realiza antes de ejecutar
  - Validaciones de negocio
  - Validaciones de estado
  - Validaciones de datos

- **Exceptions:** Excepciones específicas que puede lanzar
  - Nombre de la excepción
  - Condición que la dispara

- **Side Effects:** (si aplica) Efectos secundarios importantes
  - Liberación de recursos
  - Notificaciones enviadas
  - Eventos disparados

**Ejemplo completo:**
```markdown
### CreateOrderAction

**Purpose:** Crear un pedido validando stock, aplicando promociones y reservando productos

**Input:** `CreateOrderData`

**Output:** `Order`

**Steps:**
1. Validar límite de pedidos activos por teléfono
2. Validar stock disponible para todos los items
3. Aplicar promociones vigentes
4. Calcular totales (subtotal, descuentos, total)
5. Crear pedido con estado `NEW` y pago `PENDING`
6. Crear items del pedido con precios históricos
7. Crear dirección de entrega
8. Reservar stock de forma transaccional (lock pessimista)
9. Disparar evento OrderCreated para notificación WhatsApp

**Validations:**
- Stock suficiente para cada item
- Teléfono no excede límite de pedidos activos
- Rate limiting por IP y teléfono
- Datos de dirección completos
- Promociones válidas y vigentes

**Exceptions:**
- `InsufficientStockException`: No hay stock suficiente para un item
- `ActiveOrdersLimitExceededException`: El teléfono excede el límite de pedidos activos
- `RateLimitExceededException`: Se excedió el rate limit
- `ValidationException`: Los datos de entrada no son válidos

**Side Effects:**
- Reserva de stock transaccional
- Evento OrderCreated para notificación WhatsApp
```

---

### 7. Data Transfer Objects

**Para cada DTO:**

#### 7.1. Nombre del DTO

**Secciones:**

- **Purpose:** Propósito del DTO (input de acción, output de query, etc.)

- **Attributes:** Lista de propiedades con tipo y obligatoriedad
  - Formato: `nombre: tipo (required|optional) - descripción`

- **Validations:** (si aplica) Reglas de validación específicas

**Ejemplo completo:**
```markdown
### CreateOrderData

**Purpose:** Datos para crear un nuevo pedido

**Attributes:**
- `customer_name`: string (required) - Nombre del cliente
- `customer_phone`: string (required) - Teléfono del cliente
- `notes`: string (optional) - Observaciones del pedido
- `payment_method`: PaymentMethod (required) - Método de pago seleccionado
- `address`: AddressData (required) - Dirección de entrega
- `items`: OrderItemData[] (required, min: 1) - Items del pedido

**Validations:**
- customer_name: longitud mínima 2 caracteres
- customer_phone: formato válido para Argentina
- items: al menos 1 item requerido
```

---

### 8. Events

**Para cada Evento de Dominio:**

#### 8.1. Nombre del Evento

**Secciones:**

- **Purpose:** (opcional) Propósito del evento si no es obvio

- **Payload:** Datos incluidos en el evento
  - Formato: `campo: tipo - descripción`

- **Listeners:** Qué módulos/listeners consumen este evento
  - Nombre del listener
  - Acción que ejecuta

- **Timing:** (si aplica) Cuándo se dispara el evento

**Ejemplo completo:**
```markdown
### OrderCreated

**Payload:**
- `order_id`: int - ID del pedido creado
- `order_number`: string - Número de pedido
- `customer_phone`: string - Teléfono del cliente
- `total`: Money - Total del pedido

**Listeners:**
- `SendOrderWhatsAppNotification`: Envía notificación por WhatsApp al merchant

**Timing:** Se dispara inmediatamente después de crear el pedido y confirmar la transacción de stock
```

---

### 9. Business Rules Summary

**Secciones:**
- Agrupar reglas por tema (Creation, Editing, Status Transitions, etc.)
- Listar reglas en formato numerado
- Reglas deben ser claras, accionables y sin ambigüedad

**Ejemplo:**
```markdown
## Business Rules Summary

### Order Creation
1. Stock must be available for all items
2. Phone must not exceed active orders limit (configurable 2-5)
3. Rate limiting: 5 attempts/hour per IP, 3 orders/hour per phone
4. Stock is reserved transactionally with pessimistic locking
5. Historical prices are saved and never change
6. Promotions are applied at creation time based on validity dates

### Order Editing
1. Orders in DELIVERED or REFUNDED status cannot be edited
2. Customer name cannot be changed after confirmation
3. Products cannot be added or removed, only quantities modified
4. Quantity changes require stock validation
5. Payment method can be changed at any time
6. Address can be edited until delivery
```

---

### 10. Database Schema Considerations

**Secciones:**

- **Tables:** Lista de tablas principales del módulo

- **Indexes:** Índices requeridos con justificación
  - Tabla.columna: Justificación del índice

- **Foreign Keys:** Claves foráneas con relaciones
  - Formato: `tabla.columna → tabla_referencia.columna`

- **Soft Deletes:** Tablas que usan soft deletes y por qué

**Ejemplo:**
```markdown
## Database Schema Considerations

### Tables
- `orders`: Main order entity
- `order_items`: Order line items
- `addresses`: Delivery addresses (1:1 with orders)
- `order_status_logs`: Audit trail for status changes

### Indexes
- `orders.customer_phone`: For active orders lookup
- `orders.order_number`: Unique index
- `orders.status`: For filtering by status
- `orders.created_at`: For date range queries
- `order_status_logs.order_id`: For audit trail retrieval

### Foreign Keys
- `order_items.order_id` → `orders.id`
- `order_items.product_id` → `products.id`
- `addresses.order_id` → `orders.id`
- `order_status_logs.order_id` → `orders.id`

### Soft Deletes
- `orders`: Use soft deletes for data retention and audit
- `order_status_logs`: Never delete (audit requirement)
```

---

### 11. Integration Points

**Secciones:**
- **With {ModuleName}:** Para cada módulo con el que se integra
  - Listar operaciones o datos que se intercambian
  - Formato: bullet points con descripción clara

**Ejemplo:**
```markdown
## Integration Points

### With Catalog Module
- Validate product and variant existence
- Check stock availability
- Get current prices for historical recording
- Apply promotions to order items

### With Payments Module
- Receive payment method selection
- Generate Mercado Pago links
- Receive webhook notifications for payment status

### With WhatsApp Module
- Send order creation notification
- Send status change notifications
- Send payment confirmation notifications
```

---

### 12. Testing Strategy

**Secciones:**

- **Unit Tests:** Qué testear a nivel unitario
  - Value Objects
  - Enums
  - Métodos de utilidad

- **Feature Tests:** Qué testear a nivel de integración
  - Flujos completos
  - Actions
  - Validaciones

- **Integration Tests:** (si aplica) Tests de integración con otros módulos

- **Edge Cases:** Casos extremos importantes a testear

**Ejemplo:**
```markdown
## Testing Strategy

### Unit Tests
- Value Objects validation and behavior
- Enum transitions validation
- Money arithmetic operations
- PhoneNumber normalization

### Feature Tests
- Order creation with stock validation
- Order editing with restrictions
- Status transitions and validation
- Active orders limit validation
- Stock release on cancellation
- Audit log creation

### Edge Cases
- Concurrent order creation with limited stock
- Status transition attempts from terminal states
- Editing orders in non-editable states
- Exceeding active orders limit
```

---

### 13. Performance Considerations

**Secciones:**

- **Optimizations:** Optimizaciones implementadas o requeridas

- **Scalability:** Consideraciones de escalabilidad
  - Queries pesadas
  - Caching
  - Queue processing

**Ejemplo:**
```markdown
## Performance Considerations

### Optimizations
- Pessimistic locking only during stock reservation
- Eager loading of order_items and address in queries
- Indexed phone lookups for active orders validation
- Cached configuration values (rate limits, active orders limit)

### Scalability
- Queue-based stock operations for high concurrency
- Read replicas for order listing and reports
- Caching of order counts per phone
```

---

### 14. AI-Friendly Annotations

**Secciones:**

- **Key Concepts:** Lista de conceptos clave con etiquetas
  - aggregate_root
  - value_objects
  - enums
  - actions
  - domain_events

- **Invariants to Maintain:** Lista de invariantes críticas

- **Critical Paths:** Flujos críticos del módulo

- **Data Consistency:** Reglas de consistencia de datos

**Ejemplo:**
```markdown
## AI-Friendly Annotations

**Key Concepts:**
- aggregate_root: Order
- value_objects: PhoneNumber, Money, Quantity, Coordinates
- enums: OrderStatus, PaymentStatus, PaymentMethod
- actions: CreateOrderAction, UpdateOrderAction, ChangeOrderStatusAction
- domain_events: OrderCreated, OrderStatusChanged, PaymentStatusChanged

**Invariants to Maintain:**
- order.total = order.subtotal - order.discount
- order_item.total = order_item.subtotal - order_item.discount
- quantity > 0 for all order items
- stock reservation must be transactional and atomic

**Critical Paths:**
- Order creation with stock validation and reservation
- Status transitions with audit logging
- Stock release on cancellation

**Data Consistency:**
- Historical prices never change after order creation
- Audit logs are immutable
- Stock operations must be atomic
```

---

## Lineamientos de Redacción

### Estilo
- **Lenguaje:** Técnico pero claro, evitar ambigüedades
- **Formato:** Markdown con estructura consistente
- **Código:** Usar bloques de código para ejemplos de clases, métodos, diagramas
- **Diagramas:** Mermaid para diagramas de clases y flujos

### Precisión
- **Nombres:** Usar nombres exactos de clases, métodos, enums
- **Tipos:** Especificar tipos completos (int, string, Money, PhoneNumber, etc.)
- **Obligatoriedad:** Distinguir entre required, optional, nullable

### Consistencia
- Respetar convenciones de Laravel y del proyecto
- Usar nomenclatura consistente con `modular-architecture.md`
- Alinearse con patrones del ejemplo `orders/domain_model.md`

### Completitud
- No omitir secciones de la estructura
- Si una sección no aplica, indicar "N/A" o "Not applicable in MVP"
- Documentar todos los Value Objects, Enums, Actions, Eventos

---

## Validación del Resultado

Antes de entregar el documento, verifica:

1. ✅ **Estructura completa:** Todas las secciones numeradas están presentes
2. ✅ **Diagrama Mermaid:** El diagrama se renderiza correctamente y es completo
3. ✅ **Consistencia arquitectónica:** El diseño respeta `modular-architecture.md`
4. ✅ **Convenciones técnicas:** El diseño respeta `conventions.md`
5. ✅ **Reglas de negocio:** Todas las reglas de `project_definition.md` están reflejadas
6. ✅ **Value Objects justificados:** Cada VO cumple al menos 1 criterio de uso
7. ✅ **Actions completas:** Cada Action tiene Purpose, Input, Output, Steps, Validations, Exceptions
8. ✅ **Eventos bien definidos:** Payload claro, listeners identificados
9. ✅ **Sin contradicciones:** No hay inconsistencias con otros módulos
10. ✅ **Testing exhaustivo:** Strategy cubre Unit, Feature, Integration, Edge Cases

---

## Output Esperado

Un archivo Markdown con el nombre `{module_name}/domain_model.md` que:
- Tiene entre 800-1000 líneas aproximadamente (según complejidad del módulo)
- Sigue la estructura especificada al 100%
- Es autónomo y completo (no requiere leer otros archivos para entenderlo)
- Sirve como referencia definitiva para implementar el módulo
- Es optimizado para lectura por IA y humanos

---

## Notas Finales

- **Completitud sobre brevedad:** Es mejor ser exhaustivo que resumir en exceso
- **Ejemplos cuando ayudan:** Incluir ejemplos de código si clarifican conceptos complejos
- **Diagramas cuando ayudan:** Además del Class Diagram principal, se pueden agregar diagramas de secuencia o estado si clarifican flujos complejos
- **Revisión final:** Asegúrate de que el documento sea coherente de principio a fin

**Este prompt está diseñado para generar documentación profesional y completa que sirva como fundamento sólido para el desarrollo del módulo.**
