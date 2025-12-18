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

| ID | Módulo | Título | Agente | Estado | Prioridad | Horas |
|----|---------|--------|--------|--------|-----------|-------|
| 001 | Order | Contratos, Data, VOs y Enums | A | Pending | High | 6 |
| 002 | Order | Actions y Tests Unitarios | B | Pending | High | 8 |
| 003 | Order | Repositorios y Persistencia | C | Pending | High | 6 |
| 004 | Order | HTTP, Filament y Tests Feature | D | Pending | High | 10 |
| 005 | Order | Events, Listeners y Jobs | E | Pending | Medium | 4 |

---

## Módulos Planificados

### Order (Pedidos)
- [x] 001 - Contratos, Data, VOs, Enums
- [ ] 002 - Actions
- [ ] 003 - Persistencia
- [ ] 004 - HTTP/Filament
- [ ] 005 - Events/Jobs

### Product (Productos)
- [ ] 011 - Contratos, Data, VOs, Enums
- [ ] 012 - Actions
- [ ] 013 - Persistencia
- [ ] 014 - HTTP/Filament

### Payment (Pagos)
- [ ] 021 - Contratos, Data, VOs, Enums
- [ ] 022 - Actions
- [ ] 023 - Persistencia
- [ ] 024 - HTTP/Filament
- [ ] 025 - Events/Jobs (Webhooks MP)

### Promotion (Promociones)
- [ ] 031 - Contratos, Data, VOs, Enums
- [ ] 032 - Actions
- [ ] 033 - Persistencia
- [ ] 034 - HTTP/Filament

### Category (Categorías)
- [ ] 041 - Contratos, Data, VOs, Enums
- [ ] 042 - Actions
- [ ] 043 - Persistencia
- [ ] 044 - HTTP/Filament

---

## Orden de Ejecución Recomendado

### Fase 1: Fundamentos
1. Order Module (001-005)
2. Product Module (011-014)
3. Category Module (041-044)

### Fase 2: Pagos y Promociones
4. Payment Module (021-025)
5. Promotion Module (031-034)

### Fase 3: Integraciones
6. WhatsApp Integration
7. Mercado Pago Webhooks

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

**Última actualización**: 2025-12-18
