# Slice 6 — Configuración del Comercio

## 1. Objetivo
El administrador puede personalizar la información básica de su tienda (nombre, teléfono de contacto, dirección) de forma dinámica desde el panel, asegurando que los datos mostrados al cliente y usados para notificaciones sean siempre correctos.

---

## 2. Actores
- Actor principal: Administrador
- Actores secundarios: N/A

---

## 3. Precondiciones
- Slice 3 completado: El panel de administración (Filament) está configurado y operativo.
- El usuario Administrador está autenticado.

---

## 4. Flujo principal (Happy Path)
Given que el Administrador ha iniciado sesión en el panel
When accede a la sección de "Configuración" o "Ajustes del Sitio"
When modifica el "Nombre de la Tienda" o el "Número de WhatsApp para Pedidos" y guarda los cambios
Then el sistema persiste los nuevos valores y los aplica inmediatamente en el encabezado del catálogo y en el destino de las notificaciones.

---

## 5. Flujos alternativos / errores relevantes
- Given que el Administrador deja vacío el número de WhatsApp
  Then el sistema debe mostrar un error de validación, ya que es crítico para el flujo de notificaciones (Slice 2).

- Given que no se ha configurado nada aún (primer inicio)
  Then el sistema muestra valores por defecto (tomados del .env o valores iniciales de la migración).

---

## 6. Reglas de negocio
- Solo existe un único conjunto de configuraciones para la tienda (patrón Singleton).
- El número de WhatsApp configurado aquí tiene prioridad sobre el valor definido en las variables de entorno para el envío de notificaciones.
- Los cambios realizados impactan en tiempo real en la vista pública (Catálogo).
- No se permite la eliminación del registro de configuración, solo su edición.

---

## 7. Estados involucrados (si aplica)
- N/A (Persistencia de valores de configuración).

---

## 8. Validaciones clave
- **WhatsApp:** Debe tener formato internacional (solo números, incluyendo código de país, sin espacios ni signos +).
- **Nombre de la Tienda:** Obligatorio, longitud mínima 3 caracteres.
- **Email de contacto:** Formato de email válido (si se incluye).

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo usuarios con rol o permisos de Administrador.
- Integridad: Solo un registro debe existir en la tabla de configuraciones para evitar inconsistencias de datos.

---

## 10. Límites del slice (out of scope)
- No incluye personalización de colores, fuentes o temas visuales (CSS).
- No incluye carga de Logo (se mantiene como texto por ahora).
- No incluye configuración de múltiples monedas.
- No incluye gestión de redes sociales avanzadas (solo WhatsApp básico).

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Herramienta: Filament `Settings Page` o un Resource con restricción de creación/eliminación.
- Modelo: `Setting` con columnas: `store_name`, `contact_whatsapp`, `contact_email`, `address`.
- Acceso: Crear un Helper o Service Class `Settings` para acceder a estos valores desde cualquier parte de la app (Blade, Jobs, Controllers).
- Utilizar siempre sail para el entorno de desarrollo, siempre `./vendor/bin/sail {command}`

---

## 12. Criterio de finalización
- El Administrador cambia el nombre de la tienda en el panel y este cambia en el encabezado del catálogo.
- El Job de notificación (Slice 2) utiliza el número configurado en este slice.
- El formulario de configuración valida correctamente los datos ingresados.
- El acceso está restringido únicamente a administradores.
