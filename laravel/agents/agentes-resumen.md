---
name: "Resumen de Agentes - División de Responsabilidades"
version: "1.0"
author: "Alejandro Leone"
last_updated: "2025-12-17"
purpose: "Documentar la división de responsabilidades entre agentes especializados"
---

# Resumen de Agentes - División de Responsabilidades

## Visión General

La implementación de un módulo se divide en **agentes especializados**, cada uno con un alcance estricto y
responsabilidades claramente definidas.

**Principio fundamental**: Cada agente produce artefactos que el siguiente agente consume, pero **nunca modifica**.

---

## Flujo de Trabajo

```
Agente A (Contratos) 
    ↓
Agente B (Actions)
    ↓
Agente C (Persistencia)
    ↓
Agente D (Controllers/Filament/Livewire)
    ↓
Agente E (Events/Listeners)
```

---

## Agente A — Contratos, Data, Enums y Value Objects

### 🎯 Propósito

Definir la **frontera pública del módulo** mediante tipos y contratos.

### 📁 Archivos Permitidos

- ✅ `Contracts/Commands/*` - Interfaces síncronas que modifican estado
- ✅ `Contracts/Queries/*` - Interfaces síncronas de solo lectura
- ✅ `Contracts/Repositories/*` - Interfaces de repositorios
- ✅ `Data/*` - Spatie Laravel Data objects
- ✅ `Enums/*` - Enumeraciones PHP
- ✅ `ValueObjects/*` - Value Objects con reglas de negocio
- ✅ `tests/Unit/ValueObjects/*` - Tests de VOs
- ✅ `tests/Unit/Enums/*` - Tests de Enums

### ⛔ Archivos Prohibidos

- ❌ Actions
- ❌ Services
- ❌ Models
- ❌ Repositories concretos
- ❌ Controllers
- ❌ Filament
- ❌ Events/Listeners
- ❌ Migrations
- ❌ Factories
- ❌ Casts

### 📤 Salida (Output)

Interfaces, Data Objects, Value Objects, Enums → **Consumidos por Agente B**

### 📖 Referencia

[`agent-contracts.md`](agent-a-contracts.md)

---

## Agente B — Actions y Tests Unitarios

### 🎯 Propósito

Implementar los **casos de uso del módulo** (comportamiento del negocio).

### 📁 Archivos Permitidos

- ✅ `Actions/Commands/*` - Implementaciones de Commands
- ✅ `Actions/Queries/*` - Implementaciones de Queries
- ✅ `Actions/Internal/*` - Actions auxiliares privadas
- ✅ `Exceptions/*` - Excepciones de dominio
- ✅ `tests/Unit/Actions/*` - Tests unitarios con mocks

### ⛔ Archivos Prohibidos

- ❌ Models
- ❌ Repositories concretos
- ❌ Controllers
- ❌ Filament
- ❌ Events/Listeners
- ❌ Migrations
- ❌ Factories
- ❌ Queries SQL directas
- ❌ Acceso a DB
- ❌ Código HTTP
- ❌ Value Objects nuevos
- ❌ Contratos nuevos
- ❌ Casts

### 📥 Entrada (Input)

Del Agente A:

- ✅ Contratos (Interfaces)
- ✅ Data Objects
- ✅ Value Objects
- ✅ Enums

**Restricción**: ❌ NO puede modificar nada del Agente A

### 📤 Salida (Output)

Actions implementadas, Excepciones de dominio → **Consumidos por Agente C y posteriores**

### 📖 Referencia

[`agent-actions.md`](agent-b-actions.md)

---

## Agente C — Repositorios, Modelos Eloquent e Infraestructura de Persistencia

### 🎯 Propósito

Implementar la **capa de persistencia** y acceso a datos del módulo.

### 📁 Archivos Permitidos

- ✅ `Models/*` - Modelos Eloquent
- ✅ `Repositories/*` - Implementaciones de repositorios
- ✅ `Casts/*` - Eloquent Casts para Value Objects
- ✅ `Factories/*` - Factories de modelos
- ✅ `Database/Migrations/*` - Migraciones
- ✅ `tests/Unit/Models/*` - Tests de modelos
- ✅ `tests/Unit/Casts/*` - Tests de Casts
- ✅ `tests/Feature/Repositories/*` - Tests de integración con DB

### ⛔ Archivos Prohibidos

- ❌ Actions (ya creadas por Agente B)
- ❌ Controllers
- ❌ Filament
- ❌ Events/Listeners
- ❌ Jobs
- ❌ Value Objects nuevos
- ❌ Contratos nuevos
- ❌ Modificar Actions existentes
- ❌ Lógica de negocio (va en Actions)

### 📥 Entrada (Input)

Del Agente A:

- ✅ Contratos de repositorio
- ✅ Data Objects
- ✅ Value Objects
- ✅ Enums

Del Agente B:

- ✅ Actions (para entender flujos)
- ✅ Excepciones de dominio

**Restricción**: ❌ NO puede modificar nada de Agentes A y B

### 📤 Salida (Output)

Modelos Eloquent, Repositorios, Migraciones, Factories → **Consumidos por Agente D y posteriores**

### 📖 Referencia

[`agent-persistence.md`](agent-c-persistence.md)

---

## Agente D — Controllers y Filament Resources (Puntos de Entrada)

### 🎯 Propósito

Implementar **puntos de entrada HTTP** y **UI administrativa**.

### 📁 Archivos Permitidos

- ✅ `Http/Controllers/*` - Controllers HTTP
- ✅ `Filament/Resources/*` - Recursos de Filament
- ✅ `Filament/Pages/*` - Páginas personalizadas
- ✅ `Http/Requests/*` - Form Requests
- ✅ `Http/Resources/*` - API Resources
- ✅ `tests/Feature/Http/*` - Tests de endpoints
- ✅ `tests/Feature/Filament/*` - Tests de UI

### ⛔ Archivos Prohibidos

- ❌ Lógica de negocio (debe delegar a Actions)
- ❌ Acceso directo a Eloquent (usar repositorios)
- ❌ Value Objects nuevos
- ❌ Contratos nuevos
- ❌ Actions nuevas
- ❌ Migrations

### 📥 Entrada (Input)

Del Agente A:

- ✅ Data Objects (para input/output)

Del Agente B:

- ✅ Actions (para ejecutar casos de uso)
- ✅ Excepciones de dominio

Del Agente C:

- ✅ Factories (para tests)

**Restricción**: ❌ NO puede modificar nada de Agentes A, B y C

### 📤 Salida (Output)

Endpoints HTTP funcionales, UI administrativa → **Disponible para usuarios**

### 📖 Referencia

[`agent-http.md`](agent-d-http.md)

---

## Agente E — Events, Listeners y Jobs (Efectos Secundarios)

### 🎯 Propósito

Implementar **efectos secundarios** y **comunicación asíncrona** entre módulos.

### 📁 Archivos Permitidos

- ✅ `Events/*` - Eventos de dominio
- ✅ `Listeners/*` - Listeners de eventos
- ✅ `Jobs/*` - Jobs asíncronos
- ✅ `tests/Unit/Events/*` - Tests de eventos
- ✅ `tests/Feature/Listeners/*` - Tests de listeners

### ⛔ Archivos Prohibidos

- ❌ Lógica de negocio (debe delegar a Actions)
- ❌ Acceso directo a Eloquent (usar repositorios)
- ❌ Value Objects nuevos
- ❌ Contratos nuevos
- ❌ Actions nuevas
- ❌ Migrations

### 📥 Entrada (Input)

Del Agente A:

- ✅ Data Objects

Del Agente B:

- ✅ Actions (para ejecutar desde listeners)

Del Agente C:

- ✅ Repositorios (si necesita persistir)

**Restricción**: ❌ NO puede modificar nada de Agentes A, B y C

### 📤 Salida (Output)

Sistema de eventos funcional → **Comunicación entre módulos**

### 📖 Referencia

(Pendiente: `agent-events.md`)

---

## Reglas Universales

### ✅ Todos los Agentes DEBEN

1. **PHPStan level 6** sin errores
2. **Pint** ejecutado antes de commit
3. **Rector** ejecutado cuando aplique
4. **Tests con cobertura del 100%**
5. **Tipado fuerte**: sin `mixed`, `array` o tipos genéricos
6. **final class** para todas las clases (sin herencia)
7. **readonly** cuando aplique (Value Objects, Actions, Repositories)
8. **declare(strict_types=1)** en todos los archivos PHP

---

### ❌ Ningún Agente PUEDE

1. Modificar archivos creados por agentes anteriores
2. Crear lógica de negocio fuera de Actions (Agente B)
3. Exponer modelos Eloquent fuera del módulo
4. Usar arrays o tipos primitivos en signatures públicas
5. Crear Value Objects o Contratos fuera del Agente A
6. Acceder directamente a DB fuera de Repositorios (Agente C)

---

## Tabla Resumen: ¿Quién Hace Qué?

| Artefacto                  | Agente A | Agente B     | Agente C     | Agente D | Agente E |
|----------------------------|----------|--------------|--------------|----------|----------|
| **Contratos (Interfaces)** | ✅ Crea   | ❌            | ❌            | ❌        | ❌        |
| **Data Objects**           | ✅ Crea   | ❌            | ❌            | ❌        | ❌        |
| **Value Objects**          | ✅ Crea   | ❌            | ❌            | ❌        | ❌        |
| **Enums**                  | ✅ Crea   | ❌            | ❌            | ❌        | ❌        |
| **Actions**                | ❌        | ✅ Implementa | ❌            | ❌        | ❌        |
| **Excepciones**            | ❌        | ✅ Crea       | ❌            | ❌        | ❌        |
| **Modelos Eloquent**       | ❌        | ❌            | ✅ Crea       | ❌        | ❌        |
| **Repositories**           | ❌        | ❌            | ✅ Implementa | ❌        | ❌        |
| **Casts**                  | ❌        | ❌            | ✅ Crea       | ❌        | ❌        |
| **Migrations**             | ❌        | ❌            | ✅ Crea       | ❌        | ❌        |
| **Factories**              | ❌        | ❌            | ✅ Crea       | ❌        | ❌        |
| **Controllers**            | ❌        | ❌            | ❌            | ✅ Crea   | ❌        |
| **Filament Resources**     | ❌        | ❌            | ❌            | ✅ Crea   | ❌        |
| **Form Requests**          | ❌        | ❌            | ❌            | ✅ Crea   | ❌        |
| **API Resources**          | ❌        | ❌            | ❌            | ✅ Crea   | ❌        |
| **Events**                 | ❌        | ❌            | ❌            | ❌        | ✅ Crea   |
| **Listeners**              | ❌        | ❌            | ❌            | ❌        | ✅ Crea   |
| **Jobs**                   | ❌        | ❌            | ❌            | ❌        | ✅ Crea   |

---

## Flujo de Dependencias

```
┌─────────────────────────────────────────────────────────────┐
│ Agente A: Contratos, Data, Value Objects, Enums            │
│ Define: QUÉ puede hacer el módulo                          │
└────────────────────────┬────────────────────────────────────┘
                         │ (consume)
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Agente B: Actions                                           │
│ Implementa: CÓMO se comporta el negocio                    │
└────────────────────────┬────────────────────────────────────┘
                         │ (consume)
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Agente C: Models, Repositories, Migrations                  │
│ Implementa: CÓMO se persiste y recupera información         │
└────────────────────────┬────────────────────────────────────┘
                         │ (consume)
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Agente D: Controllers, Filament                             │
│ Implementa: CÓMO se expone al usuario                       │
└────────────────────────┬────────────────────────────────────┘
                         │ (consume)
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Agente E: Events, Listeners, Jobs                           │
│ Implementa: EFECTOS SECUNDARIOS del negocio                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Ventajas de Esta División

### ✅ Claridad

Cada agente tiene un propósito único y bien definido.

### ✅ Mantenibilidad

Cambios en una capa no afectan otras capas.

### ✅ Testabilidad

Cada agente puede ser testeado independientemente con la estrategia correcta:

- **Agente A**: Unit tests (Value Objects, Enums)
- **Agente B**: Unit tests con mocks (Actions)
- **Agente C**: Feature tests con DB (Repositories)
- **Agente D**: Feature tests HTTP (Controllers/Filament)
- **Agente E**: Feature tests (Listeners/Jobs)

### ✅ Escalabilidad

Nuevas funcionalidades siguen el mismo patrón predecible.

### ✅ Colaboración

Múltiples desarrolladores pueden trabajar en paralelo sin conflictos:

- Developer 1 → Agente A + B
- Developer 2 → Agente C
- Developer 3 → Agente D + E

---

## Ejemplo Completo: Crear Producto

### 1. Agente A define el contrato

```php
interface CreateProductInterface
{
    public function execute(ProductData $data): ProductId;
}
```

### 2. Agente B implementa la lógica de negocio

```php
final class CreateProductAction implements CreateProductInterface
{
    public function execute(ProductData $data): ProductId
    {
        if ($this->repository->existsBySku($data->sku)) {
            throw new ProductAlreadyExistsException();
        }
        
        return $this->repository->create($data);
    }
}
```

### 3. Agente C implementa la persistencia

```php
final class ProductRepository implements ProductRepositoryInterface
{
    public function create(ProductData $data): ProductId
    {
        $product = Product::create([...]);
        return $product->id;
    }
}
```

### 4. Agente D expone vía HTTP

```php
final class ProductController
{
    public function store(StoreProductRequest $request): JsonResponse
    {
        $productId = $this->createProduct->execute(
            ProductData::from($request->validated())
        );
        
        return response()->json(['id' => $productId]);
    }
}
```

### 5. Agente E maneja efectos secundarios

```php
final class ProductCreatedListener
{
    public function handle(ProductCreatedEvent $event): void
    {
        // Notificar al equipo de marketing
        // Indexar en search engine
        // Enviar webhook a integración
    }
}
```

---

## Referencias

- **Convenciones del proyecto**: [`conventions.md`](../conventions/conventions.md)
- **Arquitectura de módulos**: [`modules.md`](../conventions/modules.md)
- **Value Objects**: [`value-objects.md`](../conventions/value-objects.md)
- **Agent Contracts**: [`agent-contracts.md`](agent-a-contracts.md)
- **Agent Actions**: [`agent-actions.md`](agent-b-actions.md)
- **Agent Persistence**: [`agent-persistence.md`](agent-c-persistence.md)

---

**Versión**: 1.0  
**Última actualización**: 2025-12-17  
**Autor**: Alejandro Leone

