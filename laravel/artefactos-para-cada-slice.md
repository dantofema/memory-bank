# Artefactos por Slice — Enfoque Lean / Vertical Slice First

Este documento define **qué artefactos crear por cada slice**,  
**en qué orden** y **para qué sirven**, priorizando **lean thinking**:
mínimo necesario para avanzar sin deuda conceptual.

---

## Regla base (Lean)

> Si un artefacto no reduce riesgo, ambigüedad o retrabajo  
> **no se crea**.

Cada slice debe ser:

- Independiente
- Deployable
- Medible
- Cerrable

---

## Orden obligatorio de creación (por slice)

1. **Casos de uso + tests esperados**
2. **Lista de tareas técnicas**

Nunca al revés.

---

## Artefacto 1 — Casos de uso + tests esperados

### Qué es

Documento funcional que define **comportamiento observable** del sistema.
Es el contrato del slice.

### Para qué sirve

- Alinear negocio, QA y desarrollo
- Definir el alcance real del slice
- Servir de base para tests automatizados
- Evitar sobre-implementación

### Qué incluye (mínimo)

- Casos de uso principales (happy path)
- Casos de error relevantes (unhappy paths)
- Tests esperados en formato **Given / When / Then**
- Regla de éxito del slice

### Qué NO incluye

- Detalles de implementación
- Clases, frameworks, tablas
- Optimización o escalabilidad futura

### DoD del artefacto

- Entra en **1 página**
- Cubre todo el valor del slice
- Puede ser validado sin ver código

---

## Artefacto 2 — Lista de tareas técnicas

### Qué es

Traducción técnica **directa** del artefacto anterior.
No agrega alcance nuevo.

### Para qué sirve

- Ejecutar el slice sin ambigüedad
- Ordenar el trabajo técnico
- Permitir ejecución por AI / dev

### Qué incluye

- Tareas técnicas atómicas
- Orden de ejecución
- Definition of Done por tarea
- Notas de seguridad
- Fuera de scope explícito

### Qué NO incluye

- Features no cubiertas por los casos de uso
- Refactors “por las dudas”
- Infraestructura futura

### DoD del artefacto

- Cada tarea es implementable
- No hay tareas huérfanas
- Todo mapea a un caso de uso o test

---

## Relación entre artefactos

```

Casos de uso + tests
↓
Lista de tareas técnicas
↓
Código
↓
Tests pasan
↓
Deploy

```

Si algo no baja desde arriba → **no se implementa**.

---

## Regla de corte Lean

Un slice se **detiene** si:

- Los casos de uso crecen → partir el slice
- Los tests no son claros → redefinir alcance
- Las tareas técnicas agregan funcionalidad → error

---

## Resumen rápido

| Orden | Artefacto                      | Propósito                 |
|-------|--------------------------------|---------------------------|
| 1     | Casos de uso + tests esperados | Definir qué hace el slice |
| 2     | Lista de tareas técnicas       | Ejecutar el slice         |

---

## Nota de seguridad (transversal)

En ambos artefactos:

- Considerar validaciones server-side
- Manejo de errores controlado
- No exponer datos sensibles
- No asumir “ambiente seguro”

---

## Regla final

> Si no podés explicar el slice solo con los **casos de uso**  
> todavía no es momento de escribir tareas técnicas.

