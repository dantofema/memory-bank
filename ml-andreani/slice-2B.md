# Slice 2B — Panel Customer (FilamentPHP)

> Formato optimizado para ejecución por AI / agente técnico  
> Stack objetivo: Laravel 12 · PostgreSQL · Redis · Livewire / Volt · FilamentPHP - Sail

---

## Objetivo del Slice

Implementar **panel diferenciado para customers en FilamentPHP** que proporcione una experiencia profesional de self-service.
Este slice **reduce carga de soporte** y ofrece experiencia premium diferenciada por rol.

---

## Definition of Done (global)

- Panel Filament exclusivo para rol customer configurado
- Dashboard con métricas personales del usuario
- Vista de historial de órdenes propias
- Perfil editable dentro del panel
- Políticas de acceso estrictas (customer solo ve sus datos)
- Navegación aislada del panel admin
- Tests de autorización y acceso completos

---

## Tareas técnicas

### 2B.1 Configuración de panel customer en Filament

**Descripción**
Crear un segundo panel de Filament específico para customers.

**Tareas**

- Instalar FilamentPHP si no está (ya debería estar desde Slice 0)
- Crear nuevo panel: `php artisan make:filament-panel customer`
- Configurar rutas diferentes: `/customer` vs `/admin`
- Definir middleware: `auth`, `role:customer`
- Configurar tema y branding diferenciado (opcional: colores distintos)
- Deshabilitar registro desde panel

**DoD**

- Panel customer accesible en `/customer`
- Solo usuarios con role=customer pueden acceder
- Admin no puede acceder a panel customer (y viceversa)
- Configuración en `config/filament.php` o archivo de panel separado

**Fuera de scope**

- Multi-tenancy
- Personalización avanzada de tema
- White-labeling

**Seguridad**

- Middleware de rol estricto
- Verificación de role en cada request
- Redirect a login si no autenticado
- Forbidden si role incorrecto

---

### 2B.2 Dashboard personalizado para customer

**Descripción**
Página principal del panel con métricas relevantes para el usuario.

**Tareas**

- Crear dashboard widget con: total de órdenes, total gastado, última compra
- Widget con gráfico simple (órdenes por mes, últimos 6 meses)
- Widget con estado de última orden
- Widget de acceso rápido a perfil
- Todos los datos scoped al usuario autenticado

**DoD**

- Dashboard muestra métricas correctas
- Datos calculados eficientemente (queries optimizadas)
- Widgets responsivos
- Sin N+1 queries

**Fuera de scope**

- Analytics avanzado
- Comparativas con otros usuarios
- Predicciones o recomendaciones

**Seguridad**

- Queries filtradas por user_id autenticado
- No exponer métricas de otros usuarios
- Cache de métricas con key por usuario

---

### 2B.3 Resource de Órdenes (solo lectura)

**Descripción**
Vista de órdenes del usuario en formato tabla Filament.

**Tareas**

- Crear `OrderResource` en panel customer
- Configurar tabla con columnas: número, fecha, total, estado
- Filtros: por estado, por rango de fechas
- Ordenamiento por defecto: fecha descendente
- View page para detalle de orden
- Deshabilitar create, edit, delete
- Scope global: solo órdenes del usuario

**DoD**

- Tabla lista órdenes correctamente
- Filtros funcionan
- Detalle muestra items de la orden
- Usuario solo ve sus órdenes
- Tests de scope

**Fuera de scope**

- Cancelación de órdenes (requiere lógica de negocio)
- Descarga de factura
- Re-orden

**Seguridad**

- Policy `OrderPolicy::viewAny()` y `view()`
- Scope en query: `->where('user_id', auth()->id())`
- 404 si intenta acceder a orden de otro usuario

---

### 2B.4 Detalle de orden mejorado

**Descripción**
Vista detallada de una orden específica en Filament.

**Tareas**

- Página de detalle (`ViewOrder`) en el Resource
- Secciones: Información general, Items, Totales, Estado
- Mostrar: fecha, número de orden, items (producto, cantidad, precio), total
- Badge visual para estado de orden
- Infolist component de Filament para layout limpio
- Link de vuelta a lista de órdenes

**DoD**

- Detalle completo y legible
- Layout profesional con Filament components
- Solo accesible si usuario es dueño

**Fuera de scope**

- Tracking de envío (Slice 3)
- Historial de estados
- Soporte chat inline

**Seguridad**

- Validar ownership en Policy
- No exponer datos sensibles de pago completos

---

### 2B.5 Perfil editable en panel

**Descripción**
Sección de perfil dentro del panel customer.

**Tareas**

- Crear página personalizada o usar plugin de Profile
- Formulario con: name, email (readonly), phone, address, city, province, postal_code
- Validación Filament nativa
- Guardado con feedback visual
- Opción de cambio de password (modal o sección separada)

**DoD**

- Perfil se actualiza correctamente
- Validación funciona (formato phone, campos requeridos)
- Feedback de éxito claro
- Tests de actualización

**Fuera de scope**

- Avatar upload
- Cambio de email (requiere verificación)
- Preferencias avanzadas

**Seguridad**

- Usuario solo edita su propio perfil
- Validación server-side estricta
- No permitir cambio de role
- Sanitizar inputs

---

### 2B.6 Cambio de password en panel

**Descripción**
Modal o página para cambiar contraseña desde el panel.

**Tareas**

- Crear action o página para cambio de password
- Campos: password actual, nuevo password, confirmar password
- Validación: password actual correcto, mínimo 8 caracteres
- Invalidar otras sesiones post-cambio (opcional)
- Notificación de éxito

**DoD**

- Cambio exitoso actualiza password
- Validación funciona correctamente
- Feedback claro al usuario

**Fuera de scope**

- Recuperación de password olvidada
- Historial de passwords

**Seguridad**

- Verificar password actual antes de cambiar
- Hashear con bcrypt
- Log de cambio de password
- Rate limiting en intentos

---

### 2B.7 Navegación y menú diferenciado

**Descripción**
Configurar menú de navegación específico para customers.

**Tareas**

- Definir items de menú: Dashboard, Mis Órdenes, Perfil
- Ocultar cualquier opción administrativa
- Icons apropiados para cada sección
- Badge con count de órdenes pendientes (opcional)
- User menu con logout

**DoD**

- Navegación clara y simple
- Sin opciones de admin visibles
- Logout funcional

**Fuera de scope**

- Mega menú
- Favoritos o shortcuts
- Personalización de menú por usuario

**Seguridad**

- No exponer rutas admin en navegación
- Verificar permisos incluso si se accede directo a URL

---

### 2B.8 Políticas de acceso (Policies)

**Descripción**
Implementar Policies completas para todos los recursos del panel.

**Tareas**

- `OrderPolicy` con métodos: `viewAny()`, `view()`
- Verificar que `user_id` coincide con auth()->id()
- Aplicar policies en Resources de Filament
- Tests exhaustivos de autorización

**DoD**

- Customer solo ve sus recursos
- Tests pasan: customer no ve órdenes de otros
- 404 (no 403) si intenta acceder a recurso ajeno

**Fuera de scope**

- Permissions granulares
- Compartir órdenes con otros usuarios

**Seguridad**

- Policy en cada action del Resource
- Scope global aplicado siempre
- Nunca confiar solo en frontend/ocultamiento

---

### 2B.9 Scopes globales para aislamiento

**Descripción**
Aplicar Global Scopes para garantizar aislamiento de datos.

**Tareas**

- Crear `UserOwnedScope` que filtra por `user_id = auth()->id()`
- Aplicar a modelo Order cuando se usa en panel customer
- Verificar que queries siempre incluyen el filtro
- Documentar uso del scope

**DoD**

- Scope aplicado automáticamente
- No hay forma de bypassear el filtro desde panel customer
- Tests de scope

**Fuera de scope**

- Scopes complejos multi-tabla
- Scopes desactivables

**Seguridad**

- Scope siempre activo en contexto customer
- Validación doble: scope + policy
- Tests que intenten bypassear

---

### 2B.10 Personalización visual del panel

**Descripción**
Diferenciar visualmente panel customer de panel admin.

**Tareas**

- Configurar colores primarios diferentes (opcional)
- Logo o branding específico para customer
- Título del panel: "Mi Cuenta" o "Portal Cliente"
- Favicon diferenciado (opcional)
- Desactivar opciones de globalización/idiomas si no aplican

**DoD**

- Panel visualmente distinguible del admin
- Branding consistente
- UX clara para usuarios no técnicos

**Fuera de scope**

- Tema completamente custom
- Dark mode
- Múltiples temas seleccionables

**Seguridad**

- N/A (solo visual)

---

### 2B.11 Notificaciones en panel

**Descripción**
Sistema básico de notificaciones para el customer.

**Tareas**

- Configurar notificaciones Filament
- Notificación cuando orden cambia de estado (ej: "Tu orden fue enviada")
- Database notifications o notification banner
- Marcado de leído/no leído
- Badge con count en navegación

**DoD**

- Notificaciones se muestran correctamente
- Usuario puede marcar como leídas
- No spam de notificaciones

**Fuera de scope**

- Notificaciones push
- Email + notificación duplicada
- Configuración granular de notificaciones

**Seguridad**

- Usuario solo ve sus notificaciones
- No exponer notificaciones de otros

---

### 2B.12 Estado de cuenta / resumen

**Descripción**
Widget o página con resumen financiero simple del usuario.

**Tareas**

- Mostrar: total gastado históricamente, promedio de compra
- Última orden con enlace directo
- Próximo pago pendiente si aplica (orden pending)
- Layout con Filament Stats widgets

**DoD**

- Métricas calculadas correctamente
- Queries eficientes (cache si es necesario)
- Datos claros y comprensibles

**Fuera de scope**

- Gráficos complejos
- Exportación de reportes
- Proyecciones

**Seguridad**

- Datos scoped al usuario
- No exponer métricas sensibles del negocio

---

### 2B.13 Gestión de direcciones (preparación)

**Descripción**
Vista de direcciones guardadas del usuario (si ya existe en modelo).

**Tareas**

- Si existe relación User -> Addresses, crear Resource
- Tabla simple: alias, dirección completa, default
- CRUD completo: create, edit, delete
- Validación de campos de dirección
- Marcar dirección como predeterminada

**DoD**

- Usuario puede gestionar sus direcciones
- Validación completa (CP, ciudad, provincia)
- Una dirección marcada como default

**Fuera de scope**

- Geocoding / validación con API
- Múltiples tipos de dirección (facturación/envío)
- Importar desde ML

**Seguridad**

- Usuario solo ve/edita sus direcciones
- Validación de datos de dirección
- Sanitizar inputs

**Nota:** Esta tarea es OPCIONAL si no se implementó direcciones múltiples aún. Si no aplica, skip.

---

### 2B.14 Ayuda y soporte integrado

**Descripción**
Sección de ayuda dentro del panel.

**Tareas**

- Página estática con FAQs básicas
- Link a email de soporte
- Información de contacto
- Cómo realizar una compra (guía rápida)
- Políticas de envío/devolución (enlaces)

**DoD**

- Contenido claro y útil
- Enlaces funcionan
- Accesible desde navegación

**Fuera de scope**

- Chat en vivo
- Sistema de tickets
- Knowledge base completa

**Seguridad**

- N/A (contenido público para autenticados)

---

### 2B.15 Tests de integración del panel

**Descripción**
Cobertura de tests para el panel customer.

**Tareas**

- Test: customer puede acceder a `/customer`
- Test: admin NO puede acceder a `/customer`
- Test: customer solo ve sus órdenes
- Test: navegación muestra opciones correctas
- Test: perfil se actualiza correctamente
- Test: password se cambia correctamente
- Test: dashboard muestra métricas correctas

**DoD**

- Cobertura > 80% del panel customer
- Tests pasan consistentemente
- Sin falsos positivos

**Fuera de scope**

- Tests E2E con browser (ej: Dusk)
- Tests de performance

---

### 2B.16 Documentación de panel customer

**Descripción**
Documentar configuración y uso del panel.

**Tareas**

- Agregar sección en README sobre panel customer
- Cómo acceder (URL, credenciales de prueba)
- Qué puede hacer un customer
- Diferencias con panel admin
- Screenshots opcionales

**DoD**

- Documentación clara y concisa
- Otro dev puede entender el sistema

**Fuera de scope**

- Manual de usuario final
- Videos tutoriales

---

## Restricciones

- No implementar features de admin en panel customer
- No permitir acceso cross-panel (admin en customer o viceversa)
- No implementar multi-tenancy complejo
- No agregar complejidad innecesaria en permisos

---

## Regla de corte

Si una tarea:

- Agrega features que el customer no necesita
- Introduce riesgo de acceso cross-user
- Requiere dependencias externas pesadas

→ **queda fuera del Slice 2B**

---

## Nota de seguridad (obligatoria)

- Middleware de rol en TODAS las rutas del panel
- Policies aplicadas en todos los Resources
- Global Scopes activos en queries de datos sensibles
- Validación de ownership en cada acción
- No confiar en ocultamiento de UI (validar server-side)
- Logs de accesos sospechosos
- Rate limiting en cambio de password
- Session timeout apropiado
- CSRF protection activo
- No exponer datos de otros usuarios nunca

---

## Orden de ejecución sugerido

1. Configuración panel customer (2B.1)
2. Políticas de acceso (2B.8)
3. Scopes globales (2B.9)
4. Resource de Órdenes (2B.3)
5. Detalle de orden (2B.4)
6. Dashboard personalizado (2B.2)
7. Navegación y menú (2B.7)
8. Perfil editable (2B.5)
9. Cambio de password (2B.6)
10. Personalización visual (2B.10)
11. Notificaciones (2B.11)
12. Estado de cuenta (2B.12)
13. Ayuda y soporte (2B.14)
14. Gestión direcciones (2B.13) - OPCIONAL
15. Tests de integración (2B.15)
16. Documentación (2B.16)

---

## Comandos útiles (vía Sail)

```bash
# Crear panel customer
./vendor/bin/sail artisan make:filament-panel customer

# Crear Resource en panel específico
./vendor/bin/sail artisan make:filament-resource Order --panel=customer

# Crear página custom en panel
./vendor/bin/sail artisan make:filament-page Dashboard --panel=customer

# Crear widget
./vendor/bin/sail artisan make:filament-widget OrderStats --panel=customer

# Publicar configuración de Filament
./vendor/bin/sail artisan vendor:publish --tag=filament-config

# Tests específicos de panel
./vendor/bin/sail test --filter=CustomerPanel

# Limpiar cache de Filament
./vendor/bin/sail artisan filament:cache-components
```

---

## Configuración típica del panel (ejemplo)

```php
// app/Providers/Filament/CustomerPanelProvider.php

use Filament\Panel;
use Filament\PanelProvider;
use App\Filament\Customer\Pages\Dashboard;

class CustomerPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('customer')
            ->path('customer')
            ->login()
            ->colors([
                'primary' => 'blue',
            ])
            ->discoverResources(in: app_path('Filament/Customer/Resources'), for: 'App\\Filament\\Customer\\Resources')
            ->discoverPages(in: app_path('Filament/Customer/Pages'), for: 'App\\Filament\\Customer\\Pages')
            ->pages([
                Dashboard::class,
            ])
            ->discoverWidgets(in: app_path('Filament/Customer/Widgets'), for: 'App\\Filament\\Customer\\Widgets')
            ->middleware([
                'web',
                'auth',
                'role:customer', // Middleware custom
            ])
            ->authMiddleware([
                'auth',
            ])
            ->brandName('Mi Cuenta')
            ->registration(false); // Deshabilitar registro desde panel
    }
}
```

---

## Dependencias con otros slices

- **Requiere:** Slice 0 (infraestructura y FilamentPHP instalado)
- **Requiere:** Slice 1 (modelo Order)
- **Requiere:** Slice 2 (sistema de roles y usuarios)
- **Habilita:** Mejora experiencia de usuario
- **Complementa:** Slice 4 (panel admin separado)

---

## Valor entregado

Al finalizar este slice:

- **Experiencia premium para customers** → diferenciación de servicio
- **Self-service completo** → reduce carga de soporte
- **Dashboard profesional** → confianza y transparencia
- **Aislamiento de datos garantizado** → seguridad por diseño
- **Base para features futuras** → notificaciones, reportes personalizados

---

## Próximo slice recomendado

**Slice 3 — Logística Argentina: Andreani** para completar el checkout con cálculo de envíos, o **Slice 4 — Backoffice operativo (Filament Admin)** para habilitar gestión completa del negocio.
