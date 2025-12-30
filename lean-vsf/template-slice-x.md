### Qué y por qué

El `slice-X.md` es el **contrato previo al código**: define comportamiento, límites y decisiones mínimas. Bien hecho, elimina ambigüedad, reduce retrabajo y permite codear sin frenar por decisiones abiertas.

Abajo tenés un **template cerrado**, listo para copiar/pegar. Incluye **límites claros** y **seguridad explícita**.

---

## Template `slice-X.md`

```md
# Slice X — <Nombre del slice>

## 1. Objetivo
Qué valor concreto entrega este slice al usuario o al negocio.
(1–2 frases, sin contexto técnico)

---

## 2. Actores
- Actor principal:
- Actores secundarios (si aplica):

---

## 3. Precondiciones
- Estado del sistema requerido
- Permisos necesarios
- Datos mínimos existentes

---

## 4. Flujo principal (Happy Path)
Given <contexto inicial>
When <acción principal del actor>
Then <resultado observable esperado>

---

## 5. Flujos alternativos / errores relevantes
- Given <condición inválida>
  Then <comportamiento esperado>

- Given <falta de permisos>
  Then <respuesta esperada>

(Solo errores que cambian lógica o reglas, no exhaustivo)

---

## 6. Reglas de negocio
- Regla 1:
- Regla 2:
- Invariantes que no pueden romperse

---

## 7. Estados involucrados (si aplica)
- Estados posibles:
- Transiciones permitidas:
- Transiciones prohibidas:

---

## 8. Validaciones clave
- Campos obligatorios
- Rangos / formatos
- Dependencias entre datos

---

## 9. Autorización y seguridad
- Quién puede ejecutar el flujo
- Qué datos no deben exponerse
- Manejo de errores sin filtrar información sensible
- Controles de acceso explícitos

---

## 10. Límites del slice (out of scope)
- Qué NO hace este slice
- Qué queda para slices futuros

---

## 11. Decisiones técnicas mínimas (Just in Time)
(Solo lo que bloquea el código)
- Entidades nuevas o modificadas:
- Relaciones nuevas:
- Decisiones irreversibles (ej: enum, unicidad, integraciones):

---

## 12. Criterio de finalización
- El flujo se puede demo-ear
- Cumple todos los puntos anteriores
- No rompe slices previos
- Está listo para pasar a código
```

---

## Reglas de uso (no romper)

* No agregar secciones nuevas “por las dudas”.
* Si una sección queda vacía, preguntate si el slice está bien definido.
* Si el archivo crece más de 2 páginas → el slice está mal cortado.
* Todo lo que no cambie una decisión **no entra**.

---

## Nota de seguridad

Nunca avances a código si:

* La sección **Autorización y seguridad** está incompleta.
* Hay reglas de negocio implícitas (“se asume que…”).
* Existen estados no definidos.
