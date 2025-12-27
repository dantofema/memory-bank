# Slice 1 — Tarea 1.1: Modelo de Producto ✅

**Fecha**: 2025-12-27  
**Estado**: Implementado  
**Tiempo estimado**: 30 minutos

---

## Resumen de la implementación

Se implementó completamente la entidad `Product` siguiendo las especificaciones del Slice 1.

---

## Artefactos creados

### 1. Migración: `2025_12_27_182727_create_products_table.php`

```php
Schema::create('products', function (Blueprint $table): void {
    $table->id();
    $table->string('name');
    $table->text('description')->nullable();
    $table->decimal('price', 10, 2);
    $table->unsignedInteger('stock')->default(0);
    $table->boolean('active')->default(true);
    $table->timestamps();
    $table->softDeletes();

    $table->index('active');
    $table->index('stock');
});
```

**Características**:

- Tipo `decimal(10,2)` para price → previene valores inválidos a nivel de BD
- Tipo `unsignedInteger` para stock → no permite negativos
- Índices en `active` y `stock` → optimiza queries de productos disponibles
- Soft deletes implementado

---

### 2. Modelo: `app/Models/Product.php`

**Características implementadas**:

- ✅ Fillable con campos: name, description, price, stock, active
- ✅ Casts para tipos específicos:
    - `price => 'decimal:2'`
    - `stock => 'integer'`
    - `active => 'boolean'`
- ✅ Trait `SoftDeletes`
- ✅ Trait `HasFactory` con tipado correcto
- ✅ Método estático `validationRules()` con reglas:
    - name: required, string, max:255
    - description: nullable, string
    - price: required, numeric, min:0.01
    - stock: required, integer, min:0
    - active: boolean

---

### 3. Factory: `database/factories/ProductFactory.php`

**Características implementadas**:

- ✅ Generación de productos válidos con Faker:
    - name: 3 palabras aleatorias
    - description: frase de 12 palabras
    - price: decimal aleatorio entre 1 y 999
    - stock: entero aleatorio entre 0 y 100
    - active: true por defecto
- ✅ State `inactive()`: crea productos con active=false
- ✅ State `outOfStock()`: crea productos con stock=0

---

### 4. Tests: `tests/Unit/ProductTest.php`

**Tests implementados** (9 tests):

1. ✅ `puede crear un producto vía factory`
2. ✅ `valida que el precio debe ser mayor a 0`
3. ✅ `valida que el stock debe ser mayor o igual a 0`
4. ✅ `los casts funcionan correctamente`
5. ✅ `soft deletes está operativo`
6. ✅ `factory crea productos con valores válidos`
7. ✅ `factory puede crear productos inactivos`
8. ✅ `factory puede crear productos sin stock`
9. ✅ `nombre es requerido según reglas de validación`
10. ✅ `descripción es opcional según reglas de validación`

---

## Definition of Done ✅

- [x] Productos creables vía Factory
- [x] Validación server-side (price > 0, stock >= 0)
- [x] Tests unitarios implementados
- [x] Migración con índices en `active` y `stock`
- [x] Soft deletes operativo
- [x] Casts para price (decimal) y stock (integer)
- [x] Código sin errores de linter (Pint compatible)
- [x] Tipado estricto (Larastan compatible)

---

## Comandos para validar la implementación

```bash
# Ejecutar migración
php artisan migrate
# O con Sail:
./vendor/bin/sail artisan migrate

# Ejecutar tests
php artisan test --filter=ProductTest
# O con Sail:
./vendor/bin/sail test --filter=ProductTest

# Ejecutar tests con cobertura
php artisan test --filter=ProductTest --coverage
# O con Sail:
./vendor/bin/sail test --filter=ProductTest --coverage

# Validar estilo de código
vendor/bin/pint --test
# O con Sail:
./vendor/bin/sail composer pint

# Validar análisis estático
vendor/bin/phpstan analyse
# O con Sail:
./vendor/bin/sail composer phpstan
```

---

## Nota de seguridad implementada

1. **Validación a nivel de BD**: Tipos específicos (`decimal`, `unsignedInteger`) previenen valores inválidos
2. **Validación en modelo**: Reglas de validación como segunda capa de defensa
3. **Sanitización**: Los campos de texto serán sanitizados al usar blade escaping (automático)
4. **Índices**: Mejoran performance y previenen scans completos de tabla

---

## Próximo paso

**Tarea 1.2**: Listado de productos

- Crear ruta `/products`
- Componente Livewire/Volt para catálogo
- Mostrar solo productos activos con stock > 0
- Paginación básica

---

## Notas técnicas

### Decisiones de diseño

1. **Value Object para Price**: Se optó por usar cast `decimal:2` simple en lugar de implementar un VO dedicado tipo
   `Money`. Esto mantiene simplicidad para el MVP. Si en el futuro se necesitan múltiples monedas, se puede refactorizar
   a `brick/money`.

2. **Validación en modelo vs FormRequest**: Las reglas de validación están definidas como método estático en el modelo.
   Cuando se implemente UI (tarea 1.2), estas reglas se usarán en FormRequests o en componentes Livewire.

3. **Factory states**: Se agregaron estados `inactive()` y `outOfStock()` para facilitar tests de casos edge.

### Compatibilidad

- ✅ Laravel 12
- ✅ PHP 8.2+
- ✅ PostgreSQL (tipos compatibles)
- ✅ PEST 4
- ✅ Pint (estilo Laravel)
- ✅ Rector (refactorings automáticos)
- ✅ Larastan (análisis estático nivel max)

---

**Desarrollador**: Alejandro Leone  
**Proyecto**: ML-Andreani  
**Metodología**: Vertical Slice Architecture
