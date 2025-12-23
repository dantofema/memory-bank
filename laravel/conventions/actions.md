# Laravel Actions & Testing Conventions

Este documento define los estándares técnicos y arquitectónicos para la implementación de **Actions** y su respectiva
estrategia de **Testing**, actuando como el núcleo de la lógica de negocio y el puente hacia la persistencia en el
proyecto.

```yaml
version: "3.1"
layer: "Business Logic / Persistence Orchestration"
stack: "Laravel 12, Pest v4, Spatie Laravel Data"
priority: "MVP - Speed and Simplicity"
quality_gates: "PHPStan level 6+, Pint"
```

### 1. Filosofía de las Actions

* **Arquitectura sin Servicios**: No se utilizan clases `Service`. Toda la lógica de negocio debe residir exclusivamente
  en clases **Action**.
* **Responsabilidad Única**: Cada Action debe tener un **solo método público** para garantizar que la clase realice una
  única tarea (Single Responsibility).
* **Gobernanza del Presente**: Las Actions (o Interfaces de comunicación síncrona) son las encargadas de tomar
  decisiones, validar capacidades y gobernar el estado actual del negocio.

### 2. Estándares de Implementación

* **Definición de Clase**: Todas las Actions deben ser declaradas como **`final`** y no deben contener métodos
  protegidos.
* **Tipado Estricto de Datos**:
    * Los métodos públicos **nunca deben recibir ni devolver arrays**.
    * Se deben utilizar exclusivamente objetos **Spatie Laravel Data** o **Value Objects** para las firmas de entrada y
      salida.
* **Extracción desde la UI**: Se debe extraer lógica hacia una Action cuando un componente de Livewire Volt exceda las *
  *150 líneas** o cuando la lógica no sea específica de la interfaz de usuario.
* **Manejo de Errores**: Las Actions deben fallar de forma inmediata y propagar excepciones si los datos no cumplen los
  requisitos o si las invariantes de negocio son violadas.

### 3. Convenciones de Testing (Pest v4)

* **Herramienta Exclusiva**: Se debe utilizar **Pest v4** para todas las pruebas. Está prohibido el uso directo de
  PHPUnit.
* **Estrategia de Cobertura**:
    * La cobertura debe ser prácticamente del **100%** para toda funcionalidad nueva.
    * Se deben priorizar los **Feature Tests** sobre los Unit Tests para validar el comportamiento real de la Action.
* **Estructura de Carpetas**: La carpeta `tests/Feature/` debe replicar exactamente la jerarquía y namespaces de la
  carpeta `app/`.

### 4. Estilo de Código en Pruebas

* **Tipado de Closures**: Es obligatorio añadir tipos de retorno explícitos (**`: void`**) en las closures de las
  funciones `it()` o `test()`.
* **Aserciones**: Se deben **concatenar los `expects`** para mejorar la legibilidad y fluidez del test.
* **Entorno de Datos**:
    * **No cambiar el motor de base de datos a SQLite** para las pruebas.
    * No sobreescribir las variables de entorno de PHPUnit en `.env.testing`.
* **Mocks y Fixtures**: Se debe priorizar la testeabilidad especificando fixtures claras y utilizando mocks solo cuando
  mejore la simplicidad y el aislamiento.

### 5. Checklist de Calidad para Actions

1.  [ ] ¿La clase es `final` y posee un único método público?
2.  [ ] ¿Las firmas de entrada y salida utilizan objetos (Data/VO) en lugar de arrays?
3.  [ ] ¿La lógica de negocio reside aquí y no en un componente UI o Controller?
4.  [ ] ¿Se han incluido **Feature Tests** con Pest que cubren los flujos de éxito y error?
5.  [ ] ¿Las closures de los tests tienen el tipo de retorno `: void`?
6.  [ ] ¿El código pasa el análisis de **PHPStan nivel 6** y el formato de **Pint**?

***

**Analogía para el Arquitecto:** Considere que una **Action** es un "Escribano Público". No es una oficina general que
hace de todo (Service), sino un profesional que ejecuta un solo trámite legal específico. Para que trabaje, usted debe
entregarle formularios oficiales (Data Objects), no una pila de papeles desordenados (arrays). Una vez ejecutado el
trámite, el Escribano emite un certificado oficial (Value Object) que garantiza que la base de datos de la ciudad (
Persistencia) ha sido actualizada bajo las leyes vigentes.