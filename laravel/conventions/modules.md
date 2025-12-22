# Laravel Modular Architecture & Testing Conventions

Este documento establece las convenciones obligatorias para la **arquitectura de módulos** y su respectiva estrategia de
**testing**, asegurando el desacoplamiento y la integridad del sistema en proyectos Laravel 12.

```yaml
version: "1.0"
pattern: "Modular Monolith"
package: "nwidart/laravel-modules"
communication: "Interfaces (Sync) & Events (Async)"
testing_framework: "Pest v4"
```

### 1. Arquitectura de Módulos

El proyecto utiliza una arquitectura de monolito modular basada en el paquete `nwidart/laravel-modules`.

* **Autocontenimiento**: Todos los cambios relacionados con un módulo deben ser independientes y residir dentro del
  mismo módulo.
* **Estructura**: Las carpetas estándar proporcionadas por el paquete de módulos son suficientes para los requerimientos
  del proyecto.
* **Regla de Oro**: Los módulos emiten eventos, mientras que los módulos periféricos los consumen.

### 2. Comunicación Síncrona: Interfaces (Commands/Queries)

Se utilizan cuando un módulo requiere una **respuesta inmediata y determinista** de otro módulo.

* **Propósito**: Representan capacidades, validaciones o decisiones de negocio que gobiernan el **presente**.
* **Restricciones de Persistencia**:
    * **Nunca deben devolver modelos Eloquent**; deben retornar exclusivamente **Value Objects** o **Spatie Data Objects
      ** para mantener la encapsulación.
    * No deben exponer detalles de la base de datos ni modelos internos.
* **Manejo de Errores**: Deben fallar de forma inmediata y propagar el error al llamador si la operación no puede
  completarse.

### 3. Comunicación Asíncrona: Eventos de Dominio

Se utilizan para **notificar hechos relevantes** que ya ocurrieron en el negocio.

* **Características**: Son señales inmutables, unidireccionales y **sin valor de retorno** (`void`).
* **Semántica**: Comunican hechos del **pasado** y no deben influir en las decisiones de negocio actuales ni en el
  control del flujo.
* **Seguridad**: Está estrictamente prohibido incluir **datos sensibles** (passwords, tokens, tarjetas) en los eventos.
* **Consumo**: Los consumidores deben ser desacoplados e idempotentes, tolerando posibles duplicados.

### 4. Convenciones de Testing para Módulos

El testing es obligatorio para garantizar que el desacoplamiento modular funcione correctamente.

* **Framework**: Uso exclusivo de **Pest v4** con tipado de retorno `: void` obligatorio en las closures de las pruebas.
* **Estructura**: La carpeta `tests/Feature/` debe **replicar exactamente la jerarquía** de carpetas y namespaces de
  `app/` o de los módulos correspondientes.
* **Estrategia por Mecanismo**:
    * **Interfaces**: Se deben **mockear** en los tests para asegurar el aislamiento entre módulos.
    * **Eventos**: Se debe verificar que los eventos sean **despachados** (*dispatching*) correctamente tras una acción
      de negocio.
* **Entorno**: No se debe utilizar SQLite para las pruebas ni sobreescribir variables de entorno de PHPUnit en
  `.env.testing`.
* **Calidad**: Se deben concatenar los `expects` para mejorar la legibilidad y asegurar una cobertura cercana al **100%
  **.

### 5. Checklist de Calidad Modular

1.  [ ] ¿El módulo es autocontenido y los cambios no afectan globalmente de forma inesperada?
2.  [ ] ¿Las interfaces devuelven **Data Objects** o **Value Objects** en lugar de modelos Eloquent?
3.  [ ] ¿Los eventos notifican hechos pasados y no contienen datos sensibles?
4.  [ ] ¿Se han mockeado las interfaces en los tests de otros módulos?
5.  [ ] ¿La estructura de tests en `tests/Feature/` replica la del módulo?

***

**Analogía para el Arquitecto:** Imagine los módulos como países independientes en un tratado de comercio. Las *
*Interfaces** son llamadas telefónicas diplomáticas (síncronas): usted pide información específica y espera el reporte
oficial (Data Object) antes de colgar para tomar una decisión. Los **Eventos** son los periódicos (asíncronos): informan
sobre algo que ya pasó; usted los lee y decide si toma una acción secundaria, pero el hecho ya es inmutable y no puede
ser cambiado.