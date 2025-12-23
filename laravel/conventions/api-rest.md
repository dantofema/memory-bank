# Laravel API & Testing Conventions (Persistence Layer)

Este documento centraliza los estándares técnicos para la exposición de datos mediante **API REST** y su validación a
través de **Tests**, garantizando la integridad de la persistencia y la consistencia arquitectónica.

```yaml
version: "3.1"
layer: "API REST / Data Exposure"
stack: [ "Laravel 12", "Pest v4", "Spatie Laravel Data" ]
testing_priority: "100% Coverage"
quality_gates: [ "PHPStan Level 6+", "Pint", "Rector" ]
```

### 1. Arquitectura y Estructura de la API

* **Versionado Obligatorio**: Todas las rutas deben estar versionadas bajo el prefijo `api/v1/*`.
* **Estructura de Directorios**: Los controladores deben replicar la estructura de la ruta en
  `app/Http/Controllers/Api/V1/`.
* **Definición de Controladores**:
* Deben ser declarados como clases `final`.
* **Responsabilidad Única**: Solo se permite **un solo método público** por controlador.
* La lógica de negocio compleja debe residir en clases **Action**, no en el controlador.

### 2. Manejo de Datos (Input/Output)

* **Tipado Estricto**: Los métodos públicos **nunca deben recibir ni devolver arrays**; se deben utilizar exclusivamente
  objetos **Spatie Laravel Data** o **Value Objects**.
* **Transformación de Respuesta**: Los endpoints **siempre** deben retornar un `Resource` o `ResourceCollection` (
  Eloquent API Resources).
* **Encapsulación de Persistencia**:
* Las interfaces y controladores **nunca deben devolver modelos Eloquent** directamente hacia el exterior; esto asegura
  que no se expongan detalles internos de la base de datos.
* Se deben usar **Casts** para mapear Value Objects desde Eloquent antes de ser transformados por el Resource.

### 3. Validación y Errores

* **Mecanismos**: Uso de `FormRequest` o clases con `Illuminate\Validation`.
* **Gestión de Fallos**: Lanzar excepciones (`throw`) inmediatamente cuando los datos no cumplan con los requisitos.
* **Invariantes**: Preferir el lanzamiento de excepciones sobre la devolución de valores `null` o strings vacíos (`''`)
  por defecto para garantizar un estado consistente.

### 4. Convenciones de Testing (Pest v4)

* **Framework Exclusivo**: Uso obligatorio de **Pest v4**. No se permite el uso directo de PHPUnit u otros marcos.
* **Estrategia de Pruebas**:
* **Cobertura**: La cobertura de tests debe ser prácticamente del **100%** para toda funcionalidad nueva.
* **Prioridad**: Se deben priorizar los **Feature Tests** sobre los Unit Tests para validar el comportamiento del
  endpoint en su conjunto.
* **Smoke Tests**: Obligatorios para verificar la disponibilidad de los endpoints en `tests/Feature/SmokeTest.php`.
* **Estilo de Código en Tests**:
* **Tipado**: Añadir tipos de retorno explícitos (`: void`) en las closures de `it()` y `test()`.
* **Aserciones**: Concatenar los `expects` para mejorar la legibilidad.
* **Configuración del Entorno**:
* **No cambiar el motor de base de datos a SQLite** para las pruebas.
* No sobreescribir variables de entorno de PHPUnit en el archivo `.env.testing`.

### 5. Herramientas de Pruebas Manuales y Documentación

* **Pruebas de Integración**: Se debe incluir un archivo **`.http`** para pruebas rápidas y un archivo **`.bru` (Bruno)
  ** para testing de APIs.
* **Documentación**: La ruta de los archivos `.http` y `.bru` debe estar documentada en la sección de archivos del plan
  de desarrollo.

### 6. Checklist de Calidad API

1.  [ ] ¿El endpoint está versionado (`v1`)?
2.  [ ] ¿El controlador tiene un único método público y es `final`?
3.  [ ] ¿Se retornan `Resources` en lugar de modelos o arrays?
4.  [ ] ¿Se han incluido archivos `.http` o `.bru` para validación?
5.  [ ] ¿La cobertura de tests con Pest es cercana al 100%?
6.  [ ] ¿Las closures de los tests tienen el retorno `: void`?

***

**Analogía para el Arquitecto:** La **API** es como el mostrador de atención al cliente de un banco. El cliente no entra
a la bóveda (base de datos/modelos); interactúa con un empleado (Controlador) que solo hace una tarea a la vez (Single
Responsibility). Este empleado recibe formularios oficiales (**Data Objects**) y entrega recibos certificados (**API
Resources**), asegurando que, pase lo que pase, la integridad de los fondos (la persistencia) nunca se vea comprometida
por una interacción externa mal formulada.