# Executive Overview — Envío de Emails con AWS SES

**Proyecto:** E-commerce BaaS
**Etapa:** Próxima fase (post-MVP)

---

## Qué se va a implementar

Incorporar **AWS Simple Email Service (SES)** como proveedor central de **email transaccional** del BaaS, permitiendo que **cada proyecto (cliente)** envíe correos **con su propio dominio**, alta entregabilidad y costos controlados.

---

## Por qué es necesario

El MVP valida funcionalidad, pero **no garantiza entregabilidad ni escalabilidad económica**.
Para un BaaS multi-proyecto:

* El email es **infraestructura crítica**, no un detalle técnico.
* Envíos desde dominios no autenticados → **spam**.
* SMTP genérico o soluciones "rápidas" → **costos altos o reputación baja**.

SES resuelve esto con **reputación gestionada**, **autenticación por dominio** y **costo marginal mínimo**.

---

## Beneficio clave para el negocio

* **Emails llegan a inbox**, no a spam.
* **Costo ultra bajo** (≈ USD 0.10 / 1.000 emails).
* Permite escalar a muchos clientes sin que el email se vuelva un problema financiero.
* Diferenciador claro frente a e-commerce custom improvisados.

---

## Alcance funcional

* Envío de emails transaccionales (órdenes, pagos, notificaciones).
* **Identidad de envío por proyecto** (dominio propio del cliente).
* Validación previa de dominio (SPF, DKIM, DMARC).
* Bloqueo automático de envíos si el dominio no está verificado.
* Integración transparente con el slice de Notificaciones existente.

---

## Qué NO incluye esta etapa

* Email marketing o campañas masivas.
* Builder visual de templates.
* Warm-up avanzado de dominios.
* Dashboard de métricas para clientes finales.

---

## Arquitectura (alto nivel)

```mermaid
E-commerce
   ↓
BaaS (Laravel)
   ↓
AWS SES
   ↓
Gmail / Outlook / Yahoo
```

* La app sigue corriendo en DigitalOcean.
* SES se usa solo como **servicio externo de entrega**.

---

## Impacto técnico

* Agrega una dependencia externa (SES).
* Requiere modelar:

  * Estado de dominio (`pending`, `verified`, `blocked`)
  * Configuración de identidad por proyecto
* No rompe slices existentes.
* Se implementa como **slice incremental**.

---

## Impacto económico

* Costos variables extremadamente bajos.
* Sin fee mensual.
* Escala lineal con uso real.
* Permite **monetizar planes** sin que el email erosione margen.

---

## Riesgos y mitigación

* **Complejidad inicial de setup**
  → Documentación clara + automatización parcial.
* **Errores de DNS del cliente**
  → Validación explícita + mensajes de error claros.
* **Abuso de envío**
  → Rate limiting y cuotas por proyecto.

---

## Nota de seguridad (obligatoria)

* Uso de IAM con permisos mínimos (`ses:SendEmail`).
* Credenciales cifradas.
* Prohibido enviar desde dominios no verificados.
* Aislamiento estricto entre proyectos.
* Logs sin exponer headers ni contenido sensible.

---

## Resultado esperado

Al finalizar esta etapa:

* Cada cliente del BaaS puede enviar emails **como su propio sitio**.
* Los correos llegan a inbox de forma consistente.
* El costo del email deja de ser una preocupación.
* El BaaS queda preparado para crecer sin deuda estructural.
