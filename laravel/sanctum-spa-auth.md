# SPA Authentication con Laravel Sanctum

Sanctum proporciona un método simple para autenticar aplicaciones de página única (SPAs) que necesitan comunicarse con
una API Laravel. Estas SPAs pueden existir en el mismo repositorio que tu aplicación Laravel o ser un repositorio
completamente independiente.

Para esta funcionalidad, Sanctum **no usa tokens de ningún tipo**. En su lugar, utiliza los servicios de autenticación
basados en sesión y cookies integrados en Laravel. Este enfoque proporciona los beneficios de:

- Protección CSRF
- Autenticación de sesión
- Protección contra la filtración de credenciales mediante XSS

## Requisitos previos

Para autenticar, tu SPA y API deben compartir el **mismo dominio de nivel superior**. Sin embargo, pueden estar en
subdominios diferentes.

Además, debes asegurarte de enviar:

- El header `Accept: application/json`
- El header `Referer` o `Origin`

---

## Configuración

### 1. Configurar dominios de primera parte

Primero, debes configurar desde qué dominios tu SPA realizará las solicitudes. Puedes configurar estos dominios usando
la opción `stateful` en tu archivo de configuración de Sanctum. Esta configuración determina qué dominios mantendrán
autenticación "stateful" usando cookies de sesión de Laravel al realizar solicitudes a tu API.

Para ayudarte a configurar tus dominios stateful de primera parte, Sanctum proporciona dos funciones helper:

- `Sanctum::currentApplicationUrlWithPort()` - devuelve la URL actual de la aplicación desde la variable de entorno
  `APP_URL`
- `Sanctum::currentRequestHost()` - inyecta un placeholder en la lista de dominios stateful que, en tiempo de ejecución,
  será reemplazado por el host de la solicitud actual

> **Nota:** Si accedes a tu aplicación mediante una URL que incluye un puerto (ej: `127.0.0.1:8000`), debes asegurarte
> de incluir el número de puerto con el dominio.

### 2. Middleware Sanctum

A continuación, debes instruir a Laravel que las solicitudes entrantes desde tu SPA pueden autenticarse usando cookies
de sesión de Laravel, mientras permites que las solicitudes de terceros o aplicaciones móviles se autentiquen usando
tokens de API.

Esto se logra fácilmente invocando el método middleware `statefulApi` en el archivo `bootstrap/app.php` de tu
aplicación:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

### 3. CORS y Cookies

Si tienes problemas autenticando con tu aplicación desde una SPA que se ejecuta en un subdominio separado, probablemente
hayas configurado incorrectamente tu CORS (Cross-Origin Resource Sharing) o la configuración de cookies de sesión.

#### Publicar configuración CORS

El archivo de configuración `config/cors.php` no se publica por defecto. Si necesitas personalizar las opciones CORS de
Laravel, debes publicar el archivo completo usando el comando Artisan:

```bash
php artisan config:publish cors
```

#### Configurar Access-Control-Allow-Credentials

Debes asegurarte de que la configuración CORS de tu aplicación devuelva el header `Access-Control-Allow-Credentials` con
un valor de `True`. Esto se logra configurando la opción `supports_credentials` en `true` dentro del archivo
`config/cors.php`:

```php
'supports_credentials' => true,
```

#### Configurar Axios (frontend)

Debes habilitar las opciones `withCredentials` y `withXSRFToken` en la instancia global de axios de tu aplicación.
Típicamente, esto debe realizarse en tu archivo `resources/js/bootstrap.js`:

```javascript
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

> **Nota:** Si no estás usando Axios para realizar solicitudes HTTP desde tu frontend, debes realizar la configuración
> equivalente en tu propio cliente HTTP.

#### Configurar dominio de cookie de sesión

Finalmente, debes asegurarte de que la configuración del dominio de la cookie de sesión de tu aplicación soporte
cualquier subdominio de tu dominio raíz. Puedes lograr esto prefijando el dominio con un `.` inicial dentro del archivo
`config/session.php`:

```php
'domain' => '.domain.com',
```

---

## Autenticación

### Protección CSRF

Para autenticar tu SPA, la página de "login" de tu SPA debe primero realizar una solicitud al endpoint
`/sanctum/csrf-cookie` para inicializar la protección CSRF de la aplicación:

```javascript
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

Durante esta solicitud, Laravel configurará una cookie `XSRF-TOKEN` que contiene el token CSRF actual. Este token debe
ser decodificado mediante URL y pasado en un header `X-XSRF-TOKEN` en solicitudes subsiguientes.

Algunas librerías de cliente HTTP como Axios y Angular HttpClient harán esto automáticamente. Si tu librería HTTP de
JavaScript no configura el valor por ti, deberás configurar manualmente el header `X-XSRF-TOKEN` para que coincida con
el valor decodificado de la cookie `XSRF-TOKEN`.

### Iniciar sesión

Una vez que la protección CSRF se ha inicializado, debes realizar una solicitud `POST` a la ruta `/login` de tu
aplicación Laravel. Esta ruta puede implementarse manualmente o usando un paquete de autenticación headless como *
*Laravel Fortify**.

#### Flujo exitoso

Si la solicitud de login es exitosa, serás autenticado y las solicitudes subsiguientes a las rutas de tu aplicación
serán automáticamente autenticadas mediante la cookie de sesión que la aplicación Laravel emitió a tu cliente.

Además, como tu aplicación ya realizó una solicitud al endpoint `/sanctum/csrf-cookie`, las solicitudes subsiguientes
deberían recibir automáticamente protección CSRF siempre que tu cliente HTTP JavaScript envíe el valor de la cookie
`XSRF-TOKEN` en el header `X-XSRF-TOKEN`.

#### Expiración de sesión

Si la sesión de tu usuario expira debido a la falta de actividad, las solicitudes subsiguientes a la aplicación Laravel
pueden recibir una respuesta de error HTTP `401` o `419`. En este caso, debes redirigir al usuario a la página de login
de tu SPA.

> **Importante:** Eres libre de escribir tu propio endpoint `/login`; sin embargo, debes asegurarte de que autentique al
> usuario usando los servicios de autenticación estándar basados en sesión que Laravel proporciona. Típicamente, esto
> significa usar el guard de autenticación `web`.

### Proteger rutas

Para proteger rutas de manera que todas las solicitudes entrantes deban estar autenticadas, debes adjuntar el guard de
autenticación `sanctum` a tus rutas API dentro del archivo `routes/api.php`:

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

Este guard asegurará que las solicitudes entrantes estén autenticadas como solicitudes stateful autenticadas desde tu
SPA o contengan un header de token API válido si la solicitud es de un tercero.

---

## Autorizar canales privados de broadcast

Si tu SPA necesita autenticarse con canales de broadcast privados o de presencia, debes eliminar la entrada `channels`
del método `withRouting` contenido en el archivo `bootstrap/app.php` de tu aplicación. En su lugar, debes invocar el
método `withBroadcasting` para que puedas especificar el middleware correcto para las rutas de broadcasting de tu
aplicación:

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        // ...
    )
    ->withBroadcasting(
        __DIR__.'/../routes/channels.php',
        ['prefix' => 'api', 'middleware' => ['api', 'auth:sanctum']],
    )
```

### Configurar Pusher authorizer

Para que las solicitudes de autorización de Pusher tengan éxito, debes proporcionar un **authorizer personalizado de
Pusher** al inicializar Laravel Echo. Esto permite que tu aplicación configure Pusher para usar la instancia de axios
que está configurada correctamente para solicitudes cross-domain:

```javascript
window.Echo = new Echo({
    broadcaster: "pusher",
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    encrypted: true,
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    authorizer: (channel, options) => {
        return {
            authorize: (socketId, callback) => {
                axios.post('/api/broadcasting/auth', {
                    socket_id: socketId,
                    channel_name: channel.name
                })
                    .then(response => {
                        callback(false, response.data);
                    })
                    .catch(error => {
                        callback(true, error);
                    });
            }
        };
    },
})
```

---

## Resumen del flujo completo

1. **Inicializar CSRF**: `GET /sanctum/csrf-cookie`
2. **Login**: `POST /login` con credenciales
3. **Solicitudes autenticadas**: Todas las solicitudes subsiguientes incluyen automáticamente la cookie de sesión y el
   token CSRF
4. **Proteger rutas**: Usar `middleware('auth:sanctum')` en las rutas API

### Nota de seguridad

- Nunca uses `env()` directamente fuera de archivos de configuración
- Asegúrate de que CORS esté correctamente configurado para tu dominio específico
- Valida y autoriza todas las solicitudes en el backend
- Mantén las cookies de sesión seguras usando HTTPS en producción

