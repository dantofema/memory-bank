# Laravel Livewire Volt & Testing Conventions

Este documento detalla las convenciones técnicas obligatorias para el desarrollo con **Livewire Volt** y su respectiva
estrategia de **testing**, enfocándose en mantener la integridad de la lógica de negocio y la simplicidad del sistema.

```yaml
version: "1.0"
stack: [ "Livewire v3", "Volt", "Pest v4" ]
priority: "Archivos manejables, testeable, mantenible"
quality_gates: "PHPStan level 6+, Pint"
```

### 1. Filosofía de Desarrollo con Volt

* **Sin Controllers**: Las rutas deben llamar directamente a los componentes de Volt; no se permite el uso de
  controladores intermedios.
* **Sin Services**: Se prohíbe el uso de clases de servicio; la lógica debe residir en **Actions** o en el propio
  componente si es mínima.
* **Responsabilidad Única**: Cada componente debe realizar una sola tarea clara.
* **Composición**: Se debe preferir dividir un componente en múltiples sub-componentes pequeños en lugar de extender
  clases.

### 2. Límites de Tamaño y Complejidad

* **Regla de las 150 Líneas**: Un componente Volt **no debe superar las 150 líneas**.
* **Criterios de Rediseño**: Un componente debe ser refactorizado o dividido si presenta:
    * Más de 150 líneas totales.
    * Más de **4 métodos públicos**.
    * Más de **3 propiedades computadas**.
    * Lógica de negocio compleja mezclada con lógica de presentación.

### 3. Estrategias de Extracción y Organización

* **Extracción de Actions**: Toda lógica de negocio compleja que no sea específica de la interfaz de usuario debe
  moverse a una **Action**.
* **Objetos de Formulario**: Para formularios con validación extensa o campos condicionales, se deben extraer **Form
  Objects**.
* **Sub-componentes**: Dividir en piezas de 20-40 líneas cuando un componente maneje múltiples secciones de UI
  independientes.
* **Nomenclatura**:
    * Usar verbos CRUD (`index`, `show`, `create`, `edit`) para acciones principales.
    * Nombres descriptivos en singular para sub-componentes (ej: `product-details`).

### 4. Integración con Persistencia y Datos

* **Value Objects**: Cualquier objeto de valor utilizado en Livewire debe implementar la interfaz **`Wireable`** para
  asegurar su correcta hidratación y deshidratación.
* **Propiedades Computadas**: Deben mantenerse simples y enfocarse en la recuperación de datos necesarios para la vista.
* **Validación**: Se deben lanzar excepciones cuando los datos no cumplan los requisitos, prefiriendo esto sobre el
  retorno de valores nulos.

### 5. Convenciones de Testing (Pest v4)

* **Obligatoriedad**: Cada componente Volt debe tener **Feature Tests obligatorios** con una cobertura que aspire al
  100%.
* **Smoke Tests**: Es obligatorio implementar **Smoke Tests** para todas las páginas con interfaz de usuario en
  `tests/Feature/SmokeTest.php`.
* **Estructura de Carpetas**: Los tests deben replicar exactamente la jerarquía y los namespaces de la carpeta `app/`
  dentro de `tests/Feature/`.
* **Estilo Pest**:
    * Uso exclusivo de **Pest v4** (prohibido PHPUnit directo).
    * Tipado de retorno **`: void`** obligatorio en todas las closures de `it()` y `test()`.
    * **Concatenación de `expects`** para mejorar la legibilidad de las aserciones.
* **Entorno**: No se debe cambiar el motor de base de datos a SQLite para las pruebas ni sobreescribir variables de
  entorno en `.env.testing`.

### 6. Checklist de Calidad para Componentes Volt

1.  [ ] ¿Tiene menos de 150 líneas?
2.  [ ] ¿La lógica de negocio reside en una **Action**?
3.  [ ] ¿La validación compleja está en **FormRequests** o **Form Objects**?
4.  [ ] ¿Existen **Feature Tests** con cobertura completa?
5.  [ ] ¿Pasa el análisis de **PHPStan nivel 6+**?
6.  [ ] ¿Se ha formateado con **Pint**?
7.  [ ] ¿Los métodos de prueba tienen el retorno `: void`?

***

**Analogía para el Arquitecto:** Un componente **Volt** es como una terminal de autoservicio en un banco. Debe ser
visualmente simple y directa. Si el trámite requiere decisiones legales complejas o mover grandes fondos (lógica de
negocio), la terminal no lo procesa internamente; simplemente envía la orden a un oficial especializado (**Action**),
quien valida la operación y devuelve el resultado certificado para que la terminal solo lo muestre al usuario.