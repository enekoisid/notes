# ✅ Mejores Prácticas para el Manejo de Errores en Controladores C#

Este documento resume las opciones más efectivas para mejorar el manejo de errores en controladores C# que gestionan tareas y devuelven resultados a los clientes.

---

## 1. Uso Correcto de ActionResult y Códigos HTTP
- Devuelve siempre el código HTTP adecuado según el tipo de error: `BadRequest (400)`, `NotFound (404)`, `Conflict (409)`, `InternalServerError (500)`, etc.
- Ejemplo:

```csharp
catch (DbUpdateException ex) {
    return StatusCode(StatusCodes.Status409Conflict, new { message = "Conflicto en la base de datos", details = ex.InnerException?.Message });
}
catch (FileNotFoundException ex) {
    return NotFound(new { message = "Archivo no encontrado", details = ex.Message });
}
catch (Exception ex) {
    return StatusCode(StatusCodes.Status500InternalServerError, new { message = "Error inesperado", details = ex.Message });
}
```

## 2. Excepciones Personalizadas
- Crea excepciones específicas para cada tipo de error de negocio o técnico.
- Permite devolver códigos HTTP precisos y mensajes claros.
- Ejemplo:

```csharp
public class TaskOperationException : Exception {
    public HttpStatusCode StatusCode { get; }
    public TaskOperationException(string message, HttpStatusCode statusCode) : base(message) => StatusCode = statusCode;
}
public class NotFoundException : TaskOperationException {
    public NotFoundException(string message) : base(message, HttpStatusCode.NotFound) { }
}
public class BadRequestException : TaskOperationException {
    public BadRequestException(string message) : base(message, HttpStatusCode.BadRequest) { }
}
```

## 3. Middleware Global de Manejo de Errores
- Centraliza el manejo de excepciones y evita duplicar lógica en los controladores.
- Devuelve respuestas JSON estandarizadas y códigos HTTP correctos.
- Ejemplo:

```csharp
app.UseExceptionHandler(errorApp => {
    errorApp.Run(async context => {
        var exceptionHandler = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionHandler?.Error;
        context.Response.StatusCode = exception switch {
            NotFoundException => StatusCodes.Status404NotFound,
            BadRequestException => StatusCodes.Status400BadRequest,
            _ => StatusCodes.Status500InternalServerError
        };
        await context.Response.WriteAsync(exception?.Message ?? "Error interno");
    });
});
```

## 4. Validación Anticipada y Separación de Errores
- Valida los modelos y parámetros antes de ejecutar la lógica de negocio.
- Separa errores de dominio (validaciones) de errores técnicos (fallos de disco, OpenCV, etc).
- Ejemplo:

```csharp
if (task.s3pathIn?.accessKey == null)
    throw new BadRequestException("S3 accessKey es requerido");
```

## 5. Respuestas de Error Estandarizadas
- Utiliza una estructura JSON común para los errores:

```csharp
public class ApiErrorResponse {
    public int StatusCode { get; set; }
    public string Message { get; set; }
    public string? Details { get; set; }
}
// Uso:
return BadRequest(new ApiErrorResponse {
    StatusCode = 400,
    Message = "Parámetros inválidos",
    Details = ex.Message
});
```

## 6. Logging Detallado y Trazabilidad
- Registra los errores críticos con identificadores (`traceId`, `taskId`) para facilitar el diagnóstico.
- Ejemplo:

```csharp
_logger.LogError(ex, "Error cancelando tarea {TaskId}", id);
```

## 7. Pruebas Unitarias y Documentación
- Verifica que los controladores devuelvan los códigos HTTP correctos en los tests.
- Documenta los posibles códigos de respuesta en Swagger:

```csharp
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public ActionResult Post(...)
```

---

## Resumen de Beneficios
| Solución                  | Beneficios                                   | Complejidad  |
|---------------------------|----------------------------------------------|--------------|
| Excepciones Personalizadas| Precisión en códigos HTTP, fácil extensión   | Media        |
| Middleware Global         | Centralización, reduce código duplicado      | Alta         |
| Validación Mejorada       | Previene errores comunes                     | Baja         |
| Respuestas Estandarizadas | Consistencia en el formato de errores        | Baja         |
| Refactorización de Métodos| Elimina respuestas ambiguas (-1)             | Baja         |

---

> Estas opciones hacen que la API sea más robusta, predecible y fácil de mantener, facilitando la integración con clientes y la trazabilidad de errores.
