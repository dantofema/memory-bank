---
title: "Arquitectura de Agentes IA para Laravel"
version: "1.1"
author: "Alejandro Leone"
last_updated: "2025-12-18"
purpose: "Documentar el enfoque de agentes agnósticos + tareas específicas"
---

# Arquitectura de Agentes IA para Laravel

## Filosofía: Separación de Concerns

Este sistema implementa una arquitectura de **agentes agnósticos del proyecto** que se combinan con **definiciones
específicas** para generar **tareas ejecutables**.

```
Agentes (GENÉRICOS)
    +
Definiciones del Proyecto (ESPECÍFICAS)
    +
Módulos (ESPECÍFICOS)
    =
Tareas (EJECUTABLES)
```

---

## Principios Fundamentales

### 1. Agentes = Metodología Reutilizable

Los agentes definen **CÓMO hacer las cosas**, no **QUÉ hacer**.

- ✅ Son agnósticos del dominio de negocio
- ✅ Son reutilizables entre proyectos
- ✅ Definen estructura, patrones y convenciones
- ✅ Proporcionan ejemplos genéricos como guía
- ❌ NO contienen reglas de negocio específicas
- ❌ NO mencionan entidades concretas del proyecto

### 2. Definiciones = Contexto del Proyecto

Las definiciones describen **QUÉ es el proyecto**.

- ✅ Contexto de negocio
- ✅ Stack tecnológico específico
- ✅ Stakeholders y roles
- ✅ Restricciones y límites
- ✅ Integraciones y dependencias

### 3. Módulos = Funcionalidad Cohesiva

Los módulos describen **QUÉ funcionalidades agrupadas existen**.

- ✅ Entidades y Value Objects del módulo
- ✅ Reglas de negocio específicas
- ✅ Estados y transiciones
- ✅ Relaciones entre entidades
- ✅ Dependencias con otros módulos
- ✅ Alineados con Laravel Modules

### 4. Tareas = Aplicación Específica

Las tareas **combinan agente + definición + módulo** en instrucciones ejecutables.

- ✅ Referencian un agente (metodología)
- ✅ Referencian el proyecto (contexto)
- ✅ Referencian módulos (funcionalidad)
- ✅ Especifican entregables concretos
- ✅ Incluyen criterios de aceptación

---

## Estructura de Directorios

```
memory-bank/
├── laravel/                              # REUTILIZABLE ENTRE PROYECTOS
│   ├── agents/                           # Metodología de desarrollo
│   │   ├── agent-a-contracts.md            # Contratos, Data, VOs, Enums
│   │   ├── agent-b-actions.md              # Actions y Tests Unitarios
│   │   ├── agent-c-persistence.md          # Repositorios y Persistencia
│   │   ├── agent-d-http.md                 # HTTP, Filament y Tests Feature
│   │   └── agent-e-events.md               # Events, Listeners y Jobs
│   │
│   ├── conventions/                      # Convenciones técnicas
│   │   ├── conventions.md                # Convenciones generales
│   │   ├── value-objects.md              # Guía de Value Objects
│   │   └── modules.md                    # Arquitectura de módulos
│   │
│   └── AGENTS_ARCHITECTURE.md            # Este documento
│
├── {proyecto}/                           # ESPECÍFICO DEL PROYECTO
│   ├── project_definition.md             # Definición del proyecto
│   │
│   ├── modular_architecture.md                 # Modular Architecture Overview
│   │
│   ├── modules/                          # Módulos del sistema
│   │   ├── {module}-module.md            # Un archivo por módulo
│   │   └── ...
│   │
│   └── tasks/                            # Tareas ejecutables
│       ├── {module}/                     # Agrupadas por módulo
│       │   ├── 001-{task-name}.md        # Tarea específica
│       │   └── ...
│       └── README.md                     # Índice de tareas
│
└── otro-proyecto/                        # Reutiliza los mismos agentes
    ├── project_definition.md
    ├── modular_architecture.md
    ├── modules/
    └── tasks/
```

---

## Los 5 Agentes y sus Responsabilidades

### Agente A: Contratos, Data, VOs y Enums

**Definir la frontera pública del módulo**

- Contracts (Commands y Queries)
- Data Objects (Spatie Laravel Data)
- Value Objects (con Wireable)
- Enums (PHP enums)
- Tests unitarios de VOs y Enums

**Output**: Tipos, contratos e interfaces públicas.

---

### Agente B: Actions y Tests Unitarios

**Implementar casos de uso del módulo**

- Actions Commands (modifican estado)
- Actions Queries (solo lectura)
- Actions Internal (auxiliares)
- Excepciones de dominio
- Tests unitarios con mocks

**Output**: Lógica de negocio desacoplada de infraestructura.

---

### Agente C: Repositorios, Modelos y Persistencia

**Implementar la capa de persistencia**

- Modelos Eloquent
- Eloquent Casts (VOs)
- Repositorios concretos
- Migraciones
- Factories
- Tests de integración con DB

**Output**: Persistencia y acceso a datos.

---

### Agente D: HTTP, Livewire/Volt, Filament y Tests Feature

**Implementar puntos de entrada del sistema**

- Controllers Web/API
- Form Requests
- API Resources
- Livewire Components y Volt Pages
- Filament Resources y Pages
- Rutas
- Tests Feature y Smoke Tests

**Output**: Interfaces de usuario y APIs.

---

### Agente E: Events, Listeners y Jobs

**Implementar efectos secundarios**

- Events de dominio
- Listeners reactivos
- Jobs asíncronos
- Integraciones indirectas
- Tests de eventos

**Output**: Comunicación asíncrona y efectos secundarios.

---

## Cómo Usar Este Sistema

### Para Desarrolladores Humanos

#### 1. Crear un Nuevo Proyecto

```bash
# Crear estructura del proyecto
mkdir -p memory-bank/mi-proyecto/{modules,tasks}

# Crear definición del proyecto
touch memory-bank/mi-proyecto/project_definition.md

# Opcional: crear diagrama de módulos
touch memory-bank/mi-proyecto/module_diagram.md
```

#### 2. Definir el Proyecto

Crear `project_definition.md` con:

- Arquitectura y enfoque
- Problema y solución
- Stakeholders y roles
- Funcionalidades principales
- Stack técnico
- Convenciones específicas

Ver ejemplo: `e-commerce-wa-ml/project_definition.md`

#### 3. Definir Módulos

Crear diagrama de módulos (opcional pero recomendado):

```bash
# Ver ejemplo en e-commerce-wa-ml/module_diagram.md
vim memory-bank/mi-proyecto/module_diagram.md
```

Crear archivos de módulos en `modules/`:

```bash
touch memory-bank/mi-proyecto/modules/catalog-module.md
touch memory-bank/mi-proyecto/modules/orders-module.md
touch memory-bank/mi-proyecto/modules/auth-module.md
```

Cada módulo debe incluir:

- **Responsabilidad:** Propósito del módulo
- **Entidades:** Modelos y Value Objects
- **Reglas de negocio:** Lógica específica
- **Estados y transiciones:** Si aplica
- **Relaciones:** Dependencias con otros módulos
- **Endpoints:** APIs o rutas principales
- **Tipo:** Core, Transversal o Standard

#### 4. Crear Tareas

Crear archivos en `tasks/`:

```bash
mkdir -p memory-bank/mi-proyecto/tasks/order
touch memory-bank/mi-proyecto/tasks/order/001-contracts.md
```

Cada tarea debe referenciar:

- `agent: @laravel/agents/agente-{x}.md`
- `project: @mi-proyecto/project_definition.md`
- `module: @mi-proyecto/modules/{module}-module.md`

#### 5. Ejecutar Tarea

```bash
# Leer los archivos referenciados
cat laravel/agents/agent-a-contracts.md
cat mi-proyecto/project_definition.md
cat mi-proyecto/modules/orders-module.md
cat mi-proyecto/tasks/orders/001-contracts.md

# Aplicar la metodología al contexto específico
# Generar el código siguiendo los ejemplos del agente
```

---

### Para Agentes IA

Un agente IA debe:

#### 1. Parsear Referencias

```python
task = parse("@mi-proyecto/tasks/orders/001-contracts.md")

agent = load(task.references.agent)
project = load(task.references.project)
module = load(task.references.module)
```

#### 2. Combinar Contextos

```python
context = {
    "methodology": agent,           # CÓMO hacer
    "project": project,             # Contexto del proyecto
    "module": module,               # Funcionalidad y reglas
    "task": task,                   # QUÉ crear específicamente
}
```

#### 3. Generar Código

```python
code = generate(
    methodology=context.methodology,
    entities=context.module.entities,
    rules=context.module.rules,
    deliverables=context.task.deliverables
)
```

#### 4. Validar

```bash
./vendor/bin/sail bin rector --dry-run
./vendor/bin/sail bin phpstan
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail test
```

---

## Anatomía de una Tarea

### Template de Tarea

```yaml
---
task_id: "001"
title: "{Module} - {Phase}"
agent: "@laravel/agents/agent-{name}.md"
project: "@{proyecto}/project_definition.md"
module:
  - "@{proyecto}/modules/{module}-module.md"
  - "@{proyecto}/modules/{module2}-module.md"  # Si depende de otro módulo
dependencies: [ ]  # IDs de tareas previas
priority: "high|medium|low"
estimated_hours: 4
status: "pending|in_progress|completed"
---

# Tarea {task_id}: {Title}

## Context

  Breve descripción de la tarea y su propósito.

**Referencias** :
  - Metodología: { agent }
  - Proyecto: { project }
  - Módulo(s): { module }

---

## Alcance Específico

  ### {Tipo de Artefacto} a Crear

  **{ Nombre del Artefacto }** :
  ```php
  // Estructura o firma específica
```

### Reglas de Negocio del Dominio

1. **Regla 1**: descripción
2. **Regla 2**: descripción

---

## Entregables

- [ ] Archivo 1
- [ ] Archivo 2
- [ ] Tests con cobertura 100%
- [ ] PHPStan level 6 sin errores
- [ ] Pint ejecutado

---

## Validación de Calidad

```bash
./vendor/bin/sail composer run phpstan
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail test --filter={Module}
```

---

## Criterios de Aceptación

1. [ ] Criterio 1
2. [ ] Criterio 2
3. [ ] Criterio 3

```

---

## Ventajas de Este Enfoque

### 🔄 Reutilización
- **Un conjunto de agentes** sirve para **N proyectos**
- Mejoras en agentes benefician a todos los proyectos
- DRY a nivel metodológico

### 📦 Escalabilidad
- Agregar nuevo proyecto: crear carpeta + definiciones
- Agregar nuevo módulo: crear tareas en `tasks/`
- No modificar agentes existentes

### 🔧 Mantenibilidad
- Cambio en metodología → actualizar agente (afecta todos)
- Cambio en dominio → actualizar definición (solo ese proyecto)
- Nueva funcionalidad → crear task (no modifica nada más)

### 🧩 Composición
- Metodología + Contexto = Instrucciones
- Separación clara de responsabilidades
- Fácil de razonar y debuggear

### 🤖 IA-Friendly
- Referencias explícitas con `@`
- Estructura predecible
- Parseable automáticamente
- Context injection claro

---

## Antipatrones a Evitar

### ❌ NO: Agentes Específicos por Proyecto

```

agent-contracts-ecommerce.md
agent-contracts-blog.md
agent-contracts-crm.md

```

**Problema**: Duplicación, difícil de mantener.

### ❌ NO: Mezclar Metodología y Módulo

```markdown
# Agente A

## Crear OrderStatus Enum para Orders
...
```

**Problema**: Agente ya no es reutilizable.

### ❌ NO: Tareas sin Referencias

```yaml
---
task_id: "001"
title: "Crear Orders Module"
# No referencias a agent, project, module
---
```

**Problema**: Falta contexto, no es ejecutable.

### ✅ SÍ: Composición Clara

```
laravel/agents/agent-contracts.md (genérico)
  +
ecommerce/project_definition.md
  +
ecommerce/modules/orders-module.md
  +
ecommerce/tasks/orders/001-contracts.md
  =
Instrucciones completas y ejecutables
```

---

## Flujo de Trabajo Completo

### Fase 1: Setup Inicial

```bash
# 1. Crear estructura del proyecto
mkdir -p memory-bank/mi-proyecto/{modules,tasks}

# 2. Copiar template de project_definition
cp laravel/templates/project_definition.template.md \
   mi-proyecto/project_definition.md

# 3. Editar definición del proyecto
vim mi-proyecto/project_definition.md
```

### Fase 2: Definir Módulos

```bash
# 4. Crear diagrama de módulos (recomendado)
touch mi-proyecto/module_diagram.md
vim mi-proyecto/module_diagram.md

# 5. Crear archivos de módulos
touch mi-proyecto/modules/{module}-module.md

# 6. Documentar responsabilidades y reglas
vim mi-proyecto/modules/{module}-module.md
```

### Fase 3: Planificar Tareas

```bash
# 6. Crear estructura de tareas
mkdir -p mi-proyecto/tasks/{module}

# 7. Crear tareas siguiendo el orden de agentes
touch mi-proyecto/tasks/{module}/001-contracts.md      # agent-contracts
touch mi-proyecto/tasks/{module}/002-actions.md        # agent-actions
touch mi-proyecto/tasks/{module}/003-persistence.md    # agent-persistence
touch mi-proyecto/tasks/{module}/004-http-livewire-filament.md  # agent-http
touch mi-proyecto/tasks/{module}/005-events.md         # agent-events (opcional)

# 8. Crear índice de tareas
touch mi-proyecto/tasks/README.md
```

### Fase 4: Ejecutar Tareas

```bash
# 9. Por cada tarea (humano o IA):
#    a) Leer agente referenciado
#    b) Leer project_definition
#    c) Leer módulo(s) referenciado(s)
#    d) Aplicar metodología al contexto
#    e) Generar código
#    f) Validar con checklist

# 10. Verificar entregables
./vendor/bin/sail composer run phpstan
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail test

# 11. Marcar tarea como completada
# Actualizar status en frontmatter: completed
```

---

## Ejemplos de Referencia

### Proyecto Completo: E-Commerce WhatsApp + ML

```
memory-bank/
├── laravel/
│   └── agents/              # Agentes genéricos (reutilizables)
│
└── e-commerce-wa-ml/
    ├── project_definition.md
    │
    ├── module_diagram.md    # Diagrama Mermaid de módulos
    │
    ├── modules/
    │   ├── auth-module.md
    │   ├── catalog-module.md
    │   ├── cart-module.md
    │   ├── orders-module.md
    │   ├── payments-module.md
    │   ├── whatsapp-module.md
    │   ├── security-module.md
    │   └── reports-module.md
    │
    └── tasks/
        ├── orders/
        │   ├── 001-contracts.md
        │   ├── 002-actions.md
        │   ├── 003-persistence.md
        │   ├── 004-http-livewire-filament.md
        │   └── 005-events.md
        │
        ├── catalog/
        │   └── ...
        │
        ├── payments/
        │   └── ...
        │
        └── README.md
```

Ver: `e-commerce-wa-ml/` para implementación completa.

---

## FAQ

### ¿Los agentes pueden tener ejemplos específicos?

**Sí**, pero deben ser ejemplos **genéricos** que ilustran la metodología. Por ejemplo:

- "Catalog/Product" como dominio de ejemplo
- Aclarar que debe adaptarse al dominio del proyecto
- El ejemplo muestra la estructura, no dicta el contenido

### ¿Puedo modificar los agentes?

**Sí**, pero considera:

- ¿El cambio mejora la metodología en general?
- ¿O es específico de un proyecto?
- Si es específico → va en la task, no en el agente
- Si es metodológico → sí, modificar el agente

### ¿Cuántos módulos debo crear?

**Depende del proyecto**. Reglas generales:

- Un módulo por funcionalidad cohesiva (Orders, Catalog, Payments, etc.)
- Alinear con Laravel Modules si ya los usas
- Identificar módulos Core, Transversales y Standard
- Evitar módulos demasiado granulares o muy acoplados
- Balance entre cohesión y separación de responsabilidades
- Usar `module_diagram.md` para visualizar dependencias

### ¿Debo crear todas las tareas de antemano?

**No necesariamente**. Puedes:

- Crear tareas just-in-time
- Empezar con tareas de alta prioridad
- Iterar según avance del proyecto
- Refinar tareas basándote en aprendizajes

### ¿Qué pasa si un proyecto necesita un agente especial?

Evalúa dos opciones:

1. **¿Es metodología reutilizable?** → Crear nuevo agente genérico
2. **¿Es específico del proyecto?** → Documentar en la task

Ejemplo:

- Agente F para GraphQL (reutilizable) ✅
- Task especial para integración Stripe (específico) ✅

---

## Versionado y Evolución

### Versionado de Agentes

Los agentes usan versionado semántico en el frontmatter:

```yaml
version: "1.0"
last_updated: "2025-12-18"
```

**Cambios MAJOR** (1.x → 2.x): Reestructuración significativa
**Cambios MINOR** (x.1 → x.2): Nuevas secciones o ejemplos
**Cambios PATCH** (x.x.1 → x.x.2): Correcciones o clarificaciones

### Compatibilidad

Las tareas pueden especificar versión del agente:

```yaml
agent: "@laravel/agents/agent-a-contracts.md@1.0"
```

Si no se especifica, se usa la última versión.

---

## Herramientas y Scripts (Futuro)

### CLI Helper (Propuesta)

```bash
# Crear nuevo proyecto
agents create-project mi-proyecto

# Crear módulo
agents create-module mi-proyecto orders

# Crear task
agents create-task mi-proyecto orders 001-contracts --agent=contracts

# Ejecutar task
agents run mi-proyecto/tasks/orders/001-contracts.md

# Validar task
agents validate mi-proyecto/tasks/orders/001-contracts.md
```

### Parser de Referencias (Propuesta)

```python
from agents import TaskParser

task = TaskParser.load("@mi-proyecto/tasks/orders/001-contracts.md")
context = task.resolve_references()

print(context.agent)      # Agente A content
print(context.project)    # Project definition
print(context.module)     # Orders module
print(context.task)       # Task content
```

---

## Contribuir

### Mejorar un Agente

1. Identificar mejora metodológica
2. Verificar que beneficia a múltiples proyectos
3. Actualizar agente
4. Incrementar versión
5. Documentar cambios en changelog

### Crear Nuevo Agente

1. Verificar que sea reutilizable
2. Seguir estructura de agentes existentes
3. Incluir sección "Input del Agente"
4. Proporcionar ejemplos genéricos
5. Documentar en este archivo

### Reportar Issues

- Inconsistencias en agentes
- Ejemplos confusos
- Falta de claridad en metodología
- Sugerencias de mejora

---

## Referencias

- **Convenciones del proyecto**: `laravel/conventions/conventions.md`
- **Value Objects**: `laravel/conventions/value-objects.md`
- **Arquitectura de módulos**: `laravel/conventions/modules.md`
- **Proyecto de ejemplo**: `e-commerce-wa-ml/`

---

## Changelog

### v1.1 - 2025-12-18

- **BREAKING**: Cambio de "Dominios" a "Módulos"
- Mejor alineación con Laravel Modules
- Estructura más pragmática y granular
- Agregado `module_diagram.md` como recomendación
- Actualizada toda la documentación con la nueva terminología

### v1.0 - 2025-12-18

- Versión inicial
- Documentación completa del enfoque
- 5 agentes base (A, B, C, D, E)
- Estructura de proyecto de ejemplo

---

**Versión**: 1.1  
**Última actualización**: 2025-12-18  
**Autor**: Alejandro Leone

