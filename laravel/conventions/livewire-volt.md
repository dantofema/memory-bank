---
name: "Livewire Volt Conventions"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-19"
purpose: "Convenciones para desarrollo con Livewire Volt en proyectos Laravel"
context:
  framework: "Laravel 12 + Livewire v3 + Volt"
  priority: "Archivos manejables, testeable, mantenible"
  note: "Sin Controllers, sin Services - solo Actions"
---

# Convenciones Livewire Volt

## Resumen

Convenciones y mejores prácticas para desarrollo con Livewire Volt. Enfocado en mantener archivos de tamaño manejable,
clara separación de responsabilidades y código testeable.

---

## Filosofía General

- **Sin Controllers**: Las rutas llaman directamente a Volt
- **Archivos pequeños**: Máximo 150 líneas por componente Volt
- **Single Responsibility**: Cada componente hace una sola cosa
- **Composición sobre extensión**: Dividir en múltiples componentes pequeños

---

## Estructura de Rutas

### Definición en `routes/web.php`

```php
use Livewire\Volt\Volt;

// ✅ Rutas simples
Volt::route('/products', 'products.index');
Volt::route('/products/{product}', 'products.show');

// ✅ Rutas con nombre
Volt::route('/products/create', 'products.create')->name('products.create');

// ✅ Rutas agrupadas con middleware
Route::middleware(['auth'])->group(function () {
    Volt::route('/admin/products', 'admin.products.index');
    Volt::route('/admin/products/{product}/edit', 'admin.products.edit');
});

// ❌ NO usar Controllers
Route::get('/products', [ProductController::class, 'index']); // EVITAR
```

---

## Límites de Tamaño y Complejidad

### Regla de las 150 Líneas

**Un componente Volt NO debe superar las 150 líneas**. Si se excede, aplicar estrategias de división.

#### Indicadores de que un componente es demasiado grande:

- ✅ Más de 150 líneas totales
- ✅ Más de 4 métodos públicos
- ✅ Más de 3 computed properties
- ✅ Lógica de negocio compleja mezclada con lógica de presentación
- ✅ Múltiples responsabilidades (listado + edición + creación)

---

## Estrategias para Limitar Tamaño

### 1. Extraer Actions (Lógica de Negocio)

**Cuándo**: Cuando hay lógica compleja que no es específica de UI.

**Antes** (componente grande):

```php
<?php

use function Livewire\Volt\{state, rules};

state(['name', 'email', 'phone', 'address']);

rules(['name' => 'required', 'email' => 'email']);

$save = function () {
    $this->validate();
    
    // 50+ líneas de lógica de negocio
    $user = User::create([...]);
    $user->sendWelcomeEmail();
    $user->createDefaultSettings();
    event(new UserCreated($user));
    // ...
    
    redirect()->route('users.show', $user);
};

?>
```

**Después** (componente pequeño + Action):

```php
<?php
// resources/views/livewire/users/create.blade.php

use App\Actions\Users\CreateUserAction;
use function Livewire\Volt\{state, rules};

state(['name', 'email', 'phone', 'address']);

rules(['name' => 'required', 'email' => 'email']);

$save = function (CreateUserAction $action) {
    $validated = $this->validate();
    
    $user = $action->execute(
        CreateUserData::from($validated)
    );
    
    redirect()->route('users.show', $user);
};

?>
```

```php
<?php
// app/Actions/Users/CreateUserAction.php

namespace App\Actions\Users;

final class CreateUserAction
{
    public function execute(CreateUserData $data): User
    {
        $user = User::create($data->toArray());
        $user->sendWelcomeEmail();
        $user->createDefaultSettings();
        event(new UserCreated($user));
        
        return $user;
    }
}
```

---

### 2. Dividir en Sub-componentes (Composición)

**Cuándo**: Cuando un componente maneja múltiples secciones de UI independientes.

**Antes** (monolítico):

```php
<?php
// resources/views/livewire/products/show.blade.php

use function Livewire\Volt\{state, computed};

state(['product']);

$relatedProducts = computed(fn () => $this->product->related()->take(5)->get());
$reviews = computed(fn () => $this->product->reviews()->paginate(10));
$similar = computed(fn () => Product::similar($this->product)->take(8)->get());

?>

<div>
    <!-- Product details: 30 lines -->
    <!-- Related products: 20 lines -->
    <!-- Reviews section: 40 lines -->
    <!-- Similar products: 25 lines -->
    <!-- Total: 115+ lines solo en template -->
</div>
```

**Después** (componentizado):

```php
<?php
// resources/views/livewire/products/show.blade.php

use function Livewire\Volt\{state};

state(['product']);

?>

<div>
    <livewire:products.product-details :product="$product" />
    <livewire:products.related-products :product="$product" />
    <livewire:products.reviews-section :product="$product" />
    <livewire:products.similar-products :product="$product" />
</div>
```

Cada sub-componente es pequeño, enfocado y reutilizable (20-40 líneas cada uno).

---

### 3. Extraer Form Objects (Validación Compleja)

**Cuándo**: Formularios con muchas reglas de validación o campos condicionales.

**Antes**:

```php
<?php

use function Livewire\Volt\{state, rules};

state(['name', 'email', 'phone', 'address', 'city', 'country', 'postal_code', 'type', 'tax_id']);

rules([
    'name' => 'required|min:3|max:255',
    'email' => 'required|email|unique:users,email',
    'phone' => 'required|regex:/^\+?[1-9]\d{1,14}$/',
    'address' => 'required|min:10',
    'city' => 'required',
    'country' => 'required|in:AR,BR,CL,UY',
    'postal_code' => 'required_if:country,AR|regex:/^\d{4}$/',
    'type' => 'required|in:individual,business',
    'tax_id' => 'required_if:type,business|regex:/^\d{11}$/',
]);

?>
```

**Después**:

```php
<?php
// resources/views/livewire/users/create.blade.php

use App\Http\Requests\CreateUserRequest;
use function Livewire\Volt\{state, form};

form(CreateUserRequest::class);

$save = function () {
    $validated = $this->form->validate();
    // ...
};

?>
```

```php
<?php
// app/Http/Requests/CreateUserRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => 'required|min:3|max:255',
            'email' => 'required|email|unique:users,email',
            // ... resto de las reglas
        ];
    }
    
    public function messages(): array
    {
        return [
            'postal_code.required_if' => 'El código postal es requerido para Argentina',
            // ...
        ];
    }
}
```

---

### 4. Usar Traits para Comportamiento Compartido

**Cuándo**: Cuando múltiples componentes necesitan la misma lógica (paginación, búsqueda, filtros).

```php
<?php
// app/Livewire/Concerns/WithTableActions.php

namespace App\Livewire\Concerns;

use Livewire\WithPagination;

trait WithTableActions
{
    use WithPagination;
    
    public string $search = '';
    public string $sortField = 'created_at';
    public string $sortDirection = 'desc';
    
    public function updatedSearch(): void
    {
        $this->resetPage();
    }
    
    public function sortBy(string $field): void
    {
        if ($this->sortField === $field) {
            $this->sortDirection = $this->sortDirection === 'asc' ? 'desc' : 'asc';
        } else {
            $this->sortField = $field;
            $this->sortDirection = 'asc';
        }
    }
}
```

```php
<?php
// resources/views/livewire/products/index.blade.php

use App\Livewire\Concerns\WithTableActions;
use function Livewire\Volt\{uses, computed};

uses([WithTableActions::class]);

$products = computed(function () {
    return Product::query()
        ->when($this->search, fn ($q) => $q->where('name', 'like', "%{$this->search}%"))
        ->orderBy($this->sortField, $this->sortDirection)
        ->paginate(15);
});

?>
```

---

### 5. Separar Computed Properties Complejos

**Cuándo**: Computed properties con lógica compleja o múltiples queries.

**Antes**:

```php
<?php

use function Livewire\Volt\{computed};

$dashboardStats = computed(function () {
    $totalRevenue = Order::sum('total');
    $pendingOrders = Order::pending()->count();
    $completedOrders = Order::completed()->count();
    $topProducts = Product::withCount('orders')
        ->orderBy('orders_count', 'desc')
        ->take(10)
        ->get();
    $recentCustomers = User::where('created_at', '>', now()->subDays(7))
        ->count();
    // ... 20+ líneas más
    
    return compact('totalRevenue', 'pendingOrders', ...);
});

?>
```

**Después**:

```php
<?php
// resources/views/livewire/dashboard/index.blade.php

use App\Actions\Dashboard\GetDashboardStatsAction;
use function Livewire\Volt\{computed};

$stats = computed(fn (GetDashboardStatsAction $action) => $action->execute());

?>
```

```php
<?php
// app/Actions/Dashboard/GetDashboardStatsAction.php

namespace App\Actions\Dashboard;

final class GetDashboardStatsAction
{
    public function execute(): DashboardStatsData
    {
        return new DashboardStatsData(
            totalRevenue: Order::sum('total'),
            pendingOrders: Order::pending()->count(),
            // ...
        );
    }
}
```

---

## Organización de Archivos Volt

### Estructura de Carpetas

```
resources/views/livewire/
├── products/
│   ├── index.blade.php           # Listado (máx 150 líneas)
│   ├── show.blade.php            # Detalle (máx 150 líneas)
│   ├── create.blade.php          # Creación (máx 150 líneas)
│   ├── edit.blade.php            # Edición (máx 150 líneas)
│   ├── product-details.blade.php # Sub-componente
│   ├── related-products.blade.php # Sub-componente
│   └── reviews-section.blade.php  # Sub-componente
├── orders/
│   ├── index.blade.php
│   ├── show.blade.php
│   └── order-items.blade.php     # Sub-componente
└── shared/
    ├── search-bar.blade.php      # Componente reutilizable
    └── pagination-info.blade.php # Componente reutilizable
```

### Nomenclatura

- **Acciones**: `index`, `show`, `create`, `edit` (verbos CRUD)
- **Sub-componentes**: nombres descriptivos en singular (`product-details`, `order-summary`)
- **Compartidos**: nombres genéricos reutilizables (`search-bar`, `data-table`)

---

## Testing de Componentes Volt

### Feature Tests Obligatorios

```php
<?php
// tests/Feature/Products/ProductIndexTest.php

use App\Models\Product;
use Livewire\Volt\Volt;

it('displays products list', function () {
    $products = Product::factory()->count(3)->create();
    
    Volt::test('products.index')
        ->assertSee($products->first()->name)
        ->assertSee($products->last()->name);
});

it('searches products by name', function () {
    Product::factory()->create(['name' => 'iPhone 15']);
    Product::factory()->create(['name' => 'Samsung Galaxy']);
    
    Volt::test('products.index')
        ->set('search', 'iPhone')
        ->assertSee('iPhone 15')
        ->assertDontSee('Samsung Galaxy');
});

it('creates a new product', function () {
    Volt::test('products.create')
        ->set('name', 'New Product')
        ->set('price', 1000)
        ->call('save')
        ->assertRedirect(route('products.index'));
    
    expect(Product::where('name', 'New Product')->exists())->toBeTrue();
});
```

---

## Convenciones Adicionales

### 1. Props y State

```php
<?php

// ✅ Props tipados (inmutables desde el padre)
use function Livewire\Volt\{state};

state(['product' => fn () => Product::find($this->productId)]);

// ✅ State interno (mutable)
state(['quantity' => 1, 'selectedVariant' => null]);

// ❌ NO mezclar props con state complejo
state(['product' => [], 'relatedProducts' => []]); // EVITAR
```

### 2. Eventos

```php
<?php

// ✅ Emitir eventos con nombres claros
$addToCart = function () {
    // lógica...
    $this->dispatch('product-added-to-cart', productId: $this->product->id);
};

// ✅ Escuchar eventos en otros componentes
use function Livewire\Volt\{on};

on(['product-added-to-cart' => fn ($productId) => $this->refreshCart()]);

// ❌ NO usar eventos para todo (usar props cuando sea posible)
```

### 3. Lazy Loading

```php
<?php
// ✅ Para componentes costosos
use function Livewire\Volt\{state, on};

state(['stats' => null]);

on(['load' => function () {
    $this->stats = expensiveCalculation();
}]);

?>

<div wire:init="$dispatch('load')">
    @if($stats)
        <!-- Contenido -->
    @else
        <div>Cargando...</div>
    @endif
</div>
```

### 4. Placeholder para Carga Diferida

```php
<?php
// resources/views/livewire/products/related-products.blade.php

use function Livewire\Volt\{state, placeholder};

state(['product']);

placeholder(<<<'HTML'
    <div class="animate-pulse">
        <div class="h-4 bg-gray-200 rounded w-3/4"></div>
    </div>
HTML);

?>

<div>
    <!-- Contenido real -->
</div>
```

---

## Manejo de Errores y Validación

### Validación en Tiempo Real

```php
<?php

use function Livewire\Volt\{state, rules};

state(['email' => '']);

rules(['email' => 'required|email|unique:users,email']);

// ✅ Validación en tiempo real para campos específicos
updated(['email' => fn () => $this->validateOnly('email')]);

?>
```

### Mensajes de Error Personalizados

```php
<?php

use function Livewire\Volt\{rules};

rules(['email' => 'required|email'])->messages([
    'email.required' => 'El correo electrónico es obligatorio',
    'email.email' => 'Debe ser un correo válido',
]);

?>
```

---

## Optimización y Performance

### 1. Defer Updates para Campos de Texto

```blade
{{-- ✅ No envía request en cada keystroke --}}
<input type="text" wire:model.defer="search">
<button wire:click="performSearch">Buscar</button>

{{-- ❌ EVITAR en campos de texto largos --}}
<input type="text" wire:model.live="description">
```

### 2. Lazy Loading de Relaciones

```php
<?php

use function Livewire\Volt\{state, computed};

state(['productId']);

// ✅ Cargar solo cuando sea necesario
$product = computed(fn () => Product::with('category', 'variants')->find($this->productId));

// ❌ EVITAR eager loading excesivo
// Product::with('category', 'variants', 'reviews', 'reviews.user', ...)->find($id);

?>
```

### 3. Polling Selectivo

```blade
{{-- ✅ Solo para datos que cambian frecuentemente --}}
<div wire:poll.5s="refreshStats">
    {{ $stats }}
</div>

{{-- ❌ EVITAR polling innecesario --}}
<div wire:poll.1s><!-- contenido estático --></div>
```

---

## Checklist de Calidad

Antes de considerar un componente Volt completo, verificar:

- [ ] ¿El componente tiene menos de 150 líneas?
- [ ] ¿Tiene una sola responsabilidad clara?
- [ ] ¿La lógica de negocio está en Actions?
- [ ] ¿La validación compleja está en FormRequests?
- [ ] ¿Tiene Feature tests con cobertura completa?
- [ ] ¿Los computed properties son simples?
- [ ] ¿Usa defer/lazy donde corresponde?
- [ ] ¿Los nombres de métodos y props son descriptivos?
- [ ] ¿Pasa PHPStan level 6+?
- [ ] ¿Está formateado con Pint?

---

## Recursos Adicionales

- **Documentación oficial**: [Livewire Volt Docs](https://livewire.laravel.com/docs/volt)
- **Convenciones generales**: Ver [`conventions.md`](conventions.md)
- **Arquitectura de módulos**: Ver [`modules.md`](modules.md)

---

## Mantenimiento

- Documentar cualquier cambio de convención en este archivo
- Revisar límites de líneas periódicamente según complejidad del proyecto
- Actualizar ejemplos cuando cambien versiones de Livewire/Volt
