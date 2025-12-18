---
domain: "Order"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-18"
purpose: "Define el dominio de pedidos del e-commerce"
---

# Order Domain

## Descripción

El dominio Order gestiona todo el ciclo de vida de los pedidos desde su creación hasta su entrega o cancelación.
Es el núcleo del sistema e-commerce, coordinando productos, pagos y entrega.

---

## Entidades Principales

### Order (Pedido)

**Propósito**: Representa un pedido realizado por un cliente.

**Propiedades**:
- `id`: OrderId (UUID)
- `customerName`: string (nombre del cliente)
- `phone`: PhoneNumber (teléfono normalizado +54...)
- `address`: Address (dirección de entrega, relación uno-a-uno)
- `items`: array<OrderItem> (líneas del pedido)
- `paymentMethod`: PaymentMethod enum
- `orderStatus`: OrderStatus enum
- `paymentStatus`: PaymentStatus enum
- `total`: Money (suma de items.subtotal)
- `notes`: ?string (observaciones del cliente)
- `internalNotes`: ?string (notas internas del merchant)
- `createdAt`: CarbonImmutable
- `updatedAt`: CarbonImmutable

**Relaciones**:
- Tiene muchos `OrderItem` (uno a muchos)
- Tiene una `Address` (uno a uno)
- Pertenece a un `User` (merchant, opcional para auditoría)

---

### OrderItem (Línea de Pedido)

**Propósito**: Representa un producto o variante en un pedido con precio histórico.

**Propiedades**:
- `id`: OrderItemId (UUID)
- `orderId`: OrderId
- `productId`: ProductId (referencia al producto)
- `productVariantId`: ?ProductVariantId (si tiene variante)
- `productName`: string (nombre histórico, **inmutable**)
- `variantName`: ?string (nombre de variante histórico, **inmutable**)
- `quantity`: Quantity (cantidad pedida)
- `unitPrice`: Money (precio unitario histórico, **inmutable**)
- `subtotal`: Money (unitPrice.cents * quantity.value, calculado)

**Relaciones**:
- Pertenece a un `Order`
- Referencia a `Product` (soft, no FK strict)
- Referencia a `ProductVariant` (soft, no FK strict)

**Importante**: 
- Los precios son **históricos e inmutables**
- Se guardan `productName` y `variantName` por si el producto se elimina

---

## Value Objects

### OrderId
- **Tipo**: UUID (string)
- **Validación**: UUID v4 válido
- **Generación**: `OrderId::generate()`

### Money
- **Propiedades**: `cents` (int), `currency` (string, siempre "ARS")
- **Validación**: `cents >= 0`, `currency === 'ARS'`
- **Métodos**: `add()`, `subtract()`, `multiply()`, `format()`

### PhoneNumber
- **Formato**: `+54` + código de área + número
- **Validación**: Formato argentino válido
- **Normalización**: Automática al construir
- **Ejemplo**: `+5491112345678`

### Quantity
- **Validación**: `value > 0`
- **Métodos**: `increment()`, `decrement()`

---

## Enums

### OrderStatus

```php
enum OrderStatus: string
{
    case NEW = 'new';
    case CONFIRMED = 'confirmed';
    case IN_DELIVERY = 'in_delivery';
    case DELIVERED = 'delivered';
    case REJECTED = 'rejected';
    case CANCELLED = 'cancelled';
}
```

**Métodos**:
- `canBeEdited(): bool` → false si `delivered` o `rejected`
- `canBeCancelled(): bool` → true solo si `new` o `confirmed`

---

### PaymentStatus

```php
enum PaymentStatus: string
{
    case PENDING = 'pending';
    case PAID = 'paid';
    case REFUNDED = 'refunded';
}
```

**Métodos**:
- `isPaid(): bool`
- `canBeRefunded(): bool`

---

### PaymentMethod

```php
enum PaymentMethod: string
{
    case MERCADO_PAGO = 'mercado_pago';
    case CASH = 'cash';
    case TRANSFER = 'transfer';
}
```

**Métodos**:
- `requiresExternalLink(): bool` → true si `mercado_pago`

---

## Reglas de Negocio Críticas

### 1. Límite de Pedidos Activos
- Máximo 2-5 pedidos activos por teléfono (configurable)
- Activos = `new`, `confirmed`, `in_delivery`

### 2. Stock Transaccional
- Descuento con **lock pessimista**
- Transaccional (rollback si falla)

### 3. Precios Históricos
- `OrderItem.unitPrice` es **inmutable**
- Se guarda al momento de creación

### 4. Edición Restringida
- NO se puede editar si `delivered` o `rejected`
- NO se pueden agregar/quitar items
- NO se pueden cambiar precios

### 5. Rate Limiting
- 5 intentos/hora por IP
- 3 pedidos/hora por teléfono

---

## Contratos

### Commands
- `CreateOrderInterface`
- `UpdateOrderInterface`
- `CancelOrderInterface`
- `ChangeOrderStatusInterface`
- `ChangePaymentStatusInterface`

### Queries
- `GetOrderInterface`
- `ListOrdersInterface`
- `GetOrdersByPhoneInterface`
- `CountActiveOrdersByPhoneInterface`

---

**Ver**: `@e-commerce-wa-ml/project_definition.md` para más contexto.
