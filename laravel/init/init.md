# Guía de Inicio: Proyecto Laravel - Optimizada para IA

## Contexto del Stack

**Entorno**: Laravel Sail (Docker)  
**Versiones principales**:

- PHP 8.5+
- Laravel 12
- Filament 4
- Livewire 3
- Pest 4
- TailwindCSS 4
- PostgreSQL

**Regla fundamental**: TODOS los comandos Artisan, Composer, NPM y PHP se ejecutan a través de `./vendor/bin/sail` (o
alias `sail`).

---

## Secuencia de Inicialización

### 1. Instalación base de Sail

```bash
composer require laravel/sail --dev
php artisan sail:install
./vendor/bin/sail up -d
./vendor/bin/sail npm install
```

### 2. Habilitar Xdebug (opcional pero recomendado)

**Configuración en `.env`**:

```bash
SAIL_XDEBUG_MODE=develop,debug
```

**Reiniciar contenedores para aplicar cambios**:

```bash
./vendor/bin/sail down
./vendor/bin/sail up -d
```

**Nota IA**: Xdebug permite debugging paso a paso con breakpoints en el IDE. El modo `develop` habilita características
de desarrollo, y `debug` activa el debugging remoto. Útil para debugging complejo pero puede ralentizar la ejecución;
desactivar en producción.

### 3. Migraciones y datos iniciales

```bash
./vendor/bin/sail artisan migrate --seed
```

---

## Paquetes Obligatorios de Calidad de Código

### Pint (Formateador de código)

**Instalación**:

```bash
./vendor/bin/sail composer require laravel/pint --dev
```

**Uso inmediato**:

```bash
./vendor/bin/sail bin pint
```

**Nota IA**: Ejecutar `sail bin pint` después de CADA cambio de código antes de finalizar.

---

### Larastan (Análisis estático - PHPStan para Laravel)

**Instalación**:

```bash
./vendor/bin/sail composer require --dev "larastan/larastan:^3.0"
```

**Archivo de configuración**: Crear `phpstan.neon` en la raíz:

```neon
includes:
    - vendor/larastan/larastan/extension.neon
    - vendor/nesbot/carbon/extension.neon

parameters:
    paths:
        - app/
    level: 5  # Incrementar gradualmente hasta 10 (máximo rigor)
```

**Ejecución**:

```bash
./vendor/bin/sail bin phpstan analyse
```

**Nota IA**: Este proyecto usa Larastan en modo estricto. Validar código generado contra nivel configurado.

---

### Rector (Refactoring automatizado)

**Instalación**:

```bash
./vendor/bin/sail composer require rector/rector driftingly/rector-laravel --dev
```

**Archivo de configuración**: Crear `rector.php` en la raíz:

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\DeadCode\Rector\ClassMethod\RemoveUnusedPublicMethodParameterRector;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;
use RectorLaravel\Rector\ClassMethod\MakeModelAttributesAndScopesProtectedRector;
use RectorLaravel\Set\LaravelLevelSetList;
use RectorLaravel\Set\LaravelSetList;

return RectorConfig::configure()
    ->withPaths([
        __DIR__.'/app',
        __DIR__.'/config',
        __DIR__.'/database',
        __DIR__.'/routes',
        __DIR__.'/tests',
    ])
    ->withSkip([
        __DIR__.'/bootstrap/cache',
        __DIR__.'/storage',
        __DIR__.'/vendor',
    ])
    ->withSets([
        LaravelLevelSetList::UP_TO_LARAVEL_120,
        LaravelSetList::LARAVEL_CODE_QUALITY,
        LaravelSetList::LARAVEL_COLLECTION,
        LevelSetList::UP_TO_PHP_82,
        SetList::CODE_QUALITY,
        SetList::DEAD_CODE,
    ])
    ->withImportNames(removeUnusedImports: true)
    ->withTypeCoverageLevel(52)
    ->withSkip([
        MakeModelAttributesAndScopesProtectedRector::class,
        RemoveUnusedPublicMethodParameterRector::class,
    ]);
```

**Ejecución**:

```bash
./vendor/bin/sail bin rector process --dry-run  # Preview
./vendor/bin/sail bin rector process            # Aplicar cambios
```

---

## Framework Frontend/Admin

### FilamentPHP 4

**Instalación**:

```bash
./vendor/bin/sail composer require filament/filament:"^4.0"
./vendor/bin/sail artisan filament:install --panels
./vendor/bin/sail artisan icons:cache
```

**Configuración del Panel**:

Archivo: `app/Providers/Filament/AdminPanelProvider.php`

```php
use Filament\Panel;
use pxlrbt\FilamentEnvironmentIndicator\EnvironmentIndicatorPlugin;

public function panel(Panel $panel): Panel
{
    return $panel
        // ...existing code...
        ->spa()      // Activa modo SPA (mejora UX sin recargas completas)
        ->profile()  // Habilita gestión de perfil de usuario
        ->plugins([
            EnvironmentIndicatorPlugin::make(),  // Muestra entorno activo (local/staging/production)
        ]);
}
```

**Nota IA**: El modo SPA mejora la experiencia de usuario al eliminar recargas de página. El perfil permite a los
usuarios gestionar su cuenta. El Environment Indicator previene errores al identificar visualmente el entorno activo.

**Plugin Environment Indicator**:

```bash
./vendor/bin/sail composer require pxlrbt/filament-environment-indicator
```

**Documentación**: https://filamentphp.com/plugins/pxlrbt-environment-indicator

---

## Testing: Architecture Tests (Pest)

**Crear archivo**: `tests/Feature/ArchTest.php`

```php
<?php

declare(strict_types=1);

use App\Models\User;
use Illuminate\Database\Eloquent\Model;
use Livewire\Wireable;

arch()
    ->expect('App\Enums')
    ->toBeEnums();

arch()
    ->expect('App\Models')
    ->toBeClasses()
    ->toExtend(Model::class)
    ->ignoring(User::class);

arch()
    ->expect('App\Http')
    ->toOnlyBeUsedIn('App\Http');

arch()->preset()->security()->ignoring('md5');

arch()
    ->expect('App\ValueObjects')
    ->toImplement(Wireable::class);

arch()
    ->expect('App\Models')
    ->toHaveLineCountLessThan(100);

arch()
    ->expect('App')
    ->toHaveLineCountLessThan(300)
    ->ignoring(['App\Providers', 'App\Models']);

arch()->preset()->php();

arch()->preset()->strict()->ignoring('App\Filament');

arch()->preset()->laravel();
```

**Ejecución**:

```bash
./vendor/bin/sail artisan test --filter=ArchTest
```

---

## Testing: Browser Tests (Pest Browser Plugin)

**Instalación**:

```bash
./vendor/bin/sail composer require pestphp/pest-plugin-browser --dev
./vendor/bin/sail npm install playwright@latest
./vendor/bin/sail npx playwright install
```

**Directorio de tests**: `tests/Browser/`

**Ejemplo de test básico**:

```php
<?php

declare(strict_types=1);

it('loads the homepage successfully', function () {
    $page = visit('/');
    
    $page->assertSee('Bienvenido')
        ->assertNoJavascriptErrors()
        ->assertNoConsoleLogs();
});

it('can navigate and interact with forms', function () {
    $page = visit('/login');
    
    $page->fill('email', 'usuario@example.com')
        ->fill('password', 'password123')
        ->click('Iniciar sesión')
        ->assertSee('Dashboard');
});
```

**Ejecución**:

```bash
./vendor/bin/sail artisan test tests/Browser/
```

**Capacidades de Browser Tests**:

- Interacción real con navegadores (Chrome, Firefox, Safari)
- Testing E2E de flujos completos de usuario
- Validación de JavaScript, AJAX y comportamiento asíncrono
- Captura de screenshots y grabación de video
- Testing en múltiples viewports (móvil, tablet, desktop)
- Modo headless o visible para debugging

**Nota IA**: Los Browser Tests son ideales para validar flujos críticos de usuario que involucran múltiples pasos,
interacciones complejas con JavaScript, o comportamiento visual. Complementan los Feature Tests al probar la aplicación
como lo haría un usuario real en un navegador.

---

## Herramientas de Desarrollo Laravel

### Laravel Modules (Arquitectura modular)

**Instalación**:

```bash
./vendor/bin/sail composer require nwidart/laravel-modules
./vendor/bin/sail php artisan vendor:publish --provider="Nwidart\Modules\LaravelModulesServiceProvider"
```

**Configuración en `composer.json`**:

Agregar la siguiente configuración en la sección `extra`:

```json
"extra": {
"laravel": {
"dont-discover": []
},
"merge-plugin": {
"include": [
"Modules/*/composer.json"
]
}
},
```

**Configuración de tests para módulos**:

En `Pest.php` agregar el siguiente in() '../Modules/*/tests/Feature' para que lea los tests:

```php
pest()->extend(Tests\TestCase::class)
    ->use(Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature', '../Modules/*/tests/Feature','../Modules/*/tests/Browser');
```

**Regenerar autoload**:

```bash
./vendor/bin/sail composer dump-autoload
```

**Nota IA**: Laravel Modules permite organizar la aplicación en módulos independientes y reutilizables. Cada módulo
puede contener sus propios controladores, modelos, vistas, rutas y migraciones. Ideal para aplicaciones grandes con
múltiples dominios o funcionalidades bien delimitadas.

---

### Laravel Data (DTOs tipados)

**Instalación**:

```bash
./vendor/bin/sail composer require spatie/laravel-data
```

### Laravel Boost (MCP Server - Herramientas IA)

```bash
./vendor/bin/sail composer require laravel/boost --dev
./vendor/bin/sail artisan boost:install
```

**Nota IA**: Boost provee herramientas MCP especializadas para este proyecto (tinker, database-query, search-docs,
etc.).

---

## Checklist Post-Instalación para IA

Después de instalar estos paquetes, verificar:

- [ ] `./vendor/bin/sail up -d` → Contenedores corriendo
- [ ] `./vendor/bin/sail artisan migrate` → Base de datos inicializada
- [ ] `./vendor/bin/sail bin pint` → Sin errores de formato
- [ ] `./vendor/bin/sail bin phpstan analyse` → Sin errores de análisis estático
- [ ] `./vendor/bin/sail artisan test` → Todos los tests pasan
- [ ] `./vendor/bin/sail npm run build` → Assets compilados

---

## Comandos de Calidad para IA (ejecutar antes de finalizar cambios)

```bash
# 1. Formatear código
./vendor/bin/sail bin pint

# 2. Análisis estático
./vendor/bin/sail bin phpstan analyse

# 3. Refactoring (preview)
./vendor/bin/sail bin rector process --dry-run

# 4. Tests relevantes
./vendor/bin/sail artisan test --filter=NombreDelTest

# 5. Browser tests (si aplica)
./vendor/bin/sail artisan test tests/Browser/

# 6. Compilar assets (si hay cambios frontend)
./vendor/bin/sail npm run build
```

---

## Notas de Seguridad

- **Secretos**: Nunca incluir credenciales en ejemplos de código.
- **Variables de entorno**: Usar `config('app.key')`, NUNCA `env('APP_KEY')` fuera de archivos de configuración.
- **Validación**: Siempre usar Form Requests para validación de datos.
- **Autorización**: Implementar Gates y Policies para control de acceso.

---

## Convenciones de Proyecto

- **Identificadores**: Inglés (variables, funciones, clases).
- **Strings/Mensajes**: Español.
- **Tipado**: Fuertemente tipado (type hints, return types obligatorios).

- **Frontend**: Livewire + Filament para interactividad, TailwindCSS 4 para estilos.
- **Tests**: Pest 4, preferir Feature tests sobre Unit tests.
- **Factories**: Usar factories para crear modelos en tests, no datos manuales.
