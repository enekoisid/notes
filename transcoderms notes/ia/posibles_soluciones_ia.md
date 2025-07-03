# resultados de las investigaciones con chatgpt y deepseek

# chatgpt:

## Problema identificado:
El controlador de tareas (TaskController) devuelve HTTP 200 incluso cuando se producen errores (p. ej., violaciones de constraints en SQLite, errores al abrir archivos con OpenCV).

## Impacto:
Las aplicaciones cliente no pueden distinguir entre llamadas exitosas y fallidas correctamente, ya que el código HTTP no refleja el estado real.

## Posibles soluciones:

- Usar ActionResult< T > y devolver códigos HTTP adecuados (400, 404, 409, 500).

- Crear excepciones personalizadas y manejarlas explícitamente en los controladores.

- Implementar middleware global de manejo de errores.

- Validar anticipadamente y separar errores de negocio de errores técnicos.

- Estandarizar respuestas de error con estructura JSON (mensaje, detalle).

- Loguear errores con identificadores para trazabilidad.


## deepseek:

| Solución                 | Beneficios                                 | Complejidad |
|--------------------------|--------------------------------------------|-------------|
| Excepciones Personalizadas | Precisión en códigos HTTP, fácil extensión | Media       |
| Middleware Global        | Centralización, reduce código duplicado    | Alta        |
| Validación Mejorada      | Previene errores comunes (ej: SQLite 19)   | Baja        |
| Respuestas Estándar      | Consistencia en formato de errores         | Baja        |
| Refactor de PostCommand()| Elimina respuestas ambiguas (-1)           | Baja        |

