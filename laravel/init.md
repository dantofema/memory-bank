# Iniciar nuevo proyecto Laravel

- Siempre utilizamos Laravel Sail, esto significa que los comandos de Artisan y Composer se ejecutan a través de Sail.

## Asegurarse de tener instalado Laravel Sail

```shell
composer require laravel/sail --dev
sail up -d
sail npm install
```

## Migraciones iniciales`

```shell
sail artisan migrate --seed
```

## Paquetes que debes instalar al iniciar un nuevo proyecto Laravel

### Pint

```shell
sail composer require laravel/pint --dev
./vendor/bin/pint
```

### Larastan

```shell
sail composer require --dev "larastan/larastan:^3.0"
```

Then, create a phpstan.neon or phpstan.neon.dist file in the root of your application. It might look like this:

```neon
includes:

- vendor/larastan/larastan/extension.neon
- vendor/nesbot/carbon/extension.neon

parameters:
    paths:
        - app/
    # Level 10 is the highest level
    level: 5

# ignoreErrors:
# - '#PHPDoc tag @var#'
#

# excludePaths:
# - ./*/*/FileToBeExcluded.php
```

### Rector

```shell
sail composer require rector/rector driftingly/rector-laravel --dev
```

Then, create a rector.php file in the root of your application. It might look like this:

```php
<?php

declare(strict_types=1);

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;
use RectorLaravel\Set\LaravelLevelSetList;
use RectorLaravel\Set\LaravelSetList;

return RectorConfig::configure()
    // 1. Define paths to scan (standard Laravel structure)
    ->withPaths([
        __DIR__.'/app',
        __DIR__.'/config',
        __DIR__.'/database',
        __DIR__.'/routes',
        __DIR__.'/tests',
    ])

    // 2. Skip internal and vendor files
    ->withSkip([
        __DIR__.'/bootstrap/cache',
        __DIR__.'/storage',
        __DIR__.'/vendor',
    ])

    // 3. Apply modern rulesets
    ->withSets([
        // Upgrade everything to Laravel 12 syntax
        LaravelLevelSetList::UP_TO_LARAVEL_120,

        // Laravel-specific refactoring (e.g., Collection methods, Eloquent)
        LaravelSetList::LARAVEL_CODE_QUALITY,
        LaravelSetList::LARAVEL_COLLECTION,

        // General PHP modernization
        LevelSetList::UP_TO_PHP_82, // Min for L12; use 83 or 84 if applicable
        SetList::CODE_QUALITY,
        SetList::DEAD_CODE,
    ])

    // 4. Auto-import class names for cleaner code
    ->withImportNames(removeUnusedImports: true)
    ->withTypeCoverageLevel(52);

```

### Laravel Boost

```shell
sail composer require laravel/boost --dev
sail php artisan boost:install
```

### FilamentPHP

```shell
sail composer require filament/filament:"^4.0"

sail php artisan filament:install --panels

sail php artisan icons:cache
```

## Plugins de FilamentPHP que debes instalar

### Filament Environment Indicator

- https://filamentphp.com/plugins/pxlrbt-environment-indicator

```sh
sail composer require pxlrbt/filament-environment-indicator
```