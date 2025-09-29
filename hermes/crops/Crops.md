### ¿Cómo funciona?
- Si en el JSON del body se incluye un valor en `PathOut`, los crops se guardarán en esa carpeta.
- En cada mensaje de notificación se envía un identificador de crop, que corresponde al nombre de la imagen recortada generada.
- Al crear un Task a partir del RawTask generado en el `POST /task`, se busca en la base de datos el perfil cuyo `ProfileName` coincida.
- Se utiliza el primer resultado devuelto por la base de datos. Si se necesita un perfil con crops y otro sin ellos, crea perfiles distintos para cada caso.

### Ejemplo de JSON con crops activados
```json
{
    "SourceUrl": "http://172.29.48.1:8000/video-test.mp4",
    "ProfileName":"Matriculas",
	"Routes": ["http://172.29.48.1:5007/notification"],
    "PathOut" : "/workspaces/hermes/crops_out"
}
```