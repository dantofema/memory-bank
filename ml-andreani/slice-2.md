# Slice 2 — Usuarios reales (Auth completa + Roles)

> Formato optimizado para ejecución por AI / agente técnico  
> Stack objetivo: Laravel 12 · PostgreSQL · Redis · Livewire / Volt · FilamentPHP - Sail

---

## Objetivo del Slice

Implementar **gestión completa de usuarios** con registro, autenticación y sistema de roles para diferenciar customers
de admins.
Este slice **habilita fidelización** y prepara la base para dashboards personalizados y gestión de órdenes por usuario.

---

## Definition of Done (global)

- Usuario puede registrarse con validación completa
- Usuario puede iniciar sesión y cerrar sesión
- Sistema de roles funcional (admin/customer)
- Órdenes asociadas a usuarios autenticados
- Usuario puede ver su historial de órdenes
- Perfil de usuario editable
- Middleware de autorización por rol operativo

---

## Tareas técnicas

### 2.1 Migración de tabla users mejorada

**Descripción**
Extender tabla users existente con campos necesarios para el negocio.

**Tareas**

- Crear migración para agregar campos: `phone`, `address`, `city`, `province`, `postal_code`, `role`
- Definir `role` como enum (`admin`, `customer`) con default `customer`
- Agregar índices en `email` y `role`
- Soft deletes habilitado
- Timestamps correctos

**DoD**

- Migración idempotente
- Rollback funcional
- Índices creados correctamente

**Fuera de scope**

- Direcciones múltiples
- Datos fiscales (CUIT/DNI)
- Verificación de identidad

**Seguridad**

- Email único y validado
- Phone sanitizado
- Roles no modificables directamente por usuario

---

### 2.2 Modelo User extendido

**Descripción**
Extender modelo User con relaciones y métodos necesarios.

**Tareas**

- Agregar relación `orders()` hasMany
- Crear métodos helper: `isAdmin()`, `isCustomer()`
- Definir fillable y hidden correctamente
- Factory actualizado con datos faker argentinos

**DoD**

- Relaciones funcionando
- Helper methods testeados
- Factory genera usuarios válidos

**Fuera de scope**

- Avatar de usuario
- Preferencias complejas
- Integración con redes sociales

**Seguridad**

- Password nunca en fillable directo
- Email verification preparado
- Role protegido de mass assignment

---

### 2.3 Registro de usuario

**Descripción**
Pantalla de registro con validación completa.

**Tareas**

- Habilitar registration en en panel de Filament
- Crear una Page para el registro `sail php artisan make:filament-page Register`
- Actualizar el Panel de Filament para permitir custom page `->registration(Register::class)`
- Validación: email único, password (min 8 chars), phone format argentino
- Campos: name, email, password, password_confirmation, phone
- Role asignado automáticamente como `customer`
- Event `UserRegistered` disparado

**DoD**

- Formulario valida correctamente
- Usuario creado con role customer
- Redirect post-registro a dashboard o home
- Tests de validación completos

**Fuera de scope**

- Verificación por email obligatoria
- Registro social (Google, etc)
- Captcha

**Seguridad**

- Password hasheado con bcrypt
- Validación server-side estricta

---

### 2.4 Login de usuario

**Descripción**
Pantalla de login con autenticación Laravel nativa.

**Tareas**

- Habilitar login en panel de Filament
- Remember me funcional
- Redirect diferenciado por rol (admin → AdminPanel, customer → CustomerPanel)

**DoD**

- Login exitoso redirige correctamente
- Session persistida
- Remember me funciona
- Tests de autenticación

**Fuera de scope**

- 2FA
- Login social
- Magic links

**Seguridad**

- Throttling: 5 intentos por minuto
- Logs de intentos fallidos
- No revelar si usuario existe
- Session secure (httponly, samesite)

---

### 2.5 Logout

**Descripción**
Funcionalidad de cierre de sesión seguro.

**Tareas**

- Redirect a home o login

**DoD**

- Sesión destruida completamente
- No acceso a rutas protegidas post-logout
- Tests funcionales

**Seguridad**

- POST only (no GET)
- CSRF token validado
- Invalidar todas las sesiones del usuario

---

### 2.6 Sistema de roles con enum

**Descripción**
Implementar roles mediante enum nativo de PHP/Laravel.

**Tareas**

- Crear enum `UserRole` con valores: `ADMIN`, `CUSTOMER`
- Cast en modelo User
- Middleware `role:admin` y `role:customer`
- Gates para verificación: `Gate::define('isAdmin')`
- Seeders con usuario admin de prueba

**DoD**

- Roles asignados correctamente
- Middleware funciona
- Gates operativos
- Usuario admin creado por seeder

**Fuera de scope**

- Permissions granulares (ej: Spatie)
- Roles custom definibles
- Multi-rol por usuario

**Seguridad**

- Roles no modificables vía request
- Verificación en cada endpoint sensible
- Admin solo creado por seeder/comando

---

### 2.7 Relación User ↔ Order

**Descripción**
Asociar órdenes existentes y nuevas a usuarios.

**Tareas**

- Migración: modificar `orders.user_id` de nullable a required (con strategy para data existente)
- Actualizar checkout para asignar user_id automáticamente
- Listener que asigna user autenticado a orden
- Migrar órdenes guest existentes (si aplica)

**DoD**

- Todas las órdenes tienen user_id
- Checkout automáticamente asocia usuario
- Relación bidireccional funcional

**Fuera de scope**

- Órdenes guest (se requiere login)
- Transferir órdenes entre usuarios

**Seguridad**

- Validar ownership en todas las consultas
- Policy `OrderPolicy` con método `view()`

---

### 2.8 Historial de órdenes del usuario

**Descripción**
Pantalla donde usuario ve sus compras pasadas.

**Tareas**

- Crear ruta `/my-orders`
- Componente Livewire con listado paginado
- Mostrar: número de orden, fecha, total, estado
- Link a detalle de cada orden
- Ordenar por fecha descendente
- Filtrar solo órdenes del usuario autenticado

**DoD**

- Listado correcto y paginado
- Solo órdenes propias visibles
- Performance aceptable (eager loading)

**Fuera de scope**

- Re-orden (comprar lo mismo)
- Cancelación de orden
- Descarga de factura

**Seguridad**

- Policy que valide ownership
- No exponer órdenes de otros usuarios
- Queries scoped por user_id

---

### 2.9 Detalle de orden

**Descripción**
Página de detalle de una orden específica.

**Tareas**

- Crear ruta `/orders/{order}`
- Vista con: items, cantidades, precios, total, estado, fecha
- Validar que usuario solo vea sus órdenes
- Breadcrumb navegable

**DoD**

- Detalle completo y claro
- Acceso validado por Policy
- Tests de autorización

**Fuera de scope**

- Tracking de envío (Slice 3)
- Reembolsos
- Soporte inline

**Seguridad**

- Policy `OrderPolicy::view()`
- 404 si no es dueño (no 403 para no revelar existencia)

---

### 2.10 Perfil de usuario editable

**Descripción**
Pantalla donde usuario actualiza sus datos.

**Tareas**

- Crear ruta `/profile`
- Componente Livewire con formulario
- Campos editables: name, phone, address, city, province, postal_code
- Validación completa
- Feedback de guardado exitoso

**DoD**

- Datos se actualizan correctamente
- Validación funciona
- Tests de actualización

**Fuera de scope**

- Cambio de email (requiere verificación)
- Cambio de password (separado)
- Avatar

**Seguridad**

- Validar que usuario solo edita su perfil
- Sanitizar inputs
- No permitir cambio de role

---

### 2.11 Cambio de password

**Descripción**
Funcionalidad separada para cambiar contraseña.

**Tareas**

- Sección en perfil o ruta `/profile/password`
- Validar password actual
- Validar nuevo password (min 8 chars)
- Confirmación de password
- Invalidar sesiones anteriores post-cambio

**DoD**

- Cambio exitoso invalida otras sesiones
- Password hasheado correctamente
- Tests completos

**Fuera de scope**

- Recuperación de password (Slice futuro)
- Historial de passwords

**Seguridad**

- Validar password actual antes de cambiar
- Hashear nuevo password con bcrypt
- Log de cambio de password

---

### 2.12 Middleware de protección de rutas

**Descripción**
Asegurar que rutas sensibles requieren autenticación y rol correcto.

**Tareas**

- Aplicar middleware `auth` a: `/my-orders`, `/profile`, `/checkout`
- Aplicar middleware `role:admin` a rutas admin (preparación para Slice 4)
- Crear middleware custom `customer` si es necesario
- Redirect correcto según contexto (login vs forbidden)

**DoD**

- Rutas protegidas correctamente
- Redirects lógicos
- Tests de middleware

**Fuera de scope**

- Permissions granulares
- Logs de acceso

**Seguridad**

- Verificación en cada request
- No confiar en datos de sesión sin validar
- Rate limiting en rutas sensibles

---

### 2.13 Seeders de usuarios de prueba

**Descripción**
Crear datos de prueba para desarrollo.

**Tareas**

- Seeder con 1 admin (`admin@test.com`)
- Seeder con 5 customers de prueba
- Passwords conocidas para testing (`password`)
- Ejecutable con `sail artisan db:seed`

**DoD**

- Usuarios creados correctamente
- Datos consistentes
- No se ejecuta en producción

**Seguridad**

- Solo para entorno local/staging
- Guard contra ejecución en prod

---

### 2.14 Email de bienvenida

**Descripción**
Enviar email post-registro exitoso.

**Tareas**

- Crear Mailable `WelcomeEmail`
- Template simple en español
- Enviar via queue
- Listener de evento `UserRegistered`

**DoD**

- Email se envía correctamente
- Template claro y profesional
- Queue funcional

**Fuera de scope**

- Email de verificación obligatoria
- Drip campaign

**Seguridad**

- Rate limiting en envíos
- No exponer info sensible

---

### 2.15 Tests de autorización

**Descripción**
Cobertura completa de casos de autorización.

**Tareas**

- Test: customer no accede a rutas admin
- Test: usuario no ve órdenes de otros
- Test: usuario no autenticado redirige a login
- Test: roles asignados correctamente
- Test: policies funcionan

**DoD**

- Cobertura > 85% en autorización
- Tests pasan consistentemente
- Sin falsos positivos

---

### 2.16 Métricas de usuarios

**Descripción**
Instrumentar registros y autenticación.

**Tareas**

- Log de registro exitoso
- Log de login fallido (sin exponer password)
- Counter de usuarios activos
- Log de cambios de password

**DoD**

- Logs estructurados
- Info suficiente para debug
- Sin datos sensibles

**Fuera de scope**

- Dashboard de analytics
- Alertas automáticas

---

## Restricciones

- No implementar verificación de email obligatoria (opcional para futuro)
- No implementar recuperación de password (Slice futuro)
- No implementar panel Filament para customers (eso es Slice 2B)
- No implementar permissions granulares (usar solo roles)

---

## Regla de corte

Si una tarea:

- Agrega complejidad innecesaria al sistema de auth
- Introduce dependencias externas complejas
- No es crítica para diferenciar admin de customer

→ **queda fuera del Slice 2**

---

## Nota de seguridad (obligatoria)

- Password hasheado con bcrypt (nunca plain text)
- CSRF protection en todos los formularios
- Rate limiting en login y registro
- Validación server-side estricta en todos los campos
- Roles validados en cada request sensible
- Session secure (httponly, samesite=lax)
- Logs sin passwords ni tokens
- Policies para autorización de recursos
- No revelar si usuario existe en login fallido
- Invalidar sesiones antiguas al cambiar password

---

## Orden de ejecución sugerido

1. Migración users extendida (2.1)
2. Modelo User extendido (2.2)
3. Sistema de roles (2.6)
4. Registro de usuario (2.3)
5. Login de usuario (2.4)
6. Logout (2.5)
7. Relación User ↔ Order (2.7)
8. Middleware de protección (2.12)
9. Historial de órdenes (2.8)
10. Detalle de orden (2.9)
11. Perfil editable (2.10)
12. Cambio de password (2.11)
13. Email bienvenida (2.14)
14. Seeders (2.13)
15. Tests autorización (2.15)
16. Métricas (2.16)

---

## Comandos útiles (vía Sail)

```bash
# Crear migración
./vendor/bin/sail artisan make:migration add_profile_fields_to_users_table

# Crear enum
./vendor/bin/sail artisan make:enum UserRole

# Crear componentes Livewire
./vendor/bin/sail artisan make:livewire Auth/Register
./vendor/bin/sail artisan make:livewire Auth/Login
./vendor/bin/sail artisan make:livewire Profile/Edit

# Crear Policy
./vendor/bin/sail artisan make:policy OrderPolicy --model=Order

# Crear Mailable
./vendor/bin/sail artisan make:mail WelcomeEmail

# Crear Seeder
./vendor/bin/sail artisan make:seeder UserSeeder

# Ejecutar seeders
./vendor/bin/sail artisan db:seed --class=UserSeeder

# Tests
./vendor/bin/sail test --filter=Auth
./vendor/bin/sail test --filter=Authorization

# Crear usuario admin manualmente
./vendor/bin/sail artisan tinker
>>> User::create(['name' => 'Admin', 'email' => 'admin@test.com', 'password' => bcrypt('password'), 'role' => UserRole::ADMIN])
```

---

## Dependencias con otros slices

- **Requiere:** Slice 0 (auth básica ya existe, se extiende)
- **Requiere:** Slice 1 (modelo Order debe existir)
- **Habilita:** Slice 2B (panel customer en Filament)
- **Habilita:** Slice 4 (backoffice admin requiere roles)

---

## Valor entregado

Al finalizar este slice:

- **Usuarios pueden registrarse y comprar** → fidelización
- **Historial de compras disponible** → confianza y transparencia
- **Sistema de roles funcional** → preparación para backoffice
- **Base para postventa** → comunicación con clientes identificados
- **Seguridad mejorada** → autorización granular por recurso

---

## Próximo slice recomendado

**Slice 2B — Panel Customer (FilamentPHP)** para dar a los customers un dashboard profesional con métricas y gestión de
sus datos, o **Slice 3 — Logística Andreani** para agregar cálculo de envíos si se prioriza completar el checkout
funcional.
