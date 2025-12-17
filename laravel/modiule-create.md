# Guía: Crear Módulos en Laravel usando Laravel Modules

## Contexto

Esta guía detalla el proceso completo para crear y configurar nuevos módulos en proyectos Laravel usando el paquete
`nwidart/laravel-modules`. Los módulos permiten organizar funcionalidades complejas de forma aislada y mantenible.

---

## 1. Crear el módulo base

Genera la estructura inicial del módulo con el comando Artisan. La opción `-p` crea también archivos básicos de
configuración.

```bash
./vendor/bin/sail php artisan module:make {ModuleName} -p
```

**Ejemplo:**

```bash
./vendor/bin/sail php artisan module:make Catalog -p
```

**Resultado esperado:**

- Se crea la carpeta `Modules/{ModuleName}/` con la estructura estándar
- Incluye directorios: `app/`, `config/`, `database/`, `resources/`, `routes/`, `tests/`

---

## 2. Agregar Service Provider personalizado (opcional)

Si necesitas lógica de inicialización adicional, crea un Service Provider dedicado:

```bash
./vendor/bin/sail php artisan module:make-provider {ModuleName}ServiceProvider {ModuleName}
```

**Ejemplo:**

```bash
./vendor/bin/sail php artisan module:make-provider CatalogServiceProvider Catalog
```

**Nota:** Registrar el provider en `Modules/{ModuleName}/Providers/{ModuleName}ServiceProvider.php` si necesitas
bindings, observers o configuraciones específicas.

---

## 3. Configurar `composer.json` del módulo

Crea o edita el archivo `Modules/{ModuleName}/composer.json` para definir dependencias y autoloading.

**Ubicación:** `Modules/{ModuleName}/composer.json`

```json
{
  "name": "nwidart/{module-name-lowercase}",
  "description": "Descripción breve del módulo",
  "type": "laravel-module",
  "license": "MIT",
  "require": {
    "php": "^8.2"
  },
  "require-dev": {},
  "autoload": {
    "psr-4": {
      "Modules\\{ModuleName}\\": "app/"
    }
  },
  "autoload-dev": {
    "psr-4": {
      "Modules\\{ModuleName}\\Tests\\": "tests/"
    }
  }
}
```

**Ejemplo real (módulo Catalog):**

```json
{
  "name": "nwidart/catalog",
  "description": "Módulo de Catálogo - Gestión de productos, variantes y promociones",
  "type": "laravel-module",
  "license": "MIT",
  "require": {
    "php": "^8.2"
  },
  "require-dev": {},
  "autoload": {
    "psr-4": {
      "Modules\\Catalog\\": "app/"
    }
  },
  "autoload-dev": {
    "psr-4": {
      "Modules\\Catalog\\Tests\\": "tests/"
    }
  }
}
```

**Después de crear/editar `composer.json`:**

```bash
./vendor/bin/sail composer dump-autoload
```

---

## 4. Crear tests con PEST

### Test Feature (pruebas de integración)

**Ubicación:** `Modules/{ModuleName}/tests/Feature/`

```php
<?php

use App\Models\User;

it('can perform feature test', function () {
    $user = User::factory()->create();
    
    $this->actingAs($user);
    
    // Assertions aquí
    expect($user)->toBeInstanceOf(User::class);
});
```

### Test Unit (pruebas unitarias)

**Ubicación:** `Modules/{ModuleName}/tests/Unit/`

```php
<?php

it('validates business logic correctly', function () {
    // Arrange
    $value = 100;
    
    // Act
    $result = $value * 2;
    
    // Assert
    expect($result)->toBe(200);
});
```

### Ejecutar tests del módulo

```bash
# Todos los tests del proyecto
./vendor/bin/sail test

# Solo tests de un módulo específico
./vendor/bin/sail test Modules/{ModuleName}/tests

# Con cobertura (requiere Xdebug)
./vendor/bin/sail test --coverage
```

---

## 5. Verificaciones post-creación

### Checklist

- [ ] El módulo aparece listado: `./vendor/bin/sail php artisan module:list`
- [ ] El autoloading funciona: `./vendor/bin/sail composer dump-autoload`
- [ ] Los tests corren sin errores: `./vendor/bin/sail test`
- [ ] Las rutas del módulo están registradas (verificar en `routes/web.php` o `routes/api.php` del módulo)
- [ ] El módulo está habilitado en `modules_statuses.json`

### Habilitar/deshabilitar módulo

```bash
# Habilitar
./vendor/bin/sail php artisan module:enable {ModuleName}

# Deshabilitar
./vendor/bin/sail php artisan module:disable {ModuleName}
```

---

## 6. Comandos útiles adicionales

```bash
# Crear un modelo en el módulo
./vendor/bin/sail php artisan module:make-model {ModelName} {ModuleName}

# Crear una migración
./vendor/bin/sail php artisan module:make-migration create_{table}_table {ModuleName}

# Crear un controller
./vendor/bin/sail php artisan module:make-controller {ControllerName} {ModuleName}

# Crear un comando
./vendor/bin/sail php artisan module:make-command {CommandName} {ModuleName}

# Publicar assets del módulo
./vendor/bin/sail php artisan module:publish {ModuleName}

# Ejecutar migraciones del módulo
./vendor/bin/sail php artisan module:migrate {ModuleName}

# Ejecutar seeders del módulo
./vendor/bin/sail php artisan module:seed {ModuleName}
```

---

## 7. Convenciones y buenas prácticas

### Estructura recomendada

```
Modules/{ModuleName}/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   ├── Requests/
│   │   └── Resources/
│   ├── Models/
│   ├── Services/
│   ├── Repositories/
│   └── ValueObjects/
├── config/
│   └── config.php
├── database/
│   ├── migrations/
│   ├── seeders/
│   └── factories/
├── resources/
│   └── views/
├── routes/
│   ├── web.php
│   └── api.php
├── tests/
│   ├── Feature/
│   └── Unit/
└── composer.json
```

### Nombres y namespaces

- **Módulos:** PascalCase singular o plural según dominio (`Catalog`, `Orders`, `Payments`)
- **Namespaces:** `Modules\{ModuleName}\{Subdirectory}`
- **Rutas:** Prefijo en kebab-case (`/catalog`, `/orders`, `/payments`)
- **Identificadores:** Inglés siempre
- **Mensajes/strings:** Español

### Seguridad

- Validar requests usando Form Requests (`module:make-request`)
- Aplicar policies/gates para autorización
- No hardcodear credenciales; usar `.env` y `config()`
- Sanitizar inputs en servicios y repositorios

### Calidad de código

- Ejecutar Pint: `./vendor/bin/sail pint Modules/{ModuleName}`
- Ejecutar Larastan: `./vendor/bin/sail php artisan test:types Modules/{ModuleName}`
- Ejecutar Rector: `./vendor/bin/sail vendor/bin/rector process Modules/{ModuleName}`
- Cobertura de tests: mínimo 80% en lógica de negocio

---

## 8. Ejemplo completo: Módulo "Catalog"

```bash
# 1. Crear módulo
./vendor/bin/sail php artisan module:make Catalog -p

# 2. Crear Service Provider
./vendor/bin/sail php artisan module:make-provider CatalogServiceProvider Catalog

# 3. Crear composer.json (manual, ver sección 3)

# 4. Regenerar autoload
./vendor/bin/sail composer dump-autoload

# 5. Crear modelo Product
./vendor/bin/sail php artisan module:make-model Product Catalog -m -f

# 6. Crear controller
./vendor/bin/sail php artisan module:make-controller ProductController Catalog --api

# 7. Crear test Feature
# Archivo: Modules/Catalog/tests/Feature/ProductTest.php
./vendor/bin/sail php artisan module:make-test ProductTest Catalog

# 8. Ejecutar migraciones
./vendor/bin/sail php artisan module:migrate Catalog

# 9. Ejecutar tests
./vendor/bin/sail test Modules/Catalog/tests

# 10. Verificar calidad
./vendor/bin/sail pint Modules/Catalog
```

---

## Notas finales

- Esta guía asume uso de **Laravel Sail** (Docker). Ajustar comandos si se usa otro entorno.
- Revisar documentación oficial: https://nwidart.com/laravel-modules
- Mantener módulos cohesivos: una responsabilidad bien definida por módulo
- Documentar dependencias entre módulos si existen

---

**Última actualización:** 2025-12-17
