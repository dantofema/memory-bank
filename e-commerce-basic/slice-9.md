# Slice 9 — Galería de Imágenes

## 1. Objetivo
El administrador puede cargar múltiples imágenes para un mismo producto, permitiendo que el cliente visualice diferentes ángulos o detalles antes de realizar un pedido.

---

## 2. Actores
- Actor principal: Administrador (gestiona imágenes)
- Actor secundario: Customer (visualiza la galería)

---

## 3. Precondiciones
- Slice 0 y Slice 3 completados: La estructura básica de productos y su administración ya permiten una imagen principal.

---

## 4. Flujo principal (Happy Path)
Given que el Administrador está editando un producto en el panel
When selecciona múltiples archivos de imagen y guarda el producto
Then el sistema almacena todas las imágenes vinculadas al producto.

When el Customer accede al detalle del producto en el catálogo
Then visualiza una galería o carrusel con todas las imágenes cargadas.

---

## 5. Flujos alternativos / errores relevantes
- Given que el Administrador elimina una imagen de la galería
  Then el archivo correspondiente debe ser eliminado del storage para ahorrar espacio.

- Given que el Administrador no carga imágenes adicionales
  Then el sistema sigue mostrando únicamente la imagen principal (o el placeholder).

---

## 6. Reglas de negocio
- El producto mantiene una "Imagen Principal" (usada en listados) y una colección de "Imágenes Secundarias" (usadas en el detalle).
- El sistema debe procesar y optimizar las imágenes para su visualización web.
- Existe un límite máximo de imágenes por producto (ej: 5 o 10) para evitar abusos de almacenamiento.

---

## 7. Estados involucrados (si aplica)
- N/A.

---

## 8. Validaciones clave
- Formatos permitidos: JPG, PNG, WebP.
- Tamaño máximo por archivo (ej: 2MB).
- Validación de cantidad máxima de archivos permitidos por producto.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo Administradores.
- Privacidad: Los archivos se almacenan en un disco público pero con nombres sanitizados/aleatorios para evitar enumeración.

---

## 10. Límites del slice (out of scope)
- No incluye edición de imágenes (recorte, rotación) dentro del panel.
- No incluye videos de productos.
- No incluye zoom avanzado de imágenes o vistas 360°.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Herramienta: Uso de `FileUpload` con la opción `multiple()` en Filament.
- Relación: Nueva entidad `ProductImage` (id, product_id, image_path, sort_order) o uso de Spatie Media Library (si se decide agregar esa complejidad). Para el MVP se prefiere una tabla simple de imágenes o un campo JSON de paths en `products`.
- Almacenamiento: Disco `public`.
- Utilizar siempre sail para el entorno de desarrollo, siempre `./vendor/bin/sail {command}`

---

## 12. Criterio de finalización
- El Administrador puede subir y borrar múltiples imágenes desde el panel de Filament.
- La vista pública del producto muestra todas las imágenes en un componente de galería/carrusel.
- Los archivos se gestionan correctamente en el storage del servidor.
- La eliminación de un producto borra automáticamente todas sus imágenes asociadas.
