# Laravel Persistence & Testing Conventions

Este documento define las convenciones técnicas exclusivas para la capa de persistencia y su validación mediante tests,
optimizado para el desarrollo con Laravel 12, Filament v4 y Pest v4.

```yaml
version: "3.1"
layer: "Persistence & Data Integrity"
stack: "Laravel 12, Pest v4"
strict_typing: true
quality_gates: "PHPStan Level 6+, Pint"
```

### 1. Value Objects (VO)

Los Value Objects encapsulan lógica de negocio, garantizan invariantes y aportan semántica al dominio.

* **Criterios de Uso**: Implementar un VO si cumple al menos uno: posee reglas de negocio propias, es reutilizable en
  varios contextos, no debe existir en un estado inválido o aporta semántica clara (ej. `Money`, `PhoneNumber`,
  `OrderStatus`).
* **Implementación**:
    * Deben ser clases **`final readonly`** para asegurar la inmutabilidad.
    * El **constructor debe validar invariantes** y lanzar excepciones si los datos son inválidos.
    * Deben implementar la interfaz **`Wireable`** para su uso en Livewire.
* **Organización**: En `app/ValueObjects/{ModelName}/` o `app/ValueObjects/` si son compartidos.
* **Testing**:
    * **Unit Tests obligatorios** para validar el constructor, invariantes y métodos de dominio.
    * **Feature Tests** para verificar su comportamiento con Eloquent Casts.

### 2. Eloquent Casts

Actúan como el puente de transformación entre la base de datos y los Value Objects.

* **Flujo**: Eloquent → Cast → Value Object.
* **Validación**: El Cast debe validar los tipos de datos antes de realizar el mapeo para evitar estados inconsistentes.
* **Organización**: En `app/Casts/{ModelName}/` o `app/Casts/` para los compartidos.

### 3. Modelos Eloquent

Representan la estructura de datos y deben estar estrictamente protegidos para evitar fugas de persistencia hacia otras
capas.

* **Definición**: Todas las clases de modelos deben ser **`final`**.
* **Relaciones**: Deben declararse de forma **explícita** en todos los casos.
* **Enums**: Se deben preferir los **Enums de PHP** sobre los tipos enum nativos de la base de datos.
* **Encapsulamiento**: Las interfaces de comunicación entre módulos **nunca deben devolver modelos Eloquent**; deben
  retornar exclusivamente Value Objects o Data Objects.

### 4. Factories

Son el estándar obligatorio para la generación de datos en entornos de prueba.

* **Obligatoriedad**: Incluir **siempre un Factory** para cada modelo creado.
* **Uso en Tests**: Se utilizan para especificar fixtures y garantizar que los tests operen sobre estados de datos
  conocidos y consistentes.

### 5. Migraciones

Definen el esquema físico de la base de datos con un enfoque en rendimiento y seguridad.

* **Rendimiento**: Incluir **siempre índices** para optimizar las consultas frecuentes.
* **Seguridad Operacional**:
    * Documentar el impacto en producción usando el tag **`migration-affects-production`** en cambios críticos.
    * Prohibido incluir secretos o credenciales en los archivos de migración.

### 6. Convenciones de Testing para Persistencia (Pest v4)

* **Herramienta**: Uso exclusivo de **Pest v4**; no se permite PHPUnit directamente.
* **Estructura**: La carpeta `tests/Feature/` debe **replicar exactamente la estructura** de `app/` (namespaces y
  carpetas).
* **Configuración**: No se debe cambiar el motor de base de datos a SQLite para las pruebas ni sobreescribir variables
  en `.env.testing`.
* **Calidad de Código**:
    * Añadir tipos de retorno explícitos (normalmente **`: void`**) en las closures de `it()` o `test()`.
    * **Smoke Tests obligatorios** para todas las páginas con interfaz de usuario.
    * La cobertura de tests debe ser cercana al **100%** para toda funcionalidad nueva.

***

**Analogía para el Arquitecto:** La capa de persistencia es el "sistema de cimientos" de un edificio. Las **Migraciones
** son el hormigón armado (con índices como refuerzos), los **Modelos** son las columnas estructurales (`final` y
sólidas), y los **Value Objects** son las pruebas de calidad de los materiales: aseguran que solo entre material
certificado (datos válidos) a la estructura, evitando que cualquier grieta (error de negocio) comprometa la integridad
de la construcción.