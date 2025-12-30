### Qué y por qué

Tu objetivo es correcto: **llegar al código con todo lo necesario decidido**, pero **sin sobre-documentar ni diseñar de más**. Con los archivos que pasaste, la forma más limpia es **reducir todo a 3 artefactos con límites muy estrictos** y un flujo fijo. Eso te da foco, velocidad y evita discusiones técnicas prematuras.

---

## Propuesta concreta (cerrada y operativa)

### Artefactos finales (solo estos)

1. `project-definition.md`
2. `slices.md`
3. `slice-X.md` (uno por slice)

Nada más antes de codear.

---

## 1️⃣ `project-definition.md` — marco global (una sola vez)

**Para qué sirve**
Definir **el sistema completo** y sus bordes. No se vuelve a tocar salvo cambio fuerte de negocio.

**Incluye (checklist obligatoria)**

* Problema a resolver
* Objetivo del sistema
* Usuarios / roles
* Scope IN / OUT
* Casos de uso a alto nivel
* Reglas de negocio globales
* Riesgos conocidos
* Restricciones (legales, técnicas, tiempo)

**No incluye**

* Orden de desarrollo
* UI
* Modelos
* Tecnología concreta
* Decisiones de arquitectura interna

👉 Si algo cambia seguido, **no va acá**.

---

## 2️⃣ `slices.md` — orden de construcción

**Para qué sirve**
Decidir **qué se construye primero y por qué**. Es tu backlog estratégico.

**Formato recomendado**

```md
# Vertical Slices

Slice 0 — Crear pedido
Valor: el usuario puede comprar.

Slice 1 — Pagar pedido
Valor: el negocio cobra.

Slice 2 — Ver pedidos
Valor: el usuario controla su compra.
```

**Reglas duras**

* Cada slice es end-to-end
* 1–2 líneas máximo
* Sin validaciones
* Sin reglas
* Sin técnica

👉 Si un slice no se puede demo-ear solo, está mal cortado.

---

## 3️⃣ `slice-X.md` — contrato previo al código

Este es **el artefacto clave**. Todo lo que necesitás antes de abrir el editor vive acá.

### Estructura recomendada (fija)

```md
# Slice X — Nombre

## 1. Objetivo
Qué valor entrega este slice.

## 2. Actores
Quién interactúa.

## 3. Flujo principal (Given / When / Then)
Comportamiento esperado.

## 4. Flujos alternativos / errores relevantes
Solo los que importan.

## 5. Reglas de negocio
Invariantes que no se pueden romper.

## 6. Límites explícitos
Qué NO hace este slice.

## 7. Decisiones técnicas mínimas
Solo lo que bloquea el código.
```

### Qué va y qué no va (muy importante)

**✔ Va**

* Comportamiento observable
* Validaciones funcionales
* Estados
* Autorizaciones
* Decisiones irreversibles (ej: enum vs tabla)

**❌ No va**

* Código
* UML
* Estructura de carpetas
* Optimización
* “Después vemos”

👉 Si algo se puede decidir mientras codeás, **no entra**.

---

## Flujo real de trabajo (sin desviarse)

```
project-definition.md        (una vez)
        ↓
slices.md                    (orden)
        ↓
slice-0.md                   (contrato)
        ↓
código del slice 0
        ↓
slice-1.md
        ↓
código del slice 1
```

No se escribe `slice-2.md` hasta terminar el slice anterior.

---

## Regla de calidad (Definition of Ready)

Un `slice-X.md` está listo para codear si:

* Se entiende sin explicar
* QA podría testearlo
* No hay decisiones abiertas
* No menciona “más adelante”
* Entra en 1–2 páginas

Si falla uno → **no se codea**.

---

## Nota de seguridad (obligatoria en cada slice)

Todo `slice-X.md` debe responder explícitamente:

* ¿Quién puede ejecutar el flujo?
* ¿Qué validaciones de input son obligatorias?
* ¿Qué errores se devuelven sin filtrar datos sensibles?
* ¿Qué estados no permitidos se bloquean?

La seguridad **no es un slice**, es parte del contrato.

---

## Resultado esperado

Con estos 3 artefactos:

* Entrás a codear sin improvisar
* Evitás arquitectura anticipada
* Podés delegar slices sin contexto oral
* Tenés trazabilidad clara de decisiones
* El código refleja exactamente lo definido

Si querés, en el próximo paso puedo:

* Ajustarte un **template exacto** de `slice-X.md`
* O validar uno real que ya tengas escrito
