---
task_id: "security-003-persistence"
module: "Security"
agent: "Agente C - Repositorios, Modelos y Persistencia"
title: "Security - Modelos, Repositorios y Persistencia"
priority: "CRITICAL"
estimated_time: "8 hours"
dependencies:
  - "security-001-contracts"
status: "pending"
references:
  - "@laravel/agents/agent-c-persistence.md"
  - "@e-commerce-wa-ml/security/agents_prompt.md"
  - "@e-commerce-wa-ml/security/domain_model.md"
phase: "Fase 1 - Fundamentos"
---

# Task 003: Security - Modelos, Repositorios y Persistencia

## 🎯 Objetivo

Implementar la capa de persistencia del módulo Security: modelos para bloqueos y eventos, repositorios con cache, migraciones y factories.

## 📦 Artefactos a Crear

### Modelos Eloquent (2)

1. **BlockedEntity** - Entidad bloqueada
   - Columns: id, entity_type (enum), entity_value (string), reason (enum), blocked_at, blocked_until, is_permanent (boolean), notes, unblocked_at, timestamps
   - Relationships: none (transversal)
   - Casts: EntityType, BlockReason, BlockDuration
   - Scopes: active(), byType(), expired()

2. **SecurityEvent** - Evento de seguridad
   - Columns: id, event_type (enum), severity (enum), ip_address, phone_number, user_agent, fingerprint, details (json), created_at
   - Relationships: none
   - Casts: SecurityEventType, IpAddress, PhoneNumber, UserAgent
   - Note: Append-only, no updates/deletes

### Repositorios (2)

1. **BlockedEntityRepository** - CRUD bloqueos
   - Methods: find, findByEntity, create, update, delete (unblock), active, expired, cleanExpired

2. **SecurityEventRepository** - Solo escritura
   - Methods: create, findByType, findBySeverity, findByDate, count

### Migraciones (2)

1. **create_blocked_entities_table**
   - entity_type + entity_value (composite index, unique)
   - blocked_until nullable (NULL = permanent)
   - is_permanent boolean
   - Indexes: entity_type, entity_value, blocked_until, is_permanent

2. **create_security_events_table**
   - event_type, severity, ip_address, phone_number
   - created_at only (no updates)
   - Indexes: event_type, severity, created_at, ip_address, phone_number

### Factories (2)

1. **BlockedEntityFactory** - Generate test blocks
   - States: active, expired, permanent, ip_blocked, phone_blocked

2. **SecurityEventFactory** - Generate test events
   - States: rate_limit, captcha_failed, honeypot, low_severity, critical_severity

### No Redis Models
Redis used for rate limiting (ephemeral data, no models needed)

## 🔑 Database Rules

### Indexes Required
- blocked_entities: (entity_type, entity_value) unique
- blocked_entities: blocked_until
- blocked_entities: is_permanent
- security_events: event_type
- security_events: severity
- security_events: created_at
- security_events: ip_address
- security_events: phone_number

### Constraints
- blocked_entities: entity_value not null
- blocked_entities: blocked_until or is_permanent (one required)
- security_events: event_type not null

### Data Retention
- blocked_entities: cleanup expired automatically (job)
- security_events: retain 90 days (job)

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Models use Casts for VOs
- [ ] Repositories implement contracts
- [ ] Unique constraint on entity_type + entity_value
- [ ] Expired blocks cleaned automatically
- [ ] Security events append-only
- [ ] Factories generate valid data

### Técnicos
- [ ] All models are `final`
- [ ] All repositories are `final`
- [ ] Factories with states
- [ ] Integration tests with database
- [ ] Migration rollback works
- [ ] PHPStan level 6+ passes
- [ ] Pint executed

**Status:** ✅ Ready to Implement  
**Fase:** 1 - Fundamentos  
**Depende de:** Task 001
