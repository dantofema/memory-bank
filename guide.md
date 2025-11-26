
# Guía de uso de Memory Bank para proyectos MCP

Esta guía define cómo organizar la documentación técnica de cada proyecto en el Memory Bank, qué registrar y qué evitar. Aplica a todos los proyectos que integren MCP.

---

## Objetivo

Mantener un “contexto compartido” para que agentes MCP (ChatGPT u otros) puedan:
- Consultar arquitectura, decisiones y flujos críticos.
- Registrar cambios importantes sin intervención manual.
- Reducir conocimiento tribal en el equipo.

---

## Ubicación recomendada

```

/home/<usuario>/memory-bank/<proyecto>/

```

Cada proyecto debe estar **fuera del repositorio** y del servidor web.

---

## Estructura de carpetas

```

<proyecto>/
├── README.md                # resumen general
├── architecture/            # definiciones técnicas
│   ├── decisions.md         # ADRs: decisiones irreversibles
│   ├── modules.md           # capas, bounded contexts, dominios
│   └── infrastructure.md    # servicios externos, colas, caching
├── flows/                   # flujos funcionales importantes
│   ├── login.md
│   └── billing.md
├── tickets/                 # tickets o incidentes históricos
│   ├── 2025-001-fix.md
│   └── 2025-002-feature.md
└── knowledge/               # vocabulario y estándares
├── glossary.md
└── database.md

```

---

## Contenidos mínimos por área

### README.md
- “Qué hace el sistema”
- Stack y herramientas clave
- Estado del proyecto (MVP, prod, deprecado)

### architecture/
Registrar **solo decisiones relevantes** que persisten en el tiempo:
- base de datos
- modelo de módulos
- rediseños de API
- elecciones de seguridad

Formato recomendado:
```

## [AAAA-MM-DD] Título corto

* Problema:
* Opciones evaluadas:
* Decisión:
* Justificación:
* Estado: (Propuesto | Activo | Deprecado)

```

### flows/
Flujos de negocio críticos:
- cómo se factura
- cómo se registran leads
- cómo se genera un documento legal

Siempre en **puntos claros**, nada de teoría.

### tickets/
Solo guardar:
- incidentes con aprendizaje
- features que cambiaron arquitectura
- bugs con impacto real

Nada de micro-tareas.

### knowledge/
Información continua:
- glosario de términos
- nombres estándar de tablas y eventos
- reglas que no están escritas en código

---

## Reglas y sugerencias

### Sí hacer
✔ Registrar cambios **que alteran decisiones técnicas**  
✔ Mantener entradas **cortas y accionables**  
✔ Incluir fechas siempre  
✔ Permitir que la IA actualice por comando MCP  
✔ Nombrar archivos por **tema funcional**, no por desarrollador  

### No hacer
✘ Copiar código o lógica completa  
✘ Guardar información sensible: claves, tokens, datos personales  
✘ Documentar lo evidente (lo que se puede ver en el código sin contexto)  
✘ Usar memory-bank como wiki gigante sin mantenimiento  

---

## Buenas prácticas

- **Una decisión = un bloque separado**, fácil de buscar.
- Si un ADR se depreca: marcarlo, no borrarlo.
- Si cambia un flujo: **actualizar el existente**, no crear uno nuevo.
- Pedir a la IA:
  - “actualizá”
  - “agregá una nota”
  - “creá un ticket sobre este cambio”
- Hacer auditorías mensuales:
  - ¿Qué está desactualizado?
  - ¿Qué aprendimos que debemos registrar?

---

## Funcionalidades del Memory Bank MCP

| Acción | Descripción |
|--------|-------------|
| list_projects | Ver proyectos registrados en memory-bank |
| list_project_files | Listar archivos de un proyecto |
| memory_bank_read | Leer contenido de cualquier archivo |
| memory_bank_write | Crear archivo nuevo con contenido |
| memory_bank_update | Actualizar contenido existente |

Estas funciones permiten que la IA:
- Documente tus decisiones a medida que las tomás
- Actualice secciones cuando cambiás arquitectura
- Proponga documentación pendiente analizando el código

---

## Ejemplos de comandos para IA

Crear una decisión:
> memory_bank_write proyecto: geocodex archivo: architecture/decisions.md con contenido: …

Actualizar flujo:
> memory_bank_update proyecto: geocodex archivo: flows/billing.md agregar: “Se habilitó retry…”

Listar archivos:
> list_project_files proyecto: geocodex

---

## Beneficio final

Con muy poco esfuerzo:
- documentación viva
- legado técnico más claro
- onboarding rápido para nuevos devs
- IA que trabaja con conocimiento real del proyecto

---


