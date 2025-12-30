# Slice 0 — Catálogo de productos

## 1. Objetivo
El cliente puede visualizar los productos disponibles y filtrar por categorías para encontrar lo que desea comprar.

---

## 2. Actores
- Actor principal: Customer (Anónimo)
- Actores secundarios: N/A

---

## 3. Precondiciones
- Existen productos y categorías cargados en la base de datos (vía seeders o carga directa).
- Las imágenes de los productos están disponibles en el servidor/storage.

---

## 4. Flujo principal (Happy Path)
Given que el sistema tiene productos categorizados
When el Customer accede a la página principal del catálogo
Then visualiza la lista completa de productos (nombre, precio, imagen y categoría).

When el Customer selecciona una categoría específica
Then la lista se filtra mostrando únicamente los productos vinculados a esa categoría.

---

## 5. Flujos alternativos / errores relevantes
- Given que una categoría no tiene productos asociados
  Then el sistema muestra un mensaje claro indicando "No hay productos disponibles en esta categoría".

- Given un error en la carga de imágenes
  Then el sistema muestra un placeholder o imagen por defecto para no romper el layout.

---

## 6. Reglas de negocio
- Solo se muestran productos con estado "Activo" (si aplica).
- Los precios se muestran en la moneda local configurada.
- El orden de los productos es por fecha de creación (más nuevos primero) por defecto.

---

## 7. Estados involucrados (si aplica)
- Product: Activo / Inactivo.
- Category: Activa.

---

## 8. Validaciones clave
- Categoría existente: El filtro debe apuntar a un ID/Slug de categoría válido.
- Formato de precio: Debe mostrarse con dos decimales.

---

## 9. Autorización y seguridad
- Acceso: Público y anónimo.
- Seguridad: No se requiere login.
- Protección: Las consultas a la base de datos deben estar protegidas contra inyección.
- Privacidad: No se exponen datos sensibles de la base de datos en las respuestas JSON/HTML.

---

## 10. Límites del slice (out of scope)
- No incluye el botón "Agregar al carrito" (Slice 1).
- No incluye buscador por texto (lucene/algolia/like).
- No incluye paginación ni scroll infinito.
- No incluye gestión de catálogo (Slice 3).

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Entidades: `Product` (id, name, description, price, image_path, category_id, status), `Category` (id, name, description).
- Relaciones: `Product` belongsTo `Category`.
- Almacenamiento: Las imágenes se referencian por path en la base de datos.

---

## 12. Criterio de finalización
- El flujo de visualización y filtrado se puede demo-ear en el navegador.
- Cumple todos los puntos anteriores.
- El código está listo para ser implementado siguiendo las convenciones del proyecto.
