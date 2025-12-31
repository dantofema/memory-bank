# Slice 9 — Página de Producto y Cantidad

## 1. Objetivo
Permitir al cliente visualizar la información completa de un producto individual y seleccionar la cantidad exacta de unidades que desea añadir al carrito de compras en una sola acción.

---

## 2. Actores
- Actor principal: Customer (Anónimo)

---

## 3. Precondiciones
- El producto debe existir en la base de datos.
- El sistema de carrito (Slice 1) debe estar operativo.
- El sistema de Slugs (Slice 8) debe estar configurado para resolver las rutas.

---

## 4. Flujo principal (Happy Path)
Given que el Customer está visualizando el listado de productos
When hace clic en el nombre o imagen de un producto
Then el sistema lo redirige a la página de detalle (`/productos/{slug}`) mostrando nombre, descripción, precio, imagen y categoría.

When el Customer selecciona una cantidad (ej. 5) y presiona "Agregar al carrito"
Then el sistema añade las 5 unidades al carrito y muestra una notificación de éxito.

---

## 5. Flujos alternativos / errores relevantes
- Given que se intenta acceder a un producto inexistente o con slug inválido
  Then el sistema debe responder con un error 404.

- Given que el Customer intenta ingresar una cantidad no numérica o menor a 1
  Then el sistema debe forzar el valor a 1 o mostrar un error de validación.

---

## 6. Reglas de negocio
- La cantidad por defecto al cargar la página debe ser 1.
- No se valida el stock máximo (Fuera de alcance según MVP).
- El precio mostrado debe ser el unitario definido en el catálogo.

---

## 7. Estados involucrados
- Carrito de compras: Se actualiza con el `product_id` and the `quantity` seleccionada.

---

## 8. Validaciones clave
- Cantidad: Debe ser un entero positivo mayor o igual a 1.
- Producto: Debe ser un registro válido en la tabla `products`.

---

## 9. Autorización y seguridad
- Acceso público: No se requiere autenticación para ver productos o agregar al carrito.
- Sanitización: Validar que el input de cantidad sea procesado correctamente para evitar manipulaciones.

---

## 10. Límites del slice (out of scope)
- Gestión de stock/inventario.
- Galería de múltiples imágenes por producto.
- Productos relacionados o sugeridos.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Ruta: `/productos/{slug}`.
- Componente: Se utilizará un componente Volt para manejar la reactividad del input de cantidad y la acción de "Agregar".
- Controlador/Acción: El método de adición al carrito debe ser actualizado para recibir la cantidad como parámetro.

---

## 12. Criterio de finalización
- El enlace desde el catálogo al detalle funciona correctamente.
- La página de detalle muestra toda la información de la entidad `Product`.
- Se pueden agregar múltiples unidades de un producto al carrito satisfactoriamente.
- Cumple con los estándares de diseño de Tailwind v4 establecidos en el proyecto.
