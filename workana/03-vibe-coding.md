# Qué sigue después de los artefactos

**Inicio del Vibe Coding (flujo operativo)**

Si todo entra en una página, ya no se piensa más: **se ejecuta**.
El objetivo ahora es convertir claridad en código funcional lo más rápido posible.

---

## Paso 1. Commit de intención

**Qué es:** un punto de partida explícito.

Acción:

- Crear branch o commit inicial con el objetivo de la iteración.
- Mensaje claro orientado a valor, no a archivos.

Ejemplo:

```

feat: minimal employee creation flow

```

Por qué:

- Marca el inicio del experimento.
- Permite descartar sin fricción.

---

## Paso 2. Vertical Slice First

**Qué es:** implementar el flujo completo más simple posible.

Orden recomendado:

1. Entrada (request / acción)
2. Lógica mínima
3. Persistencia (si aplica)
4. Salida observable

Regla:
> Que funcione feo antes de que sea lindo.

---

## Paso 3. Código sin abstracciones preventivas

**Qué es:** escribir código directo, explícito.

Aplicación:

- Sin interfaces “por si acaso”.
- Sin eventos si hay un solo flujo.
- Sin patrones si no eliminan complejidad hoy.

Regla Lean:
> Duplicar es más barato que generalizar antes de tiempo.

---

## Paso 4. Validación inmediata

**Qué es:** confirmar que el flujo hace lo esperado.

Opciones válidas:

- Test puntual (unit o feature).
- Request manual.
- Script mínimo.

Regla:

- Al menos una forma reproducible de validación.

---

## Paso 5. Micro-refactor consciente

**Qué es:** ordenar solo lo que ya existe.

Acción:

- Renombrar.
- Extraer métodos obvios.
- Eliminar ruido.

Prohibido:

- Reescribir lo que todavía no duele.

---

## Paso 6. Seguridad aplicada

**Qué es:** llevar a código el Security Baseline definido.

Checklist:

- Validaciones reales.
- Autorización explícita.
- Manejo de errores consistente.
- Datos sensibles protegidos.

Si algo no se implementa:

- Se documenta como riesgo aceptado.

---

## Paso 7. Commit funcional

**Qué es:** cierre técnico de la iteración.

Condición:

- Flujo principal completo.
- Código ejecutable.
- Validación repetible.

Mensaje:

```

feat: working minimal flow for X

```

---

## Paso 8. Evaluación Lean

**Qué es:** freno consciente antes de seguir.

Preguntas clave:

- ¿Resuelve el problema definido?
- ¿Apareció complejidad innecesaria?
- ¿Qué decisión técnica quedó en evidencia?

Resultado:

- Se ajustan artefactos.
- Se define la próxima tarea o se corta.

---

## Principio clave del Vibe Coding

El vibe coding **no es improvisar**.
Es **pensar una vez, ejecutar sin fricción y corregir solo con evidencia**.

Si dudás mientras codeás, no seguís:
volvés a la página única, la ajustás y recién ahí continuás.
