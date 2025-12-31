# Slice 8 — URLs Amigables (Slugs)

## 1. Objetivo
Mejorar la legibilidad de los enlaces compartidos por los clientes y optimizar el posicionamiento en buscadores (SEO) mediante el uso de slugs descriptivos en las URLs de productos y categorías.

---

## 2. Actores
- Actor principal: Administrador (genera los datos)
- Actor secundario: Customer (consume las URLs)

---

## 3. Precondiciones
- Slice 0 y Slice 3 completados: La estructura de productos y categorías y su administración están funcionando con IDs.

---

## 4. Flujo principal (Happy Path)
Given que el Administrador crea o edita un producto llamado "Mesa de Jardín"
When guarda el registro
Then el sistema genera automáticamente el slug `mesa-de-jardin` y lo persiste en la base de datos.

When el Customer accede a la URL `/p/mesa-de-jardin`
Then el sistema resuelve el slug y muestra el detalle del producto correspondiente.

---

## 5. Flujos alternativos / errores relevantes
- Given que el Administrador intenta crear un producto con un nombre que ya existe (generando un slug duplicado)
  Then el sistema debe añadir un sufijo único (ej: `mesa-de-jardin-1`) para garantizar que la URL sea única.

- Given que el Administrador cambia el nombre de un producto
  Then el slug puede actualizarse (opcional, se recomienda precaución por enlaces rotos) o mantenerse según la política de redirección implementada. Para este slice, se prioriza la generación automática en la creación.

---

## 6. Reglas de negocio
- Los slugs deben ser únicos por entidad (Product, Category).
- Deben contener solo caracteres alfanuméricos en minúsculas y guiones. Sin espacios ni tildes.
- El slug se utiliza como el identificador en la ruta del navegador (`Route Model Binding`).

---

## 7. Estados involucrados (si aplica)
- N/A.

---

## 8. Validaciones clave
- Unicidad del campo `slug` en las tablas `products` y `categories`.
- Formato de slug válido mediante expresiones regulares.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Administrador (edición), Público (visualización).
- Integridad: Evitar la exposición de IDs internos en las URLs públicas.

---

## 10. Límites del slice (out of scope)
- No incluye gestión de redirecciones 301 para slugs antiguos (se asume que el cambio de slug es poco frecuente en un MVP).
- No incluye edición manual del slug por parte del administrador (se autogenera para simplicidad).

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Paquete: Uso de `spatie/laravel-sluggable` o el helper `Str::slug()` de Laravel en el ciclo de vida del modelo (`booted`).
- Base de Datos: Agregar columna `slug` (string, unique, index) a `products` y `categories`.
- Routing: Cambiar el `getRouteKeyName()` en los modelos para que devuelva `slug`.
- Utilizar siempre sail para el entorno de desarrollo, siempre `./vendor/bin/sail {command}`

---

## 12. Criterio de finalización
- Al crear un producto/categoría, el slug se genera correctamente en la DB.
- Las URLs del catálogo público utilizan el slug en lugar del ID.
- Al acceder a una URL con slug válido, se muestra el contenido correcto.
- Se maneja correctamente el error 404 para slugs inexistentes.
