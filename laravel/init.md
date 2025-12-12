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
sail composer require rector/rector --dev
```

### Laravel Boost

```shell
sail composer require --dev laravel-boost/laravel-boost
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