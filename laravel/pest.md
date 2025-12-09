# Test

## Notas

- No cambies las pruebas a SQLite.
- Evita sobreescribir los env de phpunit en `.env.testing`
- Concatena los expects

## Estilo de código para pruebas con Pest

- Siempre usa Pest para todas las pruebas; no uses PHPUnit directamente ni otros marcos.
- Aplica tipos de retorno explícitos en las closures de las pruebas. En Pest añade `: void` a las closures de `it()`/
  `test()` sin retorno, que requiere la regla `AddClosureVoidReturnTypeWhereNoReturnRector` de Rector.

```
-it('handles modules with mixed enabled and disabled status', function () {
+it('handles modules with mixed enabled and disabled status', function (): void {

```

- Unhandled Exception
  Inspection info: Reports the exceptions that are neither enclosed in a try-catch block nor documented via the @throws
  tag

```
-it('handles modules with mixed enabled and disabled status', function (): void {
+it('handles modules with mixed enabled and disabled status', 
    /** 
    * @throws Exception 
    */
    function (): void {

```