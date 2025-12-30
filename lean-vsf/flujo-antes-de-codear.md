### Qué y por qué

La confusión es lógica: **1, 2 y 3 parecen lo mismo** si no se fijan **límites claros de artefactos**. La clave ágil es que **cada paso responde a una pregunta distinta** y genera **un artefacto chico, con frontera explícita**. Nada más.

Abajo te lo separo **quirúrgicamente**.

---

# Paso 0 — Ultra Lean Definition

**Pregunta:** *¿Vale la pena construir esto?*
**Artefacto:**

* `project-definition.md`
  **Límite:**
* Define **qué sistema** vamos a hacer.
* **No** define cómo se implementa ni en qué orden.

---

# Paso 1 — Vertical Slices

**Pregunta:** *¿En qué orden entregamos valor?*
**Artefacto:**

* `vertical-slices.md`

**Contenido permitido**

* Lista de slices
* Prioridad
* Descripción en 1–2 líneas

**Ejemplo**

```md
# Vertical Slices

Slice 0 — Crear ticket
Valor: el usuario puede pedir ayuda.

Slice 1 — Responder ticket
Valor: soporte puede interactuar.

Slice 2 — Cerrar ticket
Valor: se completa el ciclo.
```

**Límites**

* ❌ No casos de uso
* ❌ No validaciones
* ❌ No modelos
* ❌ No técnica

👉 Solo **valor entregable end-to-end**.

---

# Paso 2 — Tests esperados (por slice, sin código)

**Pregunta:** *¿Cómo sé que este slice está bien hecho?*
**Artefacto:**

* `slice-X-expected-behavior.md`
  (uno por slice)

**Contenido permitido**

* Casos felices
* Errores relevantes
* Reglas de negocio
* Validaciones

**Ejemplo**

```md
## Slice 0 — Crear ticket

Given un usuario autenticado
When envía un formulario válido
Then el ticket queda en estado "open"

Given un formulario incompleto
Then el sistema rechaza la creación
```

**Límites**

* ❌ No base de datos
* ❌ No controllers
* ❌ No Laravel
* ❌ No diseño técnico

👉 Es **contrato de comportamiento**, no diseño.

---

# Paso 3 — Diseño técnico mínimo (Just in Time)

**Pregunta:** *¿Qué decisiones técnicas necesito tomar antes de codear este slice?*
**Artefacto:**

* `slice-X-tech-notes.md`

**Contenido permitido**

* Modelos involucrados
* Relaciones nuevas
* Servicios o módulos necesarios
* Decisiones irreversibles

**Ejemplo**

```md
## Slice 0 — Tech Notes

- Entidad Ticket
- Estados como enum
- Relación User -> Ticket (1:N)
```

**Límites**

* ❌ No UML completo
* ❌ No código
* ❌ No optimización
* ❌ No abstracciones futuras

👉 Solo lo que **bloquea el código** si no se define.

---

## Relación entre los tres (clave mental)

| Paso | Responde                  | Artefacto      | Cambia      |
| ---- | ------------------------- | -------------- | ----------- |
| 1    | Qué entrego primero       | Orden          | Prioridades |
| 2    | Qué significa “terminado” | Comportamiento | Calidad     |
| 3    | Qué necesito decidir      | Técnica mínima | Riesgo      |

---

## Regla de oro (ágil de verdad)

* Si algo **no cambia una decisión**, no entra.
* Si el artefacto crece → está mal ubicado.
* Cada paso es **descartable y reescribible**.

---

## Flujo resumido

```
project-definition.md
        ↓
vertical-slices.md
        ↓
slice-0-expected-behavior.md
        ↓
slice-0-tech-notes.md
        ↓
código
```
