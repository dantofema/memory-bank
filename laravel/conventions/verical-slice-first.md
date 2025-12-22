
# Vertical Slice First (VSF) — Aplicado a mi perfil de programador senior

## Rol
Arquitecto de Software Senior  
Stack base: Laravel 12 · Livewire · FilamentPHP · PostgreSQL · Redis · TailwindCSS

---

## 1. Qué es Vertical Slice First (VSF)

**Vertical Slice First** es una estrategia de diseño y ejecución donde el desarrollo avanza por **rebanadas funcionales completas**, en lugar de capas técnicas (UI, backend, DB).

Cada slice atraviesa:
- UI / UX mínima usable
- Lógica de dominio
- Persistencia
- Integraciones
- Tests
- Observabilidad básica

El objetivo es **validar valor real temprano**, reducir riesgo y evitar arquitectura especulativa.

---

## 2. Por qué VSF encaja con mi perfil

Alinea con mis fortalezas:
- Diseño de sistemas mantenibles y escalables
- Enfoque en performance y calidad
- Entregas end-to-end sin deuda estructural
- Métricas claras desde el inicio
- Priorización por impacto de negocio

Evita:
- Overengineering inicial
- Capas vacías “por si acaso”
- Backlogs enormes sin feedback real

---

## 3. Principios operativos

1. **Valor antes que cobertura**
   - Un slice aporta valor observable al usuario o al negocio.

2. **Arquitectura emergente, no improvisada**
   - Se define el *esqueleto mínimo*, el resto emerge slice a slice.

3. **Contrato antes que detalle**
   - Interfaces, eventos y boundaries primero.
   - Implementación interna después.

4. **Todo slice es producción-ready**
   - Seguridad, validaciones y tests incluidos desde el primer slice.

---

## 4. Qué define un Vertical Slice

Un slice es válido si cumple **todos** estos puntos:

- Caso de uso claro (acción + resultado)
- UI mínima funcional (no mockups)
- Dominio encapsulado (service / action / use case)
- Persistencia real
- Tests automatizados (feature + unit clave)
- Logging / métricas básicas
- Feature flag o control de acceso

Si falta uno → **no es un slice**, es trabajo parcial.

---

## 5. Orden recomendado de slices (heurística)

1. **Core Value Slice**
   - La razón por la que el sistema existe.

2. **Money / Riesgo Slice**
   - Facturación, pagos, permisos críticos.

3. **Data Integrity Slice**
   - Validaciones, consistencia, reglas duras.

4. **Scale Slice**
   - Performance, cache, colas, async.

5. **Nice-to-have Slice**
   - UX refinada, automatizaciones secundarias.

---

## 6. Estructura técnica de un slice (Laravel)

```

app/
    └─ Casts/
    ├─ Actions/
    ├─ Contracts/
    ├─ Events/
    ├─ Http/
    │   └─ Livewire/
    │   └─ Filament/
    │   └─ Controllers/
    │       └─ API/
    │           └─ V1/
    ├─ Models/
    ├─ Policies/
    ├─ ValueObjects/
└─ tests/
    └─ Feature/
    ├─ Unit/
    └─ Browser/

```

- Slice ≠ módulo gigante
- Slice = **flujo completo y cerrado**
- Comunicación entre slices: interfaces o eventos, nunca acoplamiento directo

---

## 7. Artefactos mínimos por slice

- Use Case Definition (1 párrafo)
- Contract / Interface pública
- Diagrama simple (flujo, no clases)
- Tests como especificación viva

Todo debe entrar en **una sola página**.  
Si no entra, el slice está mal definido.

---

## 8. Métricas de éxito

- Tiempo desde idea → producción
- Cantidad de retrabajo por slice
- Bugs por slice vs por capa
- Feedback real obtenido temprano

---

## 9. Anti-patrones a evitar

- “Primero armemos la base”
- Repositorios genéricos sin caso de uso
- Módulos definidos sin flujo real
- Tests globales sin contexto funcional
- UI desacoplada del dominio real

---

## 10. Regla de oro

> Si no puedo explicar el slice en 30 segundos  
> y no lo puedo poner en producción solo,  
> todavía no está listo para codearse.

---

## Nota de seguridad

Todo slice debe incluir:
- Validación de input (request / DTO)
- Control de acceso explícito
- Manejo de errores sin filtrar datos sensibles
- Logs sin información crítica (PII, tokens)

La seguridad **no es un slice aparte**, es parte de todos.

---

## Resultado esperado

- Arquitectura clara, evolutiva y verificable
- Backlog pequeño y accionable
- Menos deuda técnica acumulada
- Decisiones basadas en uso real, no supuestos

