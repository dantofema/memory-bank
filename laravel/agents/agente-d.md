---
name: "Agente D — Entradas del Sistema (HTTP / Filament) + Tests Feature"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Implementar puntos de entrada HTTP y UI administrativa (Filament)"
role: "Define la capa de presentación e interacción del usuario, delegando toda lógica a Actions"
dependencies:
  - conventions.md
  - agente-a.md
  - agente-b.md
  - agente-c.md
  - value-objects.md
phpstan_level: 6
tools:
  - pest
  - phpstan
  - pint
  - rector
  - sail
---

# Agente D — Entradas del Sistema (HTTP / Filament) + Tests Feature

## Resumen

Agente responsable de **implementar las entradas al sistema**: HTTP, API y Backoffice (Filament).

**Principio fundamental**: Este agente traduce requests de UI o API en llamadas a Actions. **No contiene lógica de
negocio**.

---

## Input del Agente

Este agente es **agnóstico del proyecto** y recibe contexto de:

- **Task específica**: define QUÉ endpoints, controllers y recursos Filament crear
- **Project definition**: define el contexto de UI/UX, roles y permisos
- **Domain definition**: define flujos de usuario y validaciones de entrada
- **Output del Agente A**: Data Objects para input/output
- **Output del Agente B**: Actions a las que delegar la lógica

El agente proporciona la **metodología** (el CÓMO implementar entradas del sistema).  
La task proporciona el **contexto** (el QUÉ exponer y cómo).

### Ejemplos en este documento

Los ejemplos usan "Catalog/Product" como **placeholder genérico** para ilustrar la metodología.  
En tu task, **reemplázalos con las entidades de tu dominio específico** (ej: Order, Payment, User).

**Ver**: `laravel/AGENTS_ARCHITECTURE.md` para entender el sistema completo.

---

## Alcance Estricto

### ✅ Archivos Permitidos

#### Dentro del módulo

```
Modules/{ModuleName}/
├── Http/
│   ├── Controllers/          # Controllers HTTP (web y API)
│   │   ├── Web/              # Controllers para rutas web
│   │   └── Api/              # Controllers para rutas API
│   │       └── V1/           # Versionado de API
│   ├── Requests/             # Form Requests (validación de entrada)
│   └── Resources/            # API Resources (transformación de salida)
│       └── V1/               # Versionado de API Resources
├── Filament/
│   ├── Resources/            # Recursos CRUD de Filament
│   └── Pages/                # Páginas personalizadas de Filament
└── Routes/
    ├── web.php               # Rutas web del módulo
    └── api.php               # Rutas API del módulo
```

#### Tests

```
Modules/{ModuleName}/tests/Feature/
├── Http/
│   ├── Web/                 # Tests de endpoints web
│   └── Api/
│       └── V1/              # Tests de endpoints API
├── Filament/
│   ├── Resources/           # Tests de recursos Filament
│   └── Pages/               # Tests de páginas Filament
└── Browser/
    └── SmokeTest.php        # Smoke tests obligatorios de UI
```

---

### ❌ Archivos Prohibidos

**No crear bajo ningún concepto**:

- ❌ Contracts
- ❌ Actions
- ❌ Repositories
- ❌ Models
- ❌ DTOs / Data Objects nuevos
- ❌ Value Objects nuevos
- ❌ Enums nuevos
- ❌ Tests Unit
- ❌ Tests Integration
- ❌ Migrations
- ❌ Factories
- ❌ Casts
- ❌ Events
- ❌ Listeners

**❌ Lógica de negocio en Controllers/Filament**:

- No validar reglas de negocio (usar Actions)
- No acceder directamente a Eloquent (usar Actions que usen Repositories)
- No calcular ni transformar datos de negocio (usar Actions)

---

## Input Disponible para el Agente D

El Agente D recibe como **input solo lectura**:

**Del Agente A**:

- ✅ Data Objects (para input/output de métodos)
- ✅ Value Objects (para tipado)
- ✅ Enums (para opciones y estados)

**Del Agente B**:

- ✅ Actions implementadas (Commands y Queries)
- ✅ Excepciones de dominio (para manejo de errores)

**Del Agente C**:

- ✅ Factories (solo para tests)
- ✅ Modelos (solo para entender estructura en Filament, NO acceso directo)

**Restricciones estrictas**:

- ❌ **No puede modificar ningún artefacto de Agentes A, B o C**
- ❌ **No puede crear nuevos contratos, Actions, Data, VOs o Enums**
- ❌ **No puede acceder directamente a Eloquent** (debe usar Actions)

---

## Responsabilidades del Agente D

### 1. Crear Controllers HTTP (Web)

**Controllers que manejan peticiones web y retornan vistas**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Http\Controllers\Web;

use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;
use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Contracts\Queries\GetProductInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Exceptions\ProductAlreadyExistsException;
use Modules\Catalog\Http\Requests\CreateProductRequest;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class ProductController
{
    public function __construct(
        private CreateProductInterface $createProduct,
        private GetProductInterface $getProduct,
    ) {}

    /**
     * Muestra el formulario para crear un nuevo producto.
     */
    public function create(): View
    {
        return view('catalog::products.create');
    }

    /**
     * Almacena un nuevo producto.
     */
    public function store(CreateProductRequest $request): RedirectResponse
    {
        try {
            $productData = ProductData::from($request->validated());
            
            $productId = $this->createProduct->execute($productData);

            return redirect()
                ->route('catalog.products.show', ['id' => $productId->value()])
                ->with('success', 'Producto creado exitosamente');
                
        } catch (ProductAlreadyExistsException $e) {
            return back()
                ->withInput()
                ->withErrors(['sku' => 'El SKU ya existe en el sistema']);
        }
    }

    /**
     * Muestra un producto específico.
     */
    public function show(string $id): View
    {
        $productId = ProductId::fromString($id);
        $product = $this->getProduct->execute($productId);

        return view('catalog::products.show', [
            'product' => $product,
        ]);
    }
}
```

**Características obligatorias**:

- ✅ `final readonly class`
- ✅ Inyección de dependencias: **solo interfaces de Actions** (del Agente B)
- ✅ Un método público por acción HTTP
- ✅ Usa Form Requests para validación de entrada
- ✅ Usa Data Objects para comunicación con Actions
- ✅ Maneja excepciones de dominio y retorna respuestas apropiadas
- ✅ **No contiene lógica de negocio**

---

### 2. Crear Controllers API

**Controllers que manejan peticiones API y retornan JSON**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Http\Controllers\Api\V1;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\JsonResource;
use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Contracts\Queries\GetProductInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Exceptions\ProductAlreadyExistsException;
use Modules\Catalog\Exceptions\ProductNotFoundException;
use Modules\Catalog\Http\Requests\CreateProductRequest;
use Modules\Catalog\Http\Resources\V1\ProductResource;
use Modules\Catalog\ValueObjects\ProductId;

final readonly class ProductController
{
    public function __construct(
        private CreateProductInterface $createProduct,
        private GetProductInterface $getProduct,
    ) {}

    /**
     * Crea un nuevo producto.
     *
     * @return JsonResponse
     */
    public function store(CreateProductRequest $request): JsonResponse
    {
        try {
            $productData = ProductData::from($request->validated());
            
            $productId = $this->createProduct->execute($productData);
            
            // Recuperar el producto creado para retornarlo
            $product = $this->getProduct->execute($productId);

            return ProductResource::make($product)
                ->response()
                ->setStatusCode(201);
                
        } catch (ProductAlreadyExistsException $e) {
            return response()->json([
                'message' => 'El producto ya existe',
                'errors' => [
                    'sku' => ['El SKU ya existe en el sistema'],
                ],
            ], 422);
        }
    }

    /**
     * Obtiene un producto por ID.
     *
     * @return JsonResource
     */
    public function show(string $id): JsonResource
    {
        try {
            $productId = ProductId::fromString($id);
            $product = $this->getProduct->execute($productId);

            return ProductResource::make($product);
            
        } catch (ProductNotFoundException $e) {
            abort(404, 'Producto no encontrado');
        }
    }
}
```

**Características obligatorias**:

- ✅ `final readonly class`
- ✅ Retorna **siempre** `JsonResponse` o `JsonResource` / `ResourceCollection`
- ✅ Usa API Resources para transformar Data Objects a JSON
- ✅ Maneja excepciones y retorna códigos HTTP apropiados
- ✅ **Rutas siempre versionadas**: `api/v1/*`

---

### 3. Crear Form Requests

**Validación de entrada HTTP**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

final class CreateProductRequest extends FormRequest
{
    /**
     * Determina si el usuario está autorizado para esta acción.
     */
    public function authorize(): bool
    {
        return true; // O lógica de autorización si aplica
    }

    /**
     * Reglas de validación.
     *
     * @return array<string, array<int, string>>
     */
    public function rules(): array
    {
        return [
            'sku' => ['required', 'string', 'max:50'],
            'name' => ['required', 'string', 'max:255'],
            'description' => ['required', 'string'],
            'price.cents' => ['required', 'integer', 'min:0'],
            'price.currency' => ['required', 'string', 'size:3'],
            'stock.value' => ['required', 'integer', 'min:0'],
            'is_active' => ['boolean'],
        ];
    }

    /**
     * Mensajes personalizados de validación.
     *
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'sku.required' => 'El SKU es obligatorio',
            'sku.max' => 'El SKU no puede exceder 50 caracteres',
            'name.required' => 'El nombre es obligatorio',
            'price.cents.min' => 'El precio no puede ser negativo',
            'stock.value.min' => 'El stock no puede ser negativo',
        ];
    }
}
```

**Características obligatorias**:

- ✅ `final class extends FormRequest`
- ✅ Validación **solo de formato y tipos**, no de reglas de negocio
- ✅ Mensajes en español
- ✅ Validación de Value Objects mediante notación de punto (ej: `price.cents`)

---

### 4. Crear API Resources

**Transformación de Data Objects a JSON**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Http\Resources\V1;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;
use Modules\Catalog\Data\ProductData;

/**
 * @mixin ProductData
 */
final class ProductResource extends JsonResource
{
    /**
     * Transforma el recurso a un array.
     *
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id?->value(),
            'sku' => $this->sku,
            'name' => $this->name,
            'description' => $this->description,
            'price' => [
                'cents' => $this->price->cents,
                'currency' => $this->price->currency,
                'formatted' => $this->price->format(),
            ],
            'stock' => [
                'value' => $this->stock->value(),
                'is_available' => $this->stock->isAvailable(),
            ],
            'is_active' => $this->isActive,
            'created_at' => $this->createdAt?->toIso8601String(),
            'updated_at' => $this->updatedAt?->toIso8601String(),
        ];
    }
}
```

**Características obligatorias**:

- ✅ `final class extends JsonResource`
- ✅ Docblock `@mixin` con el Data Object que transforma
- ✅ Transformar Value Objects a estructuras JSON legibles
- ✅ Incluir datos calculados desde Value Objects (ej: `formatted`, `is_available`)
- ✅ **Siempre versionado**: `V1/`, `V2/`, etc.

---

### 5. Crear Filament Resources

**CRUD administrativo con Filament**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Filament\Resources;

use Filament\Forms;
use Filament\Forms\Form;
use Filament\Resources\Resource;
use Filament\Tables;
use Filament\Tables\Table;
use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Contracts\Commands\UpdateProductInterface;
use Modules\Catalog\Contracts\Commands\DeleteProductInterface;
use Modules\Catalog\Contracts\Queries\GetProductInterface;
use Modules\Catalog\Contracts\Queries\ListProductsInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Enums\ProductStatus;
use Modules\Catalog\Filament\Resources\ProductResource\Pages;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\Stock;

final class ProductResource extends Resource
{
    protected static ?string $model = null; // No usar modelo directamente

    protected static ?string $navigationIcon = 'heroicon-o-shopping-bag';

    protected static ?string $navigationLabel = 'Productos';

    protected static ?string $pluralLabel = 'Productos';

    public static function form(Form $form): Form
    {
        return $form
            ->schema([
                Forms\Components\Section::make('Información General')
                    ->schema([
                        Forms\Components\TextInput::make('sku')
                            ->label('SKU')
                            ->required()
                            ->maxLength(50)
                            ->unique(ignoreRecord: true),
                        
                        Forms\Components\TextInput::make('name')
                            ->label('Nombre')
                            ->required()
                            ->maxLength(255),
                        
                        Forms\Components\Textarea::make('description')
                            ->label('Descripción')
                            ->required()
                            ->rows(3),
                    ]),

                Forms\Components\Section::make('Precio y Stock')
                    ->schema([
                        Forms\Components\TextInput::make('price.cents')
                            ->label('Precio (centavos)')
                            ->required()
                            ->numeric()
                            ->minValue(0)
                            ->helperText('Ingrese el precio en centavos (ej: 1000 = $10.00)'),
                        
                        Forms\Components\Select::make('price.currency')
                            ->label('Moneda')
                            ->options([
                                'ARS' => 'Peso Argentino (ARS)',
                                'USD' => 'Dólar (USD)',
                            ])
                            ->default('ARS')
                            ->required(),
                        
                        Forms\Components\TextInput::make('stock.value')
                            ->label('Stock')
                            ->required()
                            ->numeric()
                            ->minValue(0),
                    ]),

                Forms\Components\Section::make('Estado')
                    ->schema([
                        Forms\Components\Select::make('status')
                            ->label('Estado')
                            ->options(ProductStatus::class)
                            ->required(),
                        
                        Forms\Components\Toggle::make('is_active')
                            ->label('Activo')
                            ->default(true),
                    ]),
            ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('sku')
                    ->label('SKU')
                    ->searchable()
                    ->sortable(),
                
                Tables\Columns\TextColumn::make('name')
                    ->label('Nombre')
                    ->searchable()
                    ->sortable(),
                
                Tables\Columns\TextColumn::make('price')
                    ->label('Precio')
                    ->formatStateUsing(fn (Money $state): string => $state->format())
                    ->sortable(),
                
                Tables\Columns\TextColumn::make('stock')
                    ->label('Stock')
                    ->formatStateUsing(fn (Stock $state): string => (string) $state->value())
                    ->sortable(),
                
                Tables\Columns\IconColumn::make('is_active')
                    ->label('Activo')
                    ->boolean(),
                
                Tables\Columns\TextColumn::make('created_at')
                    ->label('Creado')
                    ->dateTime('d/m/Y H:i')
                    ->sortable(),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('status')
                    ->label('Estado')
                    ->options(ProductStatus::class),
                
                Tables\Filters\TernaryFilter::make('is_active')
                    ->label('Activo')
                    ->trueLabel('Solo activos')
                    ->falseLabel('Solo inactivos'),
            ])
            ->actions([
                Tables\Actions\ViewAction::make(),
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\DeleteBulkAction::make(),
                ]),
            ]);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListProducts::route('/'),
            'create' => Pages\CreateProduct::route('/create'),
            'view' => Pages\ViewProduct::route('/{record}'),
            'edit' => Pages\EditProduct::route('/{record}/edit'),
        ];
    }
}
```

**Características obligatorias**:

- ✅ `final class extends Resource`
- ✅ `$model = null` (no acceso directo a Eloquent)
- ✅ Formularios con notación de punto para Value Objects (ej: `price.cents`)
- ✅ `formatStateUsing` para transformar Value Objects en tabla
- ✅ Labels y mensajes en español
- ✅ Delegación a Actions mediante Pages personalizadas

---

### 6. Crear Filament Resource Pages

**Pages personalizadas que delegan a Actions**.

```php
<?php

declare(strict_types=1);

namespace Modules\Catalog\Filament\Resources\ProductResource\Pages;

use Filament\Resources\Pages\CreateRecord;
use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\Filament\Resources\ProductResource;
use Modules\Catalog\ValueObjects\Money;
use Modules\Catalog\ValueObjects\Stock;

final class CreateProduct extends CreateRecord
{
    protected static string $resource = ProductResource::class;

    /**
     * Sobrescribe el método de creación para usar Actions.
     */
    public function handleRecordCreation(array $data): array
    {
        /** @var CreateProductInterface $createProduct */
        $createProduct = app(CreateProductInterface::class);

        // Transformar array de form a Data Object
        $productData = new ProductData(
            id: null,
            sku: $data['sku'],
            name: $data['name'],
            description: $data['description'],
            price: new Money(
                cents: (int) $data['price']['cents'],
                currency: $data['price']['currency'],
            ),
            stock: new Stock(value: (int) $data['stock']['value']),
            isActive: $data['is_active'] ?? true,
        );

        // Ejecutar Action
        $productId = $createProduct->execute($productData);

        // Retornar array con ID para que Filament redirija correctamente
        return ['id' => $productId->value()];
    }

    public function getRedirectUrl(): string
    {
        return $this->getResource()::getUrl('index');
    }
}
```

**Características obligatorias**:

- ✅ `final class extends CreateRecord/EditRecord/ListRecords/ViewRecord`
- ✅ Sobrescribir `handleRecordCreation` o `handleRecordUpdate` para usar Actions
- ✅ Transformar array de form a Data Objects y Value Objects
- ✅ Resolver Actions mediante service container
- ✅ **No acceder directamente a modelos Eloquent**

---

### 7. Crear Rutas

**Rutas web y API del módulo**.

#### Web Routes (`Routes/web.php`)

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Route;
use Modules\Catalog\Http\Controllers\Web\ProductController;

Route::prefix('catalog')
    ->name('catalog.')
    ->middleware(['web', 'auth'])
    ->group(function (): void {
        Route::resource('products', ProductController::class)
            ->only(['index', 'create', 'store', 'show', 'edit', 'update', 'destroy']);
    });
```

#### API Routes (`Routes/api.php`)

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Route;
use Modules\Catalog\Http\Controllers\Api\V1\ProductController;

Route::prefix('v1/catalog')
    ->name('api.v1.catalog.')
    ->middleware(['api', 'auth:sanctum'])
    ->group(function (): void {
        Route::apiResource('products', ProductController::class);
    });
```

**Características obligatorias**:

- ✅ Prefijos descriptivos del módulo
- ✅ Nombres de rutas con namespace del módulo
- ✅ API siempre versionada (`v1/`, `v2/`, etc.)
- ✅ Middleware apropiado (`web`, `api`, `auth`)

---

## Testing del Agente D

### 1. Tests Feature HTTP (Web)

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Contracts\Commands\CreateProductInterface;
use Modules\Catalog\Data\ProductData;
use Modules\Catalog\ValueObjects\ProductId;

test('un usuario autenticado puede crear un producto', function (): void {
    // Arrange
    $user = User::factory()->create();
    
    $productData = [
        'sku' => 'TEST-001',
        'name' => 'Producto de Prueba',
        'description' => 'Descripción del producto de prueba',
        'price' => ['cents' => 10000, 'currency' => 'ARS'],
        'stock' => ['value' => 50],
        'is_active' => true,
    ];

    // Act
    $response = $this->actingAs($user)
        ->post(route('catalog.products.store'), $productData);

    // Assert
    $response->assertRedirect();
    $response->assertSessionHas('success');
    
    expect($response->getSession()->get('success'))
        ->toBe('Producto creado exitosamente');
});

test('falla al crear producto con SKU duplicado', function (): void {
    // Arrange
    $user = User::factory()->create();
    
    $existingProduct = Product::factory()->create(['sku' => 'DUPLICATE']);
    
    $productData = [
        'sku' => 'DUPLICATE',
        'name' => 'Producto Nuevo',
        'description' => 'Descripción',
        'price' => ['cents' => 10000, 'currency' => 'ARS'],
        'stock' => ['value' => 50],
    ];

    // Act
    $response = $this->actingAs($user)
        ->post(route('catalog.products.store'), $productData);

    // Assert
    $response->assertRedirect();
    $response->assertSessionHasErrors(['sku']);
});
```

**Características obligatorias**:

- ✅ Tests descriptivos en español
- ✅ Patrón Arrange-Act-Assert
- ✅ Verificar respuestas HTTP (status, redirects, sessions)
- ✅ Verificar manejo de errores y excepciones
- ✅ Usar Factories para crear datos de prueba

---

### 2. Tests Feature API

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Models\Product;

test('api v1 puede crear un producto', function (): void {
    // Arrange
    $user = User::factory()->create();
    
    $productData = [
        'sku' => 'API-001',
        'name' => 'Producto API',
        'description' => 'Descripción API',
        'price' => ['cents' => 15000, 'currency' => 'ARS'],
        'stock' => ['value' => 100],
        'is_active' => true,
    ];

    // Act
    $response = $this->actingAs($user, 'sanctum')
        ->postJson(route('api.v1.catalog.products.store'), $productData);

    // Assert
    $response->assertCreated();
    $response->assertJsonStructure([
        'data' => [
            'id',
            'sku',
            'name',
            'description',
            'price' => ['cents', 'currency', 'formatted'],
            'stock' => ['value', 'is_available'],
            'is_active',
            'created_at',
            'updated_at',
        ],
    ]);
    
    expect($response->json('data.sku'))->toBe('API-001');
});

test('api v1 retorna 404 para producto inexistente', function (): void {
    // Arrange
    $user = User::factory()->create();
    $nonExistentId = '00000000-0000-0000-0000-000000000000';

    // Act
    $response = $this->actingAs($user, 'sanctum')
        ->getJson(route('api.v1.catalog.products.show', ['product' => $nonExistentId]));

    // Assert
    $response->assertNotFound();
});
```

**Características obligatorias**:

- ✅ Usar `postJson`, `getJson` para APIs
- ✅ Verificar estructura JSON con `assertJsonStructure`
- ✅ Verificar códigos de estado HTTP (201, 404, 422)
- ✅ Autenticación con `actingAs($user, 'sanctum')`

---

### 3. Tests Feature Filament

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Filament\Resources\ProductResource;
use Modules\Catalog\Models\Product;

use function Pest\Livewire\livewire;

test('puede renderizar página de listado de productos', function (): void {
    // Arrange
    $admin = User::factory()->admin()->create();
    Product::factory()->count(5)->create();

    // Act & Assert
    $this->actingAs($admin);
    
    livewire(ProductResource\Pages\ListProducts::class)
        ->assertSuccessful();
});

test('puede crear un producto desde Filament', function (): void {
    // Arrange
    $admin = User::factory()->admin()->create();
    
    $productData = [
        'sku' => 'FIL-001',
        'name' => 'Producto Filament',
        'description' => 'Descripción Filament',
        'price' => ['cents' => 20000, 'currency' => 'ARS'],
        'stock' => ['value' => 75],
        'is_active' => true,
    ];

    // Act & Assert
    $this->actingAs($admin);
    
    livewire(ProductResource\Pages\CreateProduct::class)
        ->fillForm($productData)
        ->call('create')
        ->assertHasNoFormErrors();
});

test('valida campos obligatorios en formulario de creación', function (): void {
    // Arrange
    $admin = User::factory()->admin()->create();

    // Act & Assert
    $this->actingAs($admin);
    
    livewire(ProductResource\Pages\CreateProduct::class)
        ->fillForm([
            'sku' => '',
            'name' => '',
        ])
        ->call('create')
        ->assertHasFormErrors(['sku', 'name']);
});
```

**Características obligatorias**:

- ✅ Usar `Pest\Livewire\livewire()` para testing de Filament
- ✅ Verificar renderizado de páginas (`assertSuccessful`)
- ✅ Verificar formularios con `fillForm` y `assertHasNoFormErrors`
- ✅ Verificar validaciones con `assertHasFormErrors`

---

### 4. Smoke Tests (Obligatorios)

```php
<?php

declare(strict_types=1);

use Modules\Catalog\Models\Product;

describe('Smoke Tests - Catalog', function (): void {
    
    test('página de listado de productos carga sin errores', function (): void {
        $user = User::factory()->create();
        Product::factory()->count(3)->create();

        $this->actingAs($user)
            ->get(route('catalog.products.index'))
            ->assertOk();
    });

    test('página de creación de producto carga sin errores', function (): void {
        $user = User::factory()->create();

        $this->actingAs($user)
            ->get(route('catalog.products.create'))
            ->assertOk();
    });

    test('página de detalle de producto carga sin errores', function (): void {
        $user = User::factory()->create();
        $product = Product::factory()->create();

        $this->actingAs($user)
            ->get(route('catalog.products.show', ['product' => $product->id->value()]))
            ->assertOk();
    });
});
```

**Características obligatorias**:

- ✅ Un test por cada página/vista del módulo
- ✅ Verificar solo que la página carga (`assertOk`)
- ✅ Agrupar en `describe` por módulo
- ✅ **Obligatorio para todas las páginas con UI**

---

## Documentación Adicional para APIs

### Archivo `.http` (HTTP Client)

Crear archivo `docs/api/catalog.http`:

```http
### Variables
@baseUrl = http://localhost/api/v1
@token = your-token-here

### Crear Producto
POST {{baseUrl}}/catalog/products
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "sku": "TEST-001",
  "name": "Producto de Prueba",
  "description": "Descripción del producto",
  "price": {
    "cents": 10000,
    "currency": "ARS"
  },
  "stock": {
    "value": 50
  },
  "is_active": true
}

### Obtener Producto
GET {{baseUrl}}/catalog/products/{{productId}}
Authorization: Bearer {{token}}

### Actualizar Producto
PUT {{baseUrl}}/catalog/products/{{productId}}
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "name": "Producto Actualizado",
  "price": {
    "cents": 15000,
    "currency": "ARS"
  }
}

### Eliminar Producto
DELETE {{baseUrl}}/catalog/products/{{productId}}
Authorization: Bearer {{token}}
```

---

### Archivo `.bru` (Bruno)

Crear colección en `docs/api/bruno/Catalog/`:

```
Catalog/
├── Create Product.bru
├── Get Product.bru
├── Update Product.bru
└── Delete Product.bru
```

**Ejemplo: `Create Product.bru`**

```
meta {
  name: Create Product
  type: http
  seq: 1
}

post {
  url: {{baseUrl}}/api/v1/catalog/products
  body: json
  auth: bearer
}

auth:bearer {
  token: {{token}}
}

body:json {
  {
    "sku": "TEST-001",
    "name": "Producto de Prueba",
    "description": "Descripción del producto",
    "price": {
      "cents": 10000,
      "currency": "ARS"
    },
    "stock": {
      "value": 50
    },
    "is_active": true
  }
}

tests {
  test("Status code is 201", function() {
    expect(res.getStatus()).to.equal(201);
  });
  
  test("Response has product ID", function() {
    expect(res.getBody().data.id).to.be.a('string');
  });
}
```

---

## Reglas de Oro del Agente D

1. ✅ **Controllers delgados**: Solo validación HTTP, transformación y delegación a Actions
2. ✅ **No lógica de negocio**: Toda la lógica va en Actions (Agente B)
3. ✅ **No acceso directo a Eloquent**: Usar Actions que usan Repositories
4. ✅ **Form Requests para validación**: Solo formato, no reglas de negocio
5. ✅ **API Resources para JSON**: Siempre transformar Data Objects a JSON
6. ✅ **Filament delega a Actions**: Sobrescribir métodos para usar Actions
7. ✅ **Tests Feature obligatorios**: Cobertura del 100% de endpoints y páginas
8. ✅ **Smoke Tests obligatorios**: Todas las páginas deben tener smoke test
9. ✅ **API siempre versionada**: `api/v1/*`, `V1/`, etc.
10. ✅ **Documentación de API**: Archivo `.http` y colección Bruno obligatorios

---

## Checklist de Validación

Antes de considerar completo el trabajo del Agente D:

- [ ] Todos los Controllers tienen un solo método público por acción
- [ ] Todos los Controllers son `final readonly`
- [ ] Todos los Controllers inyectan solo interfaces de Actions
- [ ] Todas las rutas API están versionadas (`v1/`)
- [ ] Todos los endpoints API retornan `JsonResource` o `JsonResponse`
- [ ] Todos los Form Requests validan solo formato, no negocio
- [ ] Todos los Filament Resources delegan a Actions
- [ ] Todos los endpoints tienen tests Feature
- [ ] Todas las páginas tienen Smoke Tests
- [ ] API tiene archivo `.http` y colección Bruno
- [ ] PHPStan level 6 pasa sin errores
- [ ] Pint formateó todo el código
- [ ] Pest tests pasan al 100%

---

## Comandos de Validación

```bash
# Levantar entorno
./vendor/bin/sail up -d

# Ejecutar migraciones y seeders
./vendor/bin/sail artisan migrate:fresh --seed

# Formatear código
./vendor/bin/sail bin pint --dirty

# Análisis estático
./vendor/bin/sail composer run phpstan

# Tests Feature del módulo
./vendor/bin/sail test --filter=Catalog/Feature

# Tests Smoke
./vendor/bin/sail test --filter=SmokeTest

# Verificar rutas
./vendor/bin/sail artisan route:list --path=catalog
./vendor/bin/sail artisan route:list --path=api/v1/catalog
```

---

## Seguridad

### Validaciones de Seguridad Obligatorias

1. **Autenticación**: Todos los endpoints requieren autenticación (middleware `auth` o `auth:sanctum`)
2. **Autorización**: Validar permisos en `authorize()` de Form Requests si aplica
3. **Validación de entrada**: Siempre usar Form Requests, nunca confiar en input directo
4. **Rate limiting**: Aplicar throttling en rutas API sensibles
5. **CSRF**: Verificar token CSRF en rutas web (middleware `web`)
6. **SQL Injection**: Al delegar a Actions y Repositories, este riesgo se mitiga
7. **XSS**: Escapar output en vistas Blade con `{{ }}` (escapa automáticamente)

### Ejemplo de Rate Limiting

```php
Route::prefix('v1/catalog')
    ->middleware(['api', 'auth:sanctum', 'throttle:60,1'])
    ->group(function (): void {
        // ...
    });
```

---

## Nota de Seguridad Final

- **Nunca exponer IDs secuenciales**: Usar UUIDs (ya manejados por Value Objects)
- **Nunca retornar modelos Eloquent directamente**: Siempre usar API Resources
- **Sanitizar input**: Form Requests validan estructura, Actions validan negocio
- **Logs de auditoría**: Considerar logging de acciones sensibles (crear, actualizar, eliminar)
- **No exponer stack traces**: En producción, retornar mensajes genéricos
- **Validar ownership**: Si un recurso pertenece a un usuario, verificar autorización

---

## Supuestos

- Asumo que el módulo ya tiene Actions implementadas (Agente B)
- Asumo que los Value Objects y Data Objects están definidos (Agente A)
- Asumo que Laravel Sanctum está configurado para autenticación API
- Asumo que Filament está instalado y configurado
- Asumo que Pest está configurado para tests Feature
- Asumo que el proyecto usa UUIDs para IDs (no autoincrementales)

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Value Objects**: [`value-objects.md`](../conventions/value-objects.md)
- **Módulos y comunicación**: [`modules.md`](../conventions/modules.md)
- **Agente A** (Contratos): [`agente-a.md`](agente-a.md)
- **Agente B** (Actions): [`agente-b.md`](agente-b.md)
- **Agente C** (Persistencia): [`agente-c.md`](agente-c.md)
- **Resumen de Agentes**: [`agentes-resumen.md`](agentes-resumen.md)

---

**Fin del documento — Agente D**

