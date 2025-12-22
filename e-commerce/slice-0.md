# Slice 0 — Infraestructura mínima (habilitante)
> Formato optimizado para ejecución por AI / agente técnico  
> Stack objetivo: Laravel 12 · PostgreSQL · Redis · Livewire / Volt · FilamentPHP - Sail

---

## Objetivo del Slice
Tener un proyecto **deployable, seguro y observable** sin dominio de negocio.
Este slice **no entrega valor funcional**, solo habilita los siguientes.

---

## Definition of Done (global)
- El proyecto levanta local y remoto
- Existe al menos un deploy funcional
- Auth básica operativa
- Infraestructura mínima validada
- Sin deuda estructural consciente

---

## Tareas técnicas

### 0.1 Inicialización del proyecto
**Descripción**
Crear un proyecto Laravel productivo con convenciones claras.

**Tareas**
- Crear proyecto Laravel 12
- Inicializar repositorio Git
- Definir estructura base del proyecto
- Configurar PHP >= 8.5

**DoD**
- `sail up -d` levanta sin errores
- Repositorio versionado correctamente

**Fuera de scope**
- Dominio de negocio
- UI final

---

### 0.2 Configuración de entorno
**Descripción**
Configurar variables de entorno y dependencias externas.

**Tareas**
- Crear `.env.example` completo
- Configurar conexión a PostgreSQL
- Configurar Redis (cache + queue)
- Configurar mailer dummy
- Definir `APP_ENV`, `APP_DEBUG`

**DoD**
- Proyecto levanta con `.env.example`
- No hay secretos hardcodeados

**Seguridad**
- Nunca commitear `.env`
- `APP_DEBUG=false` en prod

---

### 0.3 Base de datos y migraciones
**Descripción**
Preparar base de datos con convenciones sólidas.

**Tareas**
- Configurar conexión PostgreSQL
- Crear migración `users`
- Definir timestamps e índices mínimos
- Verificar rollback correcto

**DoD**
- `php artisan migrate:fresh` sin errores
- Migraciones idempotentes

**Fuera de scope**
- Modelado de negocio futuro

---

### 0.4 Autenticación básica
**Descripción**
Habilitar autenticación mínima sin UX final.

**Tareas**
- Implementar login / logout
- Hashing correcto de passwords
- Crear roles básicos (`admin`, `customer`)
- Middleware de autenticación

**DoD**
- Usuario puede loguearse
- Roles persistidos correctamente

**Seguridad**
- Password hashing nativo
- Protección CSRF activa

---

### 0.5 Autorización mínima
**Descripción**
Definir control de acceso base para el sistema.

**Tareas**
- Configurar Policies o Gates base
- Restringir acceso a rutas protegidas
- Separar acceso admin / usuario

**DoD**
- Accesos indebidos bloqueados
- Policies centralizadas

---

### 0.6 Observabilidad y manejo de errores
**Descripción**
Asegurar visibilidad mínima de fallos.

**Tareas**
- Configurar logging (daily)
- Manejo global de excepciones
- Error pages no verbosas en prod

**DoD**
- Logs generados correctamente
- Errores no exponen info sensible

**Seguridad**
- No loguear payloads sensibles
- Sanitizar mensajes de error

---

### 0.7 Feature flags
**Descripción**
Preparar toggles para slices futuros.

**Tareas**
- Implementar flags vía config o DB
- Documentar flags disponibles

**DoD**
- Flags activables/desactivables
- Sin impacto funcional actual

---

### 0.8 Primer deploy
**Descripción**
Poner el sistema online (staging o prod barato).

**Tareas**
- Configurar servidor
- Variables de entorno remotas
- HTTPS habilitado
- Storage configurado

**DoD**
- URL accesible
- App responde sin errores

**Seguridad**
- HTTPS obligatorio
- Secrets solo en entorno remoto

---

### 0.9 README técnico
**Descripción**
Documentación mínima para continuidad.

**Tareas**
- Cómo levantar local
- Cómo correr migraciones
- Cómo deployar
- Qué NO está implementado

**DoD**
- README ≤ 1 página
- Información suficiente para otro dev / AI

---

## Restricciones
- No implementar dominio de negocio
- No integrar APIs externas
- No UI final
- No optimizaciones prematuras

---

## Regla de corte
Si una tarea:
- No es deployable
- Introduce dominio
- Aumenta complejidad futura

→ **queda fuera del Slice 0**

---

## Nota de seguridad (obligatoria)
- Secrets fuera del repo
- `APP_DEBUG=false` en producción
- Rate limiting básico activo
- Validación server-side estricta
