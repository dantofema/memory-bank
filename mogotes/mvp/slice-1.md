# Slice 1 — Gestión de proyectos + autenticación por API Key

## 1. Objetivo

Registrar proyectos (tiendas) y autenticar cada request vía `X-API-KEY`, resolviendo el contexto `project_id` y
asegurando aislamiento por proyecto.

---

## 2. Actores

- Actor principal: Developer (Interno)
- Actores secundarios: ConsumerApp (E-commerce)

---

## 3. Precondiciones

- Slice 0 implementado.
- Existe base de datos lista.

---

## 4. Flujo principal (Happy Path)

Given que el Developer da de alta un proyecto
When el sistema genera la API Key
Then se muestra la API Key una sola vez y se almacena únicamente un hash.

Given que el ConsumerApp llama a cualquier endpoint protegido
When envía un header `X-API-KEY` válido
Then el BaaS autentica la request, valida que el proyecto esté activo y resuelve `project_id`.

Given que el Developer rota la API Key
When se genera una nueva key para el proyecto
Then la key anterior queda inválida inmediatamente.

---

## 5. Flujos alternativos / errores relevantes

- Given una request sin `X-API-KEY`
  Then responde 401/403 (consistente) sin filtrar pistas.

- Given una API Key inválida
  Then responde 401/403 sin indicar si el proyecto existe.

- Given un proyecto desactivado
  Then responde 403.

---

## 6. Reglas de negocio

- Cada request se asocia a un único proyecto.
- No existe acceso cross-project.

---

## 7. Estados involucrados (si aplica)

- Project: activo / inactivo.

---

## 8. Validaciones clave

- `domain` debe ser un valor consistente (no necesariamente verificado en MVP).
- `plan` debe ser uno de: `free`, `pro` (aunque `pro` no esté activo).
- API keys deben ser criptográficamente seguras y con longitud suficiente.

---

## 9. Autorización y seguridad

- Acceso: Endpoints del BaaS protegidos por API Key excepto webhooks de providers (Slice 4).
- Seguridad:
    - Nunca loggear el header `X-API-KEY`.
    - Almacenar únicamente hash de API key.
    - Rate limiting por proyecto para evitar abuso.

---

## 10. Límites del slice (out of scope)

- No incluye dashboard para clientes.
- No incluye multi-tenant complejo.
- No incluye billing.

---

## 11. Decisiones técnicas mínimas (Just in Time)

- Datos mínimos de `projects`: `name`, `domain`, `plan`, `is_active`, `api_key_hash`.
- Alta de proyecto (MVP): comando Artisan o seeder.
- Contexto: middleware que agrega `project_id` al request.

---

## 12. Criterio de finalización

- Se puede crear un proyecto y obtener una API Key one-time.
- Todas las rutas protegidas rechazan requests sin key o con key inválida.
- Rate limiting por proyecto activo.
- Rotación y revocación funcionan.
