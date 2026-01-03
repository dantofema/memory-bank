# Vertical Slices — Envío de Emails con AWS SES

## Slice 0 — Configurar dominio de envío

**Valor**: El cliente puede registrar su dominio para enviar emails.

## Slice 1 — Verificar dominio

**Valor**: El sistema valida que el dominio está correctamente configurado (SPF, DKIM).

## Slice 2 — Enviar email transaccional

**Valor**: El sistema puede enviar emails desde dominios verificados.

## Slice 3 — Bloquear envíos no autorizados

**Valor**: El sistema rechaza envíos desde dominios no verificados o sin permisos.

## Slice 4 — Gestionar cuotas de envío

**Valor**: El sistema controla límites de envío por proyecto para prevenir abuso.
