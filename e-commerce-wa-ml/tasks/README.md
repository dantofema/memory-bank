# Tasks - E-Commerce WhatsApp + ML

Índice de tareas del proyecto organizadas por módulo.

## Arquitectura de Tareas

Cada tarea sigue el patrón:

```
Agente (metodología genérica)
  +
Project Definition (contexto del proyecto)
  +
Domain (entidades y reglas específicas)
  =
Task (instrucciones ejecutables)
```

Ver: `@laravel/AGENTS_ARCHITECTURE.md` para más información.

---

## Estado de Tareas

### Auth Module

| ID | Título | Agente | Estado | Prioridad | Horas |
|----|--------|--------|--------|-----------|-------|
| agente-e-task-01 | Eventos de Dominio del Módulo Auth | E | Pending | High | 1 |
| agente-e-task-02 | Listeners de Auditoría y Seguridad | E | Pending | High | 2 |
| agente-e-task-03 | Job de Limpieza de Tokens Expirados | E | Pending | Medium | 1 |
| agente-e-task-04 | Integración de Eventos en Actions y Validación Final | E | Pending | High | 1.5 |

### Orders Module

| ID | Título | Agente | Estado | Prioridad | Horas |
|----|--------|--------|--------|-----------|-------|
| agente-e-task-05 | Eventos de Dominio del Módulo Orders | E | Pending | High | 2 |
| agente-e-task-06 | Listeners de Auditoría para Módulo Orders | E | Pending | High | 2.5 |

### Agente C Tasks

| ID | Título | Agente | Estado | Prioridad | Horas |
|----|--------|--------|--------|-----------|-------|
| agente-c-task-01 | Repositories Auth | C | Pending | High | 3 |
| agente-c-task-02 | Repositories Catalog | C | Pending | High | 4 |
| agente-c-task-03 | Repositories Orders | C | Pending | High | 5 |
| agente-c-task-04 | Repositories Payments | C | Pending | High | 3 |

### Agente D Tasks

| ID | Título | Agente | Estado | Prioridad | Horas |
|----|--------|--------|--------|-----------|-------|
| agente-d-task-01 | Controllers y Resources Auth | D | Pending | High | 4 |
| agente-d-task-02 | Controllers y Resources Catalog | D | Pending | High | 5 |
| agente-d-task-03 | Controllers y Resources Orders | D | Pending | High | 6 |

---

## Módulos por Agente

### Agente E (Events, Listeners, Jobs)

#### Auth Module
- [ ] agente-e-task-01: Eventos de Dominio
- [ ] agente-e-task-02: Listeners de Auditoría
- [ ] agente-e-task-03: Job de Limpieza
- [ ] agente-e-task-04: Integración y Validación

#### Orders Module
- [ ] agente-e-task-05: Eventos de Dominio
- [ ] agente-e-task-06: Listeners de Auditoría

### Agente C (Repositories)

- [ ] agente-c-task-01: Auth Repositories
- [ ] agente-c-task-02: Catalog Repositories
- [ ] agente-c-task-03: Orders Repositories
- [ ] agente-c-task-04: Payments Repositories

### Agente D (Controllers, Resources)

- [ ] agente-d-task-01: Auth Controllers
- [ ] agente-d-task-02: Catalog Controllers
- [ ] agente-d-task-03: Orders Controllers

---

## Orden de Ejecución Recomendado

### Fase 1: Auth Module (Agente E)
1. agente-e-task-01: Eventos de Dominio Auth
2. agente-e-task-02: Listeners de Auditoría Auth
3. agente-e-task-03: Job de Limpieza
4. agente-e-task-04: Integración y Validación Auth

### Fase 2: Orders Module (Agente E)
5. agente-e-task-05: Eventos de Dominio Orders
6. agente-e-task-06: Listeners de Auditoría Orders

### Fase 3: Repositories (Agente C)
7. agente-c-task-01: Auth Repositories
8. agente-c-task-02: Catalog Repositories
9. agente-c-task-03: Orders Repositories
10. agente-c-task-04: Payments Repositories

### Fase 4: Controllers (Agente D)
11. agente-d-task-01: Auth Controllers
12. agente-d-task-02: Catalog Controllers
13. agente-d-task-03: Orders Controllers

---

## Resumen de Mejoras Implementadas (2025-12-19)

### Alta Prioridad ✅
1. **Retry Backoff en Job**: Agregado `public array $backoff = [10, 30, 60];`
2. **Timeout Configurable**: `config('auth.password_timeout', 60)` en Repository
3. **Eventos de Orders**: Creada task-05 con 4 eventos críticos
4. **Listeners de Orders**: Creada task-06 con auditoría obligatoria

### Media Prioridad ✅
5. **Métricas de Monitoreo**: Warning cuando deleted_count > 1000
6. **Test de Performance**: Test con 10k registros (debe completar en < 5s)
7. **Rate Limiting en Eventos**: Documentación de throttling en endpoints
8. **Configuración de Alertas**: Ejemplos de alertas en sistemas de monitoreo

---

## Cómo Usar Este Sistema

### Para Desarrolladores Humanos

```bash
# 1. Leer la task
cat tasks/order/001-contracts.md

# 2. Leer el agente referenciado
cat ../../laravel/agents/agente-a.md

# 3. Leer el proyecto
cat ../project_definition.md

# 4. Leer el dominio
cat ../domain/order-domain.md

# 5. Aplicar metodología al contexto y generar código

# 6. Validar
./vendor/bin/sail composer run phpstan
./vendor/bin/sail bin pint --dirty
./vendor/bin/sail test
```

### Para Agentes IA

```python
task = load("@ecommerce/tasks/order/001-contracts.md")
agent = load(task.agent)  # @laravel/agents/agente-a.md
project = load(task.project)
domain = load(task.domain)

code = generate(agent, project, domain, task)
validate(code)
```

---

## Referencias

- **Arquitectura de Agentes**: `@laravel/AGENTS_ARCHITECTURE.md`
- **Agentes Genéricos**: `@laravel/agents/`
- **Project Definition**: `@e-commerce-wa-ml/project_definition.md`
- **Dominios**: `@e-commerce-wa-ml/domain/`
- **Convenciones**: `@laravel/conventions/`

---

---

## Estadísticas

- **Total de tareas**: 13
- **Agente E**: 6 tareas (5.5 horas estimadas)
- **Agente C**: 4 tareas (15 horas estimadas)
- **Agente D**: 3 tareas (15 horas estimadas)
- **Total estimado**: 35.5 horas

---

**Última actualización**: 2025-12-19
