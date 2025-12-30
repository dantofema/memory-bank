# Slice 4 — Administración de Pedidos

## 1. Objetivo
El administrador puede visualizar y gestionar los pedidos realizados por los clientes, permitiendo el seguimiento de los estados de pago y entrega de forma manual.

---

## 2. Actores
- Actor principal: Administrador
- Actores secundarios: N/A

---

## 3. Precondiciones
- Slice 1 completado: Existen pedidos (`Order` y `OrderItem`) en la base de datos.
- Slice 3 completado: El panel de administración (Filament) está operativo y el Administrador está autenticado.

---

## 4. Flujo principal (Happy Path)
Given que el Administrador ha iniciado sesión en el panel
When accede al recurso de Pedidos
Then visualiza un listado con todos los pedidos, mostrando: número, cliente, total, fecha y estados actuales.

When el Administrador selecciona un pedido específico
Then visualiza el detalle completo: datos de contacto del cliente y el desglose de productos comprados (ítems, cantidades, precios unitarios).

When el Administrador cambia el "Estado de Pago" a "Pagado" o el "Estado de Pedido" a "Entregado"
Then el sistema actualiza el registro y persiste el nuevo estado.

---

## 5. Flujos alternativos / errores relevantes
- Given que el Administrador necesita encontrar un pedido específico
  Then utiliza los filtros del listado (por número de pedido, cliente o estado).

- Given que un pedido fue cancelado por el cliente (vía comunicación externa)
  Then el Administrador marca el pedido con el estado "Cancelado" para cerrar el ciclo.

---

## 6. Reglas de negocio
- El cambio de estados es puramente manual y responsabilidad del Administrador (basado en su coordinación externa vía WhatsApp).
- No se permite eliminar pedidos del sistema (Soft Delete o simplemente no habilitar el borrado) para mantener el historial de ventas.
- Los precios e ítems del pedido no deben ser editables (ya fueron congelados en el Slice 1). Solo se gestionan estados.

---

## 7. Estados involucrados (si aplica)
- **Order - Estado Pedido:** Pendiente (default), Entregado, Cancelado.
- **Order - Estado Pago:** Sin Pagar (default), Pagado.

---

## 8. Validaciones clave
- Integridad de Estados: Solo se permiten los valores definidos en los enums/constantes de la entidad `Order`.
- Solo lectura de detalles: Los campos de montos y productos no deben ser editables en el formulario de edición del pedido.

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo: Solo usuarios con rol o permisos de Administrador.
- Privacidad: Los datos de contacto del cliente (teléfono, dirección) solo son visibles dentro del panel administrativo protegido.
- Auditoría: (Opcional para MVP) Se recomienda registrar quién hizo el último cambio de estado.

---

## 10. Límites del slice (out of scope)
- No incluye generación de facturas PDF o tickets.
- No incluye integraciones con servicios de logística/correo.
- No incluye reembolsos automatizados.
- No incluye edición de los productos incluidos en el pedido una vez realizado.

---

## 11. Decisiones técnicas mínimas (Just in Time)
- Recurso: `OrderResource` en Filament.
- Listado: Uso de `Tables\Columns\BadgeColumn` para visualizar estados de forma clara (colores).
- Detalle: Uso de `RelationManager` o `RepeatableEntry` para mostrar los `OrderItems` asociados dentro de la vista del pedido.
- Acciones: Los cambios de estado pueden hacerse mediante el formulario de edición estándar o mediante "Actions" rápidas en el listado.

---

## 12. Criterio de finalización
- El Administrador puede ver la lista de pedidos entrantes.
- El detalle del pedido muestra correctamente qué compró el cliente y dónde vive.
- Los cambios de estado de pago y entrega se guardan correctamente en la base de datos.
- El acceso está restringido únicamente a administradores.
