# Slice 6 — Operatividad mínima (límites, observabilidad y documentación)

## 1. Objetivo

Hacer el MVP operable: límites por proyecto, logs consultables y documentación de variables de entorno y despliegue
reproducible.

---

## 2. Actores

- Actor principal: Developer (Interno)
- Actores secundarios: N/A

---

## 3. Precondiciones

- Slices 0–4 implementados o en estado funcional.

---

## 4. Flujo principal (Happy Path)

Given que existen proyectos consumiendo el BaaS
When ocurre un error en notificaciones, webhooks o pagos
Then el Developer puede diagnosticar mirando logs y registros (sin datos sensibles).

Given que un proyecto supera un umbral de requests
When se aplica rate limiting
Then el BaaS protege sus recursos sin afectar a otros proyectos.

Given que se necesita desplegar el servicio
When el Developer sigue una guía de deploy
Then obtiene un despliegue determinístico (pasos reproducibles).

---

## 5. Flujos alternativos / errores relevantes

- Given logs excesivos
  Then se ajusta nivel y se aseguran formatos consistentes.

- Given rate limit mal calibrado
  Then se parametriza por env o por proyecto para ajuste rápido.

---

## 6. Reglas de negocio

- Rate limiting es por `project_id`, no global.
- Logs deben ser útiles y mínimos.

---

## 7. Estados involucrados (si aplica)

- N/A.

---

## 8. Validaciones clave

- Variables de entorno mínimas documentadas, con valores por defecto seguros.
- Sanitización de logs: no tokens, no API keys, no payload completo si contiene PII.

---

## 9. Autorización y seguridad

- Endpoints internos sensibles (si existen) deben restringirse (por IP, auth adicional o entorno).
- Webhooks públicos (providers) deben validar firma y tener rate limiting.

---

## 10. Límites del slice (out of scope)

- No incluye observabilidad avanzada (metrics/tracing distribuido) más allá de lo mínimo.
- No incluye alertas pagas.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Rate limiting en Laravel con key derivada de `project_id`.
- Formato de logging consistente (JSON si aplica).
- Checklist de deploy y env vars.

---

## 12. Criterio de finalización

- Variables de entorno documentadas.
- Pasos de deploy reproducibles.
- Rate limiting por proyecto activo.
- Logs claros y consultables para notificaciones, webhooks y pagos.
