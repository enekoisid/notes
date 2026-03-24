## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_FaceRecognition`

**Motor NEURO (`nombre` contiene "NEURO"):**


| Dato                       | Origen                                                                 |
| -------------------------- | ---------------------------------------------------------------------- |
| `pathIn`                   | `peticionPendiente->localizacion`                                      |
| `analizadorMSRoute`        | `listaDatosServicio[i]->mUrlMotor` (XML)                               |
| `usaSaas`                  | `listaDatosServicio[i]->mUsaSaas` (XML)                                |
| `proxy`                    | `proxyurl`, `proxyuser`, `proxypwd`                                    |
| `action`                   | `ObtenerParametroAnalizador("FACE RECOGNITION NEURO", "ACTION")`       |
| `database`                 | `ObtenerParametroAnalizador("FACE RECOGNITION NEURO", "DATABASE")`     |
| `threshold1`, `threshold2` | `ObtenerParametroAnalizador("FACE RECOGNITION NEURO", "THRESHOLD1/2")` |
| `distance`, `fps`          | `ObtenerParametroAnalizador("FACE RECOGNITION NEURO", "DISTANCE/FPS")` |


**Motor ONNX (`nombre` contiene "ONNX"):**


| Dato                | Origen                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `pathIn`            | `peticionPendiente->localizacion`                                                           |
| `analizadorMSRoute` | `listaDatosServicio[i]->mUrlMotor` (XML)                                                    |
| `proxy`             | `proxyurl`, `proxyuser`, `proxypwd`                                                         |
| `threshold1`        | `ObtenerParametroAnalizador("FACE RECOGNITION", "THRESHOLD1")` — se usa como `targetHeight` |
| `fps`               | `ObtenerParametroAnalizador("FACE RECOGNITION", "FPS")` — se usa como `frames2Skip`         |
| `distance`          | `ObtenerParametroAnalizador("FACE RECOGNITION", "DISTANCE")`                                |


**Motor DLIB (por defecto):**


| Dato                                                                | Origen                                                |
| ------------------------------------------------------------------- | ----------------------------------------------------- |
| `pathIn`                                                            | `peticionPendiente->localizacion`                     |
| `analizadorMSRoute`                                                 | `listaDatosServicio[i]->mUrlMotor` (XML)              |
| `usaSaas`                                                           | `listaDatosServicio[i]->mUsaSaas` (XML)               |
| `proxy`                                                             | `proxyurl`, `proxyuser`, `proxypwd`                   |
| `action`, `database`, `threshold1`, `threshold2`, `distance`, `fps` | `ObtenerParametroAnalizador("FACE RECOGNITION", ...)` |


### Dato de salida

- JSON/XML con los resultados de reconocimiento facial → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno por motor

```
dispatcher() — NEURO / DLIB (usaSaas=1)
  ├─ POST {analizadorMSRoute} con JSON de tarea → obtiene task ID
  └─ polling GET {analizadorMSRoute}/{id}  (cada 30s)
       └─ cuando estado="completed" → lee XML resultado → devuelve JSON

dispatcherOnnx() — ONNX
  └─ POST {analizadorMSRoute}/detectInVideo
       body: { videoUrl, doRace, doAge, doGender, doEmotions, doEmbeddings, targetHeight, frames2Skip }
       └─ devuelve JSON directamente (síncrono)
```

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra como plugin en AIHub. Toda la lógica de reconocimiento facial reside en servicios externos (NEURO, DLIB, ONNX) accesibles por red. El plugin actúa como intermediario: gestiona el polling (NEURO/DLIB) o la llamada síncrona (ONNX) y expone dos endpoints con contratos diferentes.

---

## 3. Contrato de la API del nuevo módulo

### `POST /facerecognition/analizar`

Para motores **NEURO** y **DLIB**.

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "engine": "neuro | dlib",
  "action": "recognition",
  "database": "/var/db/faces",
  "threshold1": "0.6",
  "threshold2": "0.4",
  "distance": "0.5",
  "fps": "25",
  "usaSaas": "1",
  "proxy": {
    "url": "",
    "user": "",
    "pwd": ""
  }
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `database`: ruta a la base de datos de caras, también traducida a ruta de contenedor.
>
> `analizadorMSRoute` (URL del servicio externo de reconocimiento facial) es configuración interna del módulo.
>
> El módulo gestiona el **polling** internamente y solo responde cuando el análisis está completo.

---

### `POST /facerecognition/analizar-onnx`

Para motor **ONNX**.

**Request body (JSON):**

```json
{
  "pathIn": "http://10.1.1.62/videos/fichero.mp4",
  "targetHeight": "128",
  "frames2Skip": "50",
  "distance": "0.5",
  "proxy": {
    "url": "",
    "user": "",
    "pwd": ""
  }
}
```

> `pathIn` para ONNX es una **URL HTTP** (no ruta local), ya que el servicio ONNX externo la consume directamente.
>
> `analizadorMSRoute` es configuración interna del módulo.

---

### Response body — éxito (ambos endpoints)

```json
{
  "status": "ok",
  "resultado": "{ ...JSON con resultados de reconocimiento... }"
}
```

### Response body — error

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

---

## 4. Decisión de endpoint en `analizadoresws`


| `nombre` XML       | Endpoint                                            |
| ------------------ | --------------------------------------------------- |
| Contiene `"NEURO"` | `POST /facerecognition/analizar` con `engine=neuro` |
| Contiene `"ONNX"`  | `POST /facerecognition/analizar-onnx`               |
| Resto (DLIB)       | `POST /facerecognition/analizar` con `engine=dlib`  |


---

## 5. Estructura del nuevo módulo

AIHub expone los endpoints y, al recibirlos, ejecuta un binario local que gestiona el polling (NEURO/DLIB) o la llamada síncrona (ONNX) al servicio externo, y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── facerecognition/
        └── facerec_analyzer        ← ejecutable Linux que recibe parámetros, llama a los servicios externos de reconocimiento, y escribe resultado en JSON compartido (exit code por stdout)
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

> Los servicios externos de reconocimiento facial (`analizadorMSRoute`) son accesibles por red desde AIHub. No requiere ejecutables locales adicionales.

---

## 6. Cambios en `analizadoresws`

### En `TaskVideoma_FaceRecognition` (dentro de `AnalizadoresBase.cpp`)

1. Según `nombre`, construir el JSON y llamar al endpoint correspondiente
2. El módulo gestiona el polling de NEURO/DLIB internamente → `analizadoresws` espera la respuesta
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 7. Integración en AIHub

- Para NEURO/DLIB: montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`; montar la base de datos de caras: `-v /ruta/faces:/var/db/faces`
- Para ONNX: no requiere volumen de medios — el servicio ONNX consume la URL HTTP directamente.
- Los servicios externos de reconocimiento son accesibles por red desde AIHub, sin volúmenes adicionales.
- Sin ejecutables locales en ningún motor: toda la lógica de análisis reside en los servicios externos.
- El Factory llama a los endpoints de AIHub, que internamente ejecutan el binario `facerec_analyzer` con los parámetros.
- **No requiere estudio de compilación ni servicio Windows REST.**

