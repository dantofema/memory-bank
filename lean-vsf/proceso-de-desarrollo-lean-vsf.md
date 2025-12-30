
### Qué y por qué

Buscás **un proceso conceptual, repetible y liviano** para desarrollo de productos, no específico del sistema de tickets. La idea es **reducir riesgo antes de cada salto de costo** (pensar → diseñar → codear → operar).

Abajo va el **step by step completo**, de punta a punta, en el nivel justo para un desarrollador senior.

---

# Proceso de desarrollo **Lean → Delivery**

## 0. Ultra Lean Definition

**Objetivo:** decidir si vale la pena codear.

**Artefacto**

* `project-definition.md` (1 archivo)

**Incluye**

* Problema
* Objetivos
* Scope
* Roles
* Casos de uso
* Requerimientos
* Dominio
* Riesgos

**Salida**

* Alcance claro
* MVP definido

👉 Si no entra en 1 vista → volver atrás.

---

## 1. Vertical Slices

**Objetivo:** partir el MVP en incrementos entregables de valor.

**Qué se hace**

* Identificar slices end-to-end
* Cada slice cruza UI → lógica → datos

**Ejemplo**

* Slice 0: crear ticket
* Slice 1: responder ticket
* Slice 2: cerrar ticket

**Salida**

* Lista priorizada de slices
* Orden de entrega claro

---

## 2. Casos de uso detallados + tests esperados (sin código)

**Objetivo:** definir comportamiento observable antes de programar.

**Para cada slice**

* Caso de uso principal
* Flujos alternativos relevantes
* Validaciones
* Errores esperados

**Formato**

* Given / When / Then (conceptual)
* Texto plano, sin framework

**Salida**

* Criterios de aceptación claros
* Base para QA y tests

👉 Acá todavía no existe Laravel.

---

## 3. Diseño técnico mínimo (just in time)

**Objetivo:** reducir incertidumbre técnica antes de codear.

**Se define solo lo necesario**

* Modelos involucrados
* Relaciones clave
* Servicios o módulos nuevos
* Decisiones irreversibles

**NO**

* UML completo
* Over-engineering

**Salida**

* Confianza técnica para avanzar
* Sin deuda anticipada

---

## 4. Desglose en tareas técnicas

**Objetivo:** convertir intención en trabajo ejecutable.

**Para cada slice**

* Tareas chicas y concretas
* Orden lógico
* Dependencias explícitas

**Formato**

* Checklist o issue list
* Sin estimar en exceso

**Salida**

* Backlog técnico listo

---

## 5. Desarrollo guiado por tests

**Objetivo:** asegurar que el slice hace lo esperado.

**Orden**

1. Tests del comportamiento definido
2. Código mínimo para pasar tests
3. Refactor

**Tipo de tests**

* Feature / Integration primero
* Unitarios donde aporta valor

---

## 6. Validación del slice

**Objetivo:** confirmar valor real, no solo código correcto.

**Checklist**

* Pasa tests
* Cumple criterios del paso 2
* No rompe slices anteriores
* Se puede demo-ear

👉 Si no se puede mostrar, no está terminado.

---

## 7. Hardening incremental

**Objetivo:** preparar para uso real sin frenar el flujo.

**Incluye**

* Seguridad
* Performance básica
* Logs
* Manejo de errores

**Timing**

* Slice por slice, no al final

---

## 8. Deploy temprano y frecuente

**Objetivo:** reducir riesgo operativo.

* Ambientes simples
* Deploy automático
* Feedback real

---

## 9. Observación y ajuste

**Objetivo:** aprender antes de seguir construyendo.

* Métricas
* Uso real
* Feedback

👉 Puede implicar:

* Ajustar slices futuros
* Cambiar prioridades
* Recortar alcance

---

## 10. Repetir ciclo

Cada slice recorre:

> **Tests esperados → Diseño mínimo → Código → Validación → Deploy**

---

### Resumen mental (one-liner)

> **Pensar → partir → definir comportamiento → diseñar lo justo → construir → validar → repetir**


