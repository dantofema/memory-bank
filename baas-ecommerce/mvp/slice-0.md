# Slice 0 — Bootstrap del BaaS (API + DB + health)

## 1. Objetivo

Tener una API operable y deployable, con base de datos lista y un healthcheck confiable para construir el resto del MVP.

---

## 2. Actores

- Actor principal: Developer (Interno)
- Actores secundarios: N/A

---

## 3. Precondiciones

- Existe un repositorio del BaaS (Laravel 12, API-only) inicializado.
- Existe una instancia de PostgreSQL accesible (local vía Docker, o similar).

---

## 4. Flujo principal (Happy Path)

Given que el servicio está configurado con variables de entorno válidas
When el Developer levanta el servicio
Then la API responde un healthcheck exitoso y puede conectarse a la base de datos.

Given que el repositorio contiene migraciones
When el Developer ejecuta migraciones
Then la base queda lista sin intervención manual.

---

## 5. Flujos alternativos / errores relevantes

- Given una configuración de base de datos inválida
  When se consulta el healthcheck
  Then el sistema responde degradado y registra un log claro sin exponer secretos.

- Given que la base de datos no está disponible
  Then el servicio sigue iniciando, pero marca el healthcheck como no saludable.

---

## 6. Reglas de negocio

- N/A (slice técnico).

---

## 7. Estados involucrados (si aplica)

- Servicio: healthy / degraded.

---

## 8. Validaciones clave

- El healthcheck no debe depender de credenciales en claro ni exponer stack traces.
- La salida del healthcheck debe ser estable para tooling (status + version).

---

## 9. Autorización y seguridad

- Acceso: Público o restringido a red interna (según despliegue), pero sin información sensible.
- Seguridad: No debe exponer variables de entorno, credenciales ni detalles internos.
- Protección: Rate limit opcional si es público.

---

## 10. Límites del slice (out of scope)

- No incluye autenticación por API Key (Slice 1).
- No incluye colas en uso real (se prepara, se usa desde Slice 2).

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Endpoint sugerido: `GET /health`.
- Respuesta sugerida: `{ "status": "ok", "version": "...", "timestamp": "..." }`.
- La verificación de DB puede ser un `SELECT 1` simple.

---

## 12. Criterio de finalización

- El servicio se levanta local con Docker y responde el healthcheck.
- Las migraciones se ejecutan correctamente.
- Los logs son legibles y no exponen datos sensibles.
