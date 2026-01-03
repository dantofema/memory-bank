# Project Definition — Envío de Emails con AWS SES

## Problema a resolver

El MVP del BaaS e-commerce funciona, pero **no garantiza entregabilidad ni escalabilidad económica del email transaccional**. Los emails enviados desde dominios no autenticados terminan en spam, y las soluciones SMTP genéricas tienen costos altos o baja reputación.

## Objetivo del sistema

Incorporar **AWS Simple Email Service (SES)** como proveedor central de email transaccional del BaaS, permitiendo que cada proyecto (cliente) envíe correos con su propio dominio, alta entregabilidad y costos controlados.

## Usuarios / Roles

- **Administrador del BaaS**: Configura y gestiona la integración con SES
- **Cliente del BaaS**: Propietario de un proyecto e-commerce que necesita enviar emails
- **Usuario final**: Comprador que recibe emails (órdenes, pagos, notificaciones)

## Scope IN

- Envío de emails (órdenes, pagos, notificaciones)
- Identidad de envío por proyecto (dominio propio del cliente)
- Validación previa de dominio (SPF, DKIM, DMARC)
- Bloqueo automático de envíos si el dominio no está verificado
- Integración transparente con el slice de Notificaciones existente
- Rate limiting y cuotas por proyecto
- Logs de envío sin exponer contenido sensible

## Scope OUT

- Email marketing o campañas masivas
- Builder visual de templates
- Warm-up avanzado de dominios
- Dashboard de métricas para clientes finales
- Gestión de rebotes y quejas (bounce/complaint handling)
- A/B testing de emails
- Personalización avanzada de templates

## Casos de uso a alto nivel

1. **Configurar dominio de envío**: Cliente configura su dominio para enviar emails
2. **Verificar dominio**: Sistema valida registros DNS (SPF, DKIM, DMARC)
3. **Enviar email transaccional**: Sistema envía email desde dominio verificado
4. **Bloquear envío no autorizado**: Sistema rechaza envíos desde dominios no verificados
5. **Monitorear cuotas**: Sistema controla límites de envío por proyecto

## Reglas de negocio globales

- **Prohibido enviar desde dominios no verificados**
- **Aislamiento estricto entre proyectos**: Un proyecto no puede enviar desde el dominio de otro
- **Cuotas obligatorias**: Todo proyecto tiene límite de envíos (configurable)
- **Autenticación obligatoria**: SPF + DKIM mínimo, DMARC recomendado
- **Sin contenido sensible en logs**: Headers y contenido no se almacenan completos

## Riesgos conocidos

- **Complejidad inicial de setup**: Configuración DNS puede ser confusa para clientes
- **Errores de DNS del cliente**: Registros mal configurados bloquean envíos
- **Abuso de envío**: Clientes podrían intentar enviar spam
- **Dependencia de AWS**: Caída de SES afecta todos los envíos
- **Costos inesperados**: Uso excesivo podría generar costos no previstos

## Restricciones

### Técnicas

- La app sigue corriendo en DigitalOcean
- SES se usa solo como servicio externo de entrega
- No rompe slices existentes del MVP
- Debe integrarse con sistema de notificaciones actual

### Legales

- Cumplimiento con CAN-SPAM Act (EE.UU.)
- GDPR para datos de usuarios europeos
- Ley de Protección de Datos Personales (Argentina)
- Obligación de incluir opción de unsubscribe en emails promocionales (fuera de scope, pero considerar arquitectura)

### Tiempo

- Implementación post-MVP
- No bloquea lanzamiento inicial
- Prioridad: Media-Alta (necesario para escalar)

### Económicas

- Costo variable extremadamente bajo (≈ USD 0.10 / 1.000 emails)
- Sin fee mensual
- Escala lineal con uso real
- Permite monetizar planes sin erosionar margen
