---
task_id: "auth-004-http-filament"
module: "Auth"
agent: "Agente D - HTTP, Livewire/Volt, Filament y Tests Feature"
title: "Auth - Integración con Filament y Tests Feature"
priority: "HIGH"
estimated_time: "6 hours"
dependencies:
  - "auth-001-contracts"
  - "auth-002-actions"
  - "auth-003-persistence"
status: "pending"
references:
  - "@laravel/agents/agent-d-http.md"
  - "@e-commerce-wa-ml/auth/domain_model.md"
phase: "Fase 3 - Presentación"
---

# Task 004: Auth - Integración con Filament y Tests Feature

## 🎯 Objetivo

Integrar el módulo Auth con Filament para proporcionar autenticación segura en el backoffice. Implementar rate limiting, protección CSRF, y tests feature completos.

## ✅ Criterios de Aceptación

### Funcionales
- [ ] Login funciona con credenciales válidas
- [ ] Login falla con credenciales inválidas
- [ ] Remember me genera token correctamente
- [ ] Rate limiting bloquea después de 5 intentos
- [ ] Logout invalida sesión y revoca token
- [ ] Dashboard solo accesible con autenticación
- [ ] CSRF protection funciona en formularios

### Técnicos
- [ ] Tests con Pest 4
- [ ] Cobertura 100% de flujos principales
- [ ] PHPStan level 6+ sin errores
- [ ] Session security configurada
- [ ] Rate limiting configurable vía env

**Status:** ✅ Ready to Implement  
**Fase:** 3 - Presentación  
**Depende de:** Task 001, 002, 003
