# Slice 7 — Buscador y Paginación

## 1. Objetivo
El cliente puede localizar rápidamente productos específicos mediante un buscador de texto y navegar por el catálogo de forma eficiente mediante paginación, mejorando la experiencia de usuario y el rendimiento de carga.

---

## 2. Actores
- Actor principal: Customer (Anónimo)
- Actores secundarios: N/A

---

## 3. Precondiciones
- Slice 0 completado: El catálogo base está operativo.
- El sistema cuenta con una cantidad de productos que excede el límite de visualización por página (ej: > 12 productos).

---

## 4. Flujo principal (Happy Path)
Given que el Customer está en la vista del catálogo público
When escribe un término en la barra de búsqueda (ej: "Mesa")
Then el sistema filtra la lista de productos mostrando solo aquellos cuyos nombres o descripciones coincidan con el término.

When el Customer se desplaza al final de la lista de productos
Then visualiza los controles de navegación (Anterior/Siguiente/Números) y puede cambiar de página para ver más productos.

---

## 5. Flujos alternativos / errores relevantes
- Given que el término buscado no coincide con ningún producto
  Then el sistema muestra el mensaje: "No se encontraron productos coincidentes con su búsqueda".

- Given que el Customer filtra por categoría y luego busca por texto
  Then el sistema debe realizar la búsqueda únicamente dentro de los productos de esa categoría.

---

## 6. Reglas de negocio
- La búsqueda no distingue entre mayúsculas y minúsculas (case-insensitive).
- La búsqueda se aplica sobre los campos `name` y `description` del modelo `Product`.
- La cantidad de productos por página se define en la configuración o como una constante (ej: 12).
- Los parámetros de búsqueda y página deben persistir en la URL (query strings) para permitir compartir resultados o volver atrás.

---

## 7. Estados involucrados (si aplica)
- N/A (Filtros de consulta).

---

## 8. Validaciones clave
- El término de búsqueda debe ser sanitizado para evitar ataques de inyección.
- El número de página solicitado debe ser un entero positivo válido.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Público (Anónimo).
- Protección: Uso de bindings de base de datos para prevenir inyecciones SQL en las consultas de búsqueda.

---

## 10. Límites del slice (out of scope)
- No incluye búsqueda "fuzzy" o con corrección ortográfica (tipo Algolia o Meilisearch).
- No incluye filtros avanzados por rango de precios o atributos (color, talle).
- No incluye ordenamiento dinámico (ej: por precio ascendente/descendente).

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Herramienta: **Livewire** (si se usa en el frontend) o controladores estándar con `withQueryString()` en la paginación.
- Eloquent: Uso de `where('name', 'like', "%{$term}%")` y el método `paginate(12)`.
- Utilizar siempre sail para el entorno de desarrollo, siempre `./vendor/bin/sail {command}`

---

## 12. Criterio de finalización
- El buscador filtra correctamente la lista de productos.
- Los controles de paginación funcionan y permiten navegar por todo el catálogo.
- La combinación de filtros (categoría + búsqueda) produce resultados coherentes.
- La navegación "atrás" en el navegador respeta el estado de la búsqueda/página.
