# Descripción de la investigación sobre manejo de errores en TaskController

## Problema identificado
El controlador de tareas (`TaskController`) no maneja explícitamente el caso cuando `PostCommand` devuelve `-1`, lo que indica que el comando no es una operación de cancelación. Además, cuando no se encuentra un perfil supra, se lanza una excepción genérica. Esto lleva a respuestas ambiguas y un manejo de errores inconsistente.

## Impacto
- Los clientes pueden recibir respuestas HTTP poco claras o engañosas.
- El manejo de errores no es consistente, lo que dificulta la depuración y el mantenimiento.

## Mejores opciones identificadas

### 1. Usar `ActionResult<T>` y devolver códigos HTTP adecuados
- Devolver códigos HTTP apropiados como 400, 404, 409 o 500 según el tipo de error.
- Permite a los clientes interpretar correctamente el resultado de la operación.

### 2. Crear excepciones personalizadas y manejarlas explícitamente en los controladores
- Permite precisión en los códigos HTTP y facilita la extensión del manejo de errores.
- Mejora la trazabilidad y el control sobre los distintos tipos de errores.

### 3. Implementar middleware global de manejo de errores
- Centraliza la gestión de errores y reduce la duplicidad de código en los controladores.
- Permite estandarizar las respuestas de error y su formato.

### 4. Estandarizar respuestas de error con estructura JSON
- Definir un formato consistente para los errores (por ejemplo: mensaje, detalle).
- Facilita el consumo y la interpretación de errores por parte de los clientes.

## Recomendación
Se recomienda combinar las opciones anteriores para lograr un manejo de errores robusto, consistente y fácilmente mantenible. En particular, se sugiere:
- Implementar un middleware global para capturar y gestionar errores.
- Utilizar excepciones personalizadas para los casos de negocio relevantes.
- Estandarizar las respuestas de error en formato JSON.
- Devolver siempre códigos HTTP adecuados según el resultado de la operación.
- Manejar explícitamente el valor de retorno `-1` de `PostCommand` para asegurar el flujo correcto para comandos que no son de cancelación.
- Elegir una estrategia de manejo de errores consistente:
    - Usar excepciones personalizadas y middleware para un manejo de errores centralizado y un formato de respuesta estandarizado.
    - Devolver códigos de estado HTTP directamente para simplicidad en casos sencillos.
