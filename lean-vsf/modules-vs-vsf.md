### Qué y por qué

La duda es correcta: **VSF (Vertical Slice Flow) y módulos no son lo mismo**. VSF organiza **cómo entregás valor**; los módulos organizan **cómo estructurás el código**. Se cruzan, pero **no se acoplan**.

---

## 1. Relación correcta: VSF ≠ Modules

### VSF

* Eje: **funcional**
* Pregunta: *¿qué valor entrego de punta a punta?*
* Vive en:

    * documentación
    * tests
    * backlog

### Modules (Laravel Modules / dominios)

* Eje: **estructural**
* Pregunta: *¿cómo organizo el código para mantenerlo?*
* Vive en:

    * carpetas
    * namespaces
    * dependencias

👉 Error común: **crear módulos antes de entender los slices**.

---

## 2. Orden recomendado (importante)

### ✔ Paso correcto

1. Ultra Lean Definition
2. Vertical Slices
3. Tests esperados por slice
4. **Primer slice codeado sin módulos formales**
5. Recién ahí evaluar módulos

### ❌ Paso incorrecto

* Definir módulos “porque sí”
* Forzar cada slice a un módulo

---

## 3. ¿Vale la pena usar módulos siempre?

**No.** Y en muchos casos **empeoran** el flujo lean.

### No uses módulos cuando:

* MVP chico
* Equipo chico (1–3 devs)
* Dominio todavía inestable
* Muchas iteraciones esperadas

En esos casos:

* Monolito simple
* Carpetas claras
* Refactor cuando duela

---

## 4. Cuándo SÍ tiene sentido usar módulos

Usá módulos cuando se cumplan **al menos 2–3** de estos puntos:

* Dominios claros y estables
* Equipo mediano/grande
* Desarrollo en paralelo
* Riesgo de acoplamiento alto
* Reglas de negocio densas

Ejemplo claro:

* Tickets
* Billing
* Users
* Notifications

---

## 5. Cómo se integran VSF y módulos (bien hecho)

### Regla clave

> **Un slice puede tocar varios módulos.**
> **Un módulo no es un slice.**

---

### Ejemplo real (tickets)

#### Slice 0 — Crear ticket

**Atraviesa**

* User
* Ticket
* Notification (opcional)

**Pero no crea módulos nuevos.**

---

### Slice 3 — SLA

Acá probablemente:

* nace un módulo `Sla`
* reglas complejas
* lógica propia

👉 El módulo aparece **como consecuencia del slice**, no antes.

---

## 6. Mapeo conceptual

| Concepto    | Vive en      | Cambia    |
| ----------- | ------------ | --------- |
| Slice       | Docs / Tests | Frecuente |
| Módulo      | Código       | Poco      |
| Feature     | UX           | Variable  |
| Caso de uso | Negocio      | Medio     |

---

## 7. Estrategia práctica en Laravel

### Fase inicial

* `app/Models`
* `app/Actions`
* `app/Http`
* Tests por feature

### Fase de madurez

* Extraer a:

    * `Modules/Tickets`
    * `Modules/Billing`
* Manteniendo contratos claros

👉 **Extraer es barato. Reorganizar prematuro es caro.**

---

## 8. Regla de oro (muy importante)

> **VSF guía el orden de construcción.
> Módulos guían el orden del mantenimiento.**

