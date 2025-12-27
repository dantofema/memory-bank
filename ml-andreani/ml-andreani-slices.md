# Mapeo Vertical Slice First — Tienda Online con Envíos en Argentina

## Contexto del proyecto

E-commerce propio con:

- Catálogo + carrito + checkout
- Pagos online
- Cuentas de usuario
- Integración logística Andreani
- Sincronización bidireccional con Mercado Libre (stock + descripciones)

Alcance: **cambio mediano**, foco en valor rápido y riesgo controlado.

---

## Suposiciones explícitas (bajo riesgo)

- Stack: Laravel 12 + Livewire + FilamentPHP
- Mercado Libre vía API oficial
- Pasarela de pago local (ej: Mercado Pago)
- No marketplace multi-vendedor (1 tienda)

Si alguna no aplica, se ajusta el orden, no la estrategia.

---

## Slice 0 — Infraestructura mínima (habilitante)

**Objetivo:** poder poner algo en producción sin deuda estructural.

**Incluye**

- Proyecto Laravel productivo
- Auth básica (usuarios)
- Base de datos + migraciones
- Logging y manejo de errores

**No incluye**

- Dominio de negocio complejo
- UI final

> Este slice no entrega valor al usuario, pero habilita todos los demás. Solo uno permitido.

---

## Slice 1 — Core Value: Compra simple end-to-end

**Caso de uso**
> Usuario compra un producto único y recibe confirmación.

**Incluye**

- Producto simple (precio fijo, stock fijo)
- Listado básico
- Carrito
- Checkout sin envío
- Pago exitoso (happy path)
- Orden persistida

**Valor**

- Demuestra que el negocio funciona
- Reduce el mayor riesgo temprano

**Métricas**

- Tiempo compra → orden creada
- Errores de pago

---

## Slice 2 — Usuarios reales

**Caso de uso**
> Usuario se registra, compra y ve su historial.

**Incluye**

- Registro / login
- Órdenes asociadas a usuario
- Perfil mínimo

**Valor**

- Fidelización
- Base para postventa

---

## Slice 3 — Logística Argentina: Andreani

**Caso de uso**
> Usuario cotiza envío Andreani y elige sucursal o domicilio.

**Incluye**

- Integración API Andreani
- Cálculo de costo
- Selección de método
- Persistencia en la orden

**Valor**

- Diferencial local clave
- Riesgo técnico alto → se ataca temprano

---

## Slice 4 — Backoffice operativo (Filament)

**Caso de uso**
> Admin gestiona productos y órdenes sin tocar código.

**Incluye**

- CRUD productos
- Gestión de órdenes
- Estados (pagado, enviado, cancelado)

**Valor**

- Operabilidad real
- Reduce dependencia técnica

---

## Slice 5 — Mercado Libre: sincronización de stock

**Caso de uso**
> Cambio de stock en e-commerce impacta en ML.

**Incluye**

- Vinculación producto ↔ ML
- Sync stock e-commerce → ML
- Manejo de errores y reintentos

**Valor**

- Evita sobreventa
- Impacto directo en ventas

---

## Slice 6 — Mercado Libre: sincronización de descripción

**Caso de uso**
> Actualizo descripción en un solo lugar.

**Incluye**

- Sync descripción e-commerce → ML
- Versionado simple
- Conflictos básicos

**Valor**

- Ahorro operativo
- Consistencia de catálogo

---

## Slice 7 — UX y confianza

**Caso de uso**
> Usuario compra con claridad y confianza.

**Incluye**

- Estados claros del checkout
- Emails transaccionales
- Mensajes de error amigables

**Valor**

- Conversión
- Menos soporte

---

## Slice 8 — Escalabilidad y performance

**Caso de uso**
> El sistema responde bien con carga real.

**Incluye**

- Cache catálogo
- Colas (sync ML, Andreani)
- Índices DB
- Observabilidad básica

**Valor**

- Preparación para crecer
- Menos incidentes

---

## Dependencias críticas (mapa rápido)

- Slice 1 ← Slice 0
- Slice 3 ← Slice 1
- Slice 5 ← Slice 4
- Slice 6 ← Slice 5
- Slice 8 ← todos

---

## Qué NO se hace al inicio

- Promociones complejas
- Multi-moneda
- Reportes avanzados
- Reglas fiscales especiales

Eso va a **slices futuros**, no al core.

---

## Regla de corte

Cada slice:

- Deployable
- Medible
- Con tests
- Con seguridad mínima

Si no cumple → se parte en dos.

---

## Nota de seguridad (obligatoria)

En todos los slices:

- Validación server-side estricta
- Webhooks firmados (pagos, ML)
- Tokens y credenciales fuera del repo
- Logs sin datos sensibles

