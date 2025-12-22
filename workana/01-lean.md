# Método Lean para Proyectos de Software

**Adaptado a un desarrollador senior full-stack (Laravel / Backend)**

## Qué es Lean y por qué usarlo

Lean es un enfoque de gestión de proyectos orientado a **maximizar valor** y **minimizar desperdicio**.  
En software, significa construir **solo lo necesario**, validarlo rápido y mejorar de forma continua.

Para un desarrollador único, Lean sirve para:

- Evitar sobre-arquitectura.
- Reducir tiempo perdido en features sin impacto.
- Mantener foco técnico y entregables claros.
- Tomar decisiones basadas en valor, no en suposiciones.

---

## Principios Lean (versión práctica y técnica)

### 1. Valor primero

Valor = lo que resuelve un problema real del usuario o del negocio.

Aplicación práctica:

- Cada feature debe responder: **¿qué problema concreto resuelve?**
- Si no hay usuario/beneficio claro → no se implementa.

---

### 2. Eliminar desperdicio

Desperdicio en proyectos individuales:

- Código no usado.
- Abstracciones prematuras.
- Features “por si acaso”.
- Documentación que nadie consulta.

Regla operativa:
> Si no se usa en la próxima iteración, no se escribe.

---

### 3. Entregar en ciclos cortos

Trabajo en iteraciones pequeñas, funcionales y desplegables.

Aplicación:

- MVP técnico primero.
- Features verticales (backend + tests + mínima UI si aplica).
- Deploy frecuente, aunque sea a staging/local.

---

### 4. Aprender rápido

Cada entrega es un experimento técnico o funcional.

Aplicación:

- Medir con métricas simples: tiempo, errores, feedback.
- Ajustar arquitectura solo cuando el uso real lo exige.

---

### 5. Decidir lo más tarde posible

Postergar decisiones costosas hasta tener información real.

Ejemplos:

- No elegir microservicios sin problemas reales de escala.
- No optimizar queries hasta tener datos de performance.
- No generalizar módulos sin un segundo caso real.

---

## Lean adaptado a un solo desarrollador

### Flujo de trabajo recomendado

```

Idea → Problema → Hipótesis → Implementación mínima → Validación → Ajuste

```

Todo lo hacés vos, pero en **orden consciente**.

---

### Backlog Lean personal

No es un backlog infinito. Es una **lista corta y viva**.

Formato recomendado por tarea:

- Problema
- Solución mínima
- Criterio de éxito técnico/funcional
- Qué NO se va a hacer ahora

---

### Arquitectura Lean

- Modular, pero sin sobre-ingeniería.
- Interfaces solo cuando hay más de un consumidor real.
- Eventos solo si desacoplan un problema concreto.

Stack típico:

- Laravel bien estructurado.
- Tests donde el riesgo es alto.
- Automatización mínima pero sólida (lint + tests).

---

## Reglas Lean para desarrollo individual

- Una tarea a la vez.
- Terminar > empezar.
- Código claro antes que código “elegante”.
- Refactor solo cuando duele.
- Documentar solo decisiones importantes.

---

## Indicadores Lean personales

Medí solo lo útil:

- Tiempo desde idea a deploy.
- Cantidad de retrabajo.
- Bugs en producción.
- Complejidad innecesaria detectada.

---

## Nota de seguridad

Lean **no** significa descuidar seguridad ni calidad:

- Validaciones siempre.
- Manejo correcto de errores.
- Control de acceso desde el día uno.
- Datos sensibles protegidos desde el MVP.

La seguridad no es desperdicio, es **requisito base**.

---

## Resumen

Lean para un desarrollador es:

- Foco absoluto en valor.
- Menos código, más impacto.
- Decisiones técnicas basadas en uso real.
- Proyectos que avanzan sin volverse inmanejables.

Aplicado correctamente, Lean te permite entregar rápido **sin hipotecar el futuro del proyecto**.
