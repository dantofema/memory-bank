# Slice 5 — SDK Laravel / cliente HTTP reutilizable

## 1. Objetivo

Reducir fricción de integración entregando un SDK Laravel (o cliente HTTP reusable) que encapsule autenticación por API
key, timeouts, errores y llamadas a endpoints del BaaS.

---

## 2. Actores

- Actor principal: Developer (Interno)
- Actores secundarios: ConsumerApp (E-commerce)

---

## 3. Precondiciones

- Endpoints de los slices 1–4 definidos y relativamente estables.
- Existe al menos un e-commerce consumidor donde integrar.

---

## 4. Flujo principal (Happy Path)

Given que el Developer instala el SDK en un e-commerce Laravel
When configura `BAAS_BASE_URL` y `BAAS_API_KEY`
Then el e-commerce puede invocar métodos tipados para notificaciones, eventos y pagos.

Given que el e-commerce realiza una operación (por ejemplo enviar notificación)
When el SDK llama al endpoint correspondiente
Then maneja errores y devuelve una respuesta consistente a la aplicación.

---

## 5. Flujos alternativos / errores relevantes

- Given el BaaS está caído
  Then el SDK devuelve un error claro (excepción tipada o result object) y respeta timeouts.

- Given error 4xx
  Then el SDK no reintenta (salvo casos idempotentes explícitos).

- Given error 5xx o timeout
  Then el SDK puede reintentar con backoff solo en endpoints idempotentes.

---

## 6. Reglas de negocio

- El SDK no debe esconder el contrato: debe exponer errores accionables.
- El SDK debe ser estable y versionado.

---

## 7. Estados involucrados (si aplica)

- N/A.

---

## 8. Validaciones clave

- Validar configuración mínima al boot (base URL y api key presentes).
- Validar timeouts para no bloquear requests del e-commerce.

---

## 9. Autorización y seguridad

- Nunca loggear la API key.
- Validar HTTPS en producción.
- Permitir rotación de key sin deploy (por env vars).

---

## 10. Límites del slice (out of scope)

- No incluye SDKs para otros lenguajes.
- No incluye dashboard de configuración.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Implementación recomendada: paquete Composer interno (monorepo o repo aparte).
- Cliente basado en `Illuminate\Http\Client\PendingRequest` o Guzzle.
- Métodos sugeridos (nombres ilustrativos):
    - `sendNotification()`
    - `emitEvent()`
    - `createPayment()`

---

## 12. Criterio de finalización

- Un e-commerce puede integrarse sin copiar lógica.
- Existe documentación mínima de instalación/uso.
- Manejo de errores consistente y seguro.
