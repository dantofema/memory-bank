# Slice 3 — Administración de Catálogo

## 1. Objetivo
El administrador puede gestionar de forma centralizada los productos y categorías que se muestran en el catálogo público, asegurando que la oferta comercial esté siempre actualizada.

---

## 2. Actores
- Actor principal: Administrador
- Actores secundarios: N/A

---

## 3. Precondiciones
- El sistema base (Laravel + Filament) está configurado.
- El usuario Administrador tiene acceso autenticado al panel de gestión.
- Las tablas `products` y `categories` están creadas (Slice 0).

---

## 4. Flujo principal (Happy Path)
Given que el Administrador ha iniciado sesión en el panel
When accede al recurso de Categorías y crea una nueva (ej: "Bebidas")
Then la categoría se guarda y queda disponible para asignar a productos.

When el Administrador accede al recurso de Productos y crea uno nuevo completando: Nombre, Precio, Imagen, Categoría y Estado
Then el producto se guarda y aparece inmediatamente en el catálogo público (Slice 0).

When el Administrador edita un precio o una imagen de un producto existente
Then el cambio se refleja globalmente en el sistema.

---

## 5. Flujos alternativos / errores relevantes
- Given que el Administrador intenta eliminar una categoría que tiene productos asociados
  Then el sistema debe impedir la eliminación (o solicitar reasignación) para mantener la integridad referencial.

- Given que el Administrador sube un archivo de imagen no válido (ej: PDF)
  Then el sistema muestra un error de validación y no permite guardar el producto.

- Given que un producto se marca como "Inactivo"
  Then deja de ser visible para el Customer en el catálogo público, pero permanece en el panel admin.

---

## 6. Reglas de negocio
- Un producto debe pertenecer obligatoriamente a una categoría.
- El sistema debe procesar y almacenar las imágenes de los productos en el storage público.
- Los nombres de las categorías deben ser únicos.
- El panel administrativo debe estar protegido por autenticación.

---

## 7. Estados involucrados (si aplica)
- Product: Activo, Inactivo.
- Category: Visible (por defecto).

---

## 8. Validaciones clave
- Campos obligatorios (Producto): Nombre, Precio, Categoría, Imagen.
- Campos obligatorios (Categoría): Nombre.
- Formato de Precio: Numérico positivo.
- Formato de Imagen: JPEG, PNG, WebP (mínimo/máximo tamaño definido).

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo usuarios con rol o permisos de Administrador.
- Seguridad: Protección contra CSRF en todos los formularios de gestión.
- Almacenamiento: Las imágenes se guardan fuera del directorio público directo y se exponen vía symbolic link (`php artisan storage:link`).

---

## 10. Límites del slice (out of scope)
- No incluye gestión de stock/inventario (fuera de scope del MVP).
- No incluye gestión de pedidos (Slice 4).
- No incluye múltiples imágenes por producto (una sola imagen principal por ahora).
- No incluye logs de auditoría de cambios.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Herramienta: **Filament v4** para la generación rápida del panel administrativo.
- Recursos: `CategoryResource` y `ProductResource` de Filament.
- Formas: Uso de `FileUpload` para la carga de imágenes y `Select` para la relación con categorías.
- Middleware: `auth` aplicado a las rutas del panel administrativo.

---

## 12. Criterio de finalización
- El Administrador puede crear, leer, actualizar y eliminar categorías y productos desde el panel.
- Los productos creados/editados se visualizan correctamente en la vista pública del catálogo.
- La carga de imágenes funciona y los archivos se almacenan correctamente.
- El acceso al panel requiere autenticación.
