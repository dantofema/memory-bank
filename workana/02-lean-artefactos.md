# Artefactos imprescindibles previos a Vibe Coding

**Lean · desarrollador único**

Lista mínima y suficiente. Si falta uno, el código pierde foco. Si sobra uno, estorba.

---

## 1. Problem Statement

**Qué es:** definición clara del problema a resolver.  
**Para qué sirve:** alinear todo el código a un dolor real.  
**Regla:** si no está escrito, no se programa.

---

## 2. Objective / Outcome Definition

**Qué es:** descripción del resultado esperado (Definition of Done personal).  
**Para qué sirve:** saber exactamente cuándo terminar.  
**Regla:** define también lo que queda fuera del scope.

---

## 3. Scope Boundary

**Qué es:** delimitación explícita de alcance funcional y técnico.  
**Para qué sirve:** evitar creep y sobre-ingeniería.  
**Regla:** lo fuera de scope no se toca, aunque sea fácil.

---

## 4. Technical Hypothesis

**Qué es:** suposición técnica principal que guía la implementación.  
**Para qué sirve:** habilitar decisiones rápidas y refactor consciente.  
**Regla:** una hipótesis por iteración.

---

## 5. Primary Flow (Happy Path)

**Qué es:** flujo principal del sistema de punta a punta.  
**Para qué sirve:** pensar en vertical, no en capas.  
**Regla:** sin edge cases en esta etapa.

---

## 6. Minimal Data Model

**Qué es:** modelo de datos estrictamente necesario para el flujo principal.  
**Para qué sirve:** evitar migraciones y entidades innecesarias.  
**Regla:** solo datos que se persisten o consultan ahora.

---

## 7. Key Technical Decisions

**Qué es:** registro breve de decisiones técnicas relevantes.  
**Para qué sirve:** evitar redecisiones y dudas futuras.  
**Regla:** decisión + motivo corto.

---

## 8. Security Baseline

**Qué es:** definición mínima de seguridad para la iteración.  
**Para qué sirve:** no introducir deuda crítica desde el MVP.  
**Regla:** validación, autorización y manejo de errores siempre definidos.

---

## 9. Executable Task Definition

**Qué es:** tarea concreta, pequeña y ejecutable.  
**Para qué sirve:** entrar directo en flow de código.  
**Regla:** debe poder completarse en una sesión.

---

## Resumen operativo

Antes de vibe coding debés tener:

- Un problema claro
- Un resultado definido
- Un alcance cerrado
- Una hipótesis técnica
- Un flujo principal
- Un modelo de datos mínimo
- Decisiones técnicas explícitas
- Un baseline de seguridad
- Una tarea ejecutable

Todo lo anterior debería entrar en **una sola página**.  
Si no entra, todavía no es momento de codear.
