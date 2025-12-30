# Slice 5 — Dashboard de Administración

## 1. Objetivo
El administrador tiene una vista centralizada con métricas clave para monitorear el rendimiento del negocio de un vistazo.

---

## 2. Actores
- Actor principal: Administrador
- Actores secundarios: N/A

---

## 3. Precondiciones
- Slice 1 completado: Existen pedidos en el sistema.
- Slice 3 completado: El panel de administración (Filament) está operativo y el Administrador está autenticado.

---

## 4. Flujo principal (Happy Path)
Given que el Administrador ha iniciado sesión en el panel
When accede a la página de inicio (Dashboard)
Then visualiza widgets con información resumida: total de ventas, cantidad de pedidos recientes y total de productos activos.

---

## 5. Flujos alternativos / errores relevantes
- Given que no hay pedidos registrados
  Then los widgets de ventas y conteo de pedidos muestran "0".

- Given que el Administrador accede desde un dispositivo móvil
  Then el Dashboard es responsivo y reorganiza los widgets para una lectura clara.

---

## 6. Reglas de negocio
- El Dashboard es la página por defecto al entrar al panel administrativo.
- Las métricas deben reflejar el estado actual de la base de datos en tiempo real (o mediante caché de corta duración).
- No se permiten ediciones de datos directamente desde los widgets (son de solo lectura/estadísticos).

---

## 7. Estados involucrados (si aplica)
- N/A (Vista agregada de estados existentes en `Order` y `Product`).

---

## 8. Validaciones clave
- Cálculo de totales: La suma de ventas debe considerar únicamente los pedidos con estado de pago "Pagado" (opcional, según criterio de negocio, para el MVP se puede mostrar el total general o filtrado).
- Formato de moneda: Los montos deben coincidir con el formato utilizado en el catálogo y administración de pedidos.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo usuarios con rol o permisos de Administrador.
- Privacidad: No se muestran datos personales de clientes en el Dashboard, solo métricas agregadas.
- Rendimiento: Las consultas para el Dashboard deben estar optimizadas para no penalizar el tiempo de carga del panel.

---

## 10. Límites del slice (out of scope)
- No incluye reportes detallados para exportar (Excel/PDF).
- No incluye gráficos complejos (barras/torta) si requieren librerías externas pesadas (se priorizan Widgets simples de Filament).
- No incluye comparativas interanuales o mensuales complejas.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Utilizar siempre sail para el entorno de desarrollo, siempre `./vendor/bin/sail {command}`
- Herramienta: `Widgets` de Filament.
- Widgets: `AccountWidget` (saludo) y `StatsOverviewWidget` (métricas).
- Métricas: Total de Ingresos (Suma de `Order.total`), Pedidos Totales, Productos Activos.

---

## 12. Criterio de finalización
- El Administrador ve un resumen de su negocio al loguearse.
- Los números coinciden con la realidad de los pedidos y productos cargados.
- El Dashboard carga rápidamente y es visualmente consistente con el resto del panel.
