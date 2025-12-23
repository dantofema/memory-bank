# Laravel Jobs & Testing Conventions (Asynchronous Persistence)

Este documento define las convenciones técnicas exclusivas para la implementación de **Jobs** y su respectiva estrategia
de **Testing**, asegurando la integridad de los procesos asíncronos y efectos secundarios en la persistencia.

```yaml
version: "3.1"
layer: "Asynchronous Persistence / Side Effects"
stack: [ "Laravel 12", "Pest v4", "Sail" ]
priority: "Idempotency & Decoupling"
quality_gates: "PHPStan Level 6+, Pint"
```

### 1. Filosofía de Procesamiento Asíncrono

* **Notificación de Hechos Pasados**: Los Jobs deben utilizarse principalmente para gestionar efectos secundarios de
  hechos que ya ocurrieron en el negocio (notificaciones, reportes, integraciones, auditoría).
* **No para Decisiones Críticas**: El estado del negocio nunca debe decidirse mediante procesos asíncronos; las
  decisiones deben tomarse mediante interfaces síncronas antes de disparar el Job.
* **Autocontenimiento**: Si un Job pertenece a un módulo específico, todos sus componentes deben residir dentro de dicho
  módulo.

### 2. Estándares de Implementación de Jobs

* **Definición de Clase**: Todas las clases de Jobs deben ser declaradas como **`final`**.
* **Responsabilidad Única**: Deben tener un **único método público** (`handle`) para garantizar una sola
  responsabilidad.
* **Tipado Estricto de Datos**:
    * El constructor y el método `handle` **nunca deben recibir ni devolver arrays**.
    * Se deben utilizar exclusivamente **Value Objects** o objetos **Spatie Laravel Data**.
* **Seguridad y Privacidad**:
    * Está estrictamente prohibido incluir **datos sensibles** (passwords, tokens, tarjetas de crédito) en el payload
      del Job.
    * Se deben incluir únicamente identificadores (IDs), estados, montos o metadatos no sensibles.
* **Resiliencia**: Los Jobs deben ser diseñados para ser **idempotentes** y tolerar ejecuciones duplicadas sin corromper
  la persistencia.

### 3. Convenciones de Testing (Pest v4)

* **Herramienta Exclusiva**: Uso obligatorio de **Pest v4**; no se permite el uso directo de PHPUnit.
* **Estructura de Carpetas**: Los tests deben replicar exactamente la jerarquía de `app/` (o de los módulos) dentro de
  `tests/Feature/`, manteniendo los mismos namespaces.
* **Estrategia de Validación**:
    * **Verificación de Despacho**: En los tests de la lógica de negocio (Actions), se debe verificar que el Job/Evento
      haya sido **despachado** correctamente (*dispatching*).
    * **Aislamiento**: Utilizar **Mocks** para las interfaces cuando mejore la simplicidad y el aislamiento de las
      pruebas del Job.
* **Estilo de Código en Tests**:
    * **Tipado**: Es obligatorio añadir el tipo de retorno **`: void`** en las closures de las funciones `it()` o
      `test()`.
    * **Aserciones**: Se deben **concatenar los `expects`** para mejorar la legibilidad y fluidez del test.

### 4. Configuración del Entorno de Tests

* **Motor de Base de Datos**: Está prohibido cambiar el motor de base de datos a **SQLite** para las pruebas.
* **Variables de Entorno**: No se deben sobreescribir las variables de entorno de PHPUnit en el archivo `.env.testing`.
* **Infraestructura**: Todos los comandos de testing deben ejecutarse a través de **Laravel Sail** (
  `./vendor/bin/sail pest`).

### 5. Checklist de Calidad para Jobs

1.  [ ] ¿La clase es `final` y tiene un único método `handle`?
2.  [ ] ¿El payload está libre de datos sensibles (passwords, tokens)?
3.  [ ] ¿Recibe objetos (Data/VO) en lugar de arrays primitivos?
4.  [ ] ¿Es el proceso **idempotente**?
5.  [ ] ¿El test verifica el despacho (*dispatch*) del Job/Evento?
6.  [ ] ¿Las closures de los tests tienen el tipo de retorno `: void`?

***

**Analogía para el Arquitecto:** Un **Job** es como una "Tarea de Mensajería" tras la firma de un contrato. La decisión
legal se toma síncronamente en la oficina (Interfaces/Actions), y una vez que el hecho es inmutable (pasado), se envía a
un mensajero (Job) para que entregue las copias o actualice los archivos secundarios. El mensajero no decide los
términos del contrato, solo se asegura de que la información llegue a su destino de forma segura y repetible si fuera
necesario.