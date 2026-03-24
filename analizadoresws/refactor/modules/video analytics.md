## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_VideoAnalytics`


| Dato         | Origen                                                               |
| ------------ | -------------------------------------------------------------------- |
| `pathIn`     | `peticionPendiente->localizacion`                                    |
| `motor`      | `listaDatosServicio[i]->mMotor` (XML)                                |
| `classes`    | `peticionPendiente->idioma` — filtro de clases de objetos a detectar |
| `confidence` | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "CONFIDENCE")`        |
| `model`      | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "MODEL")`             |
| `period`     | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "PERIOD")`            |
| `distance`   | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "DISTANCE")`          |
| `proxyPath`  | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "PROXY_PATH")`        |
| `proxyUrl`   | `ObtenerParametroAnalizador("VIDEO ANALYTICS", "PROXY_URL")`         |


### Dato de salida

- JSON con los objetos detectados → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno

```
DetectObjects()
  ├─ motor YOLO  → engine="yolo"
  ├─ motor TF    → engine="tf"
  ├─ motor ONNX  → engine="onnx"
  └─ VideoAnalytics6.exe
       --fileIn pathIn
       --jsonOut obdt/output/nombre.json
       --engine yolo|tf|onnx
       --confidence confidence
       --period period
       [--fileOut proxyPath/nombre_va.mp4 --http proxyUrl/nombre_va.mp4]
       [--filter classes]
       └─ lee obdt/output/nombre.json → devuelve contenido JSON
```

> `proxyPath` y `proxyUrl` son opcionales: si están informados, el ejecutable también genera el vídeo procesado y lo sube a una URL.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — PORTABLE (con esfuerzo)

**Veredicto: SÍ, portable a Linux con cambios.**

El proyecto `videoanalytics6` está en C# .NET 6.0. Los cambios necesarios:
- Cambiar el paquete NuGet `OpenCvSharp4.runtime.win` por `OpenCvSharp4.runtime.linux`.
- Reempaquetar las DLLs de GStreamer/FFmpeg para su equivalente Linux (librerías .so).

---

## 3. Contrato de la API del nuevo módulo

### `POST /videoanalytics/analizar`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "motor": "videoma_motor",
  "classes": "person,car",
  "confidence": "50",
  "model": "yolov8n",
  "period": "25",
  "distance": "1",
  "proxyPath": "/var/proxy/output",
  "proxyUrl": "http://servidor/videos"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `classes`: filtro de objetos a detectar separados por coma. Si está vacío, detecta todos.
>
> `proxyPath` y `proxyUrl`: opcionales. Si se informan, el módulo genera también el vídeo procesado y lo publica en esa URL.

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "{ ...JSON con objetos detectados... }"
}
```

**Response body (JSON) — error:**

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

---

## 4. Estructura del nuevo módulo

AIHub expone `POST /videoanalytics/analizar` y, al recibirlo, ejecuta directamente el binario `VideoAnalytics6` con los parámetros necesarios. El binario procesa el vídeo y devuelve el resultado JSON.

```
AIHub/
└── modules/
    └── videoanalytics/
        ├── VideoAnalytics6              ← compilado a Linux desde https://gitlab.isid.com/backend/videoanalytics6.git (cambiar OpenCvSharp4.runtime.win → linux + reempaquetar GStreamer/FFmpeg)
        └── output/                      ← directorio temporal de resultados
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_VideoAnalytics` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con todos los parámetros
2. Hacer una llamada HTTP `POST /videoanalytics/analizar` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- `VideoAnalytics6.exe`: **PORTABLE a Linux** (C# .NET 6.0). Cambiar `OpenCvSharp4.runtime.win` por `OpenCvSharp4.runtime.linux` y reempaquetar las DLLs de GStreamer/FFmpeg con sus equivalentes Linux (.so). Se publica como binario Linux `VideoAnalytics6` y AIHub lo ejecuta directamente con los parámetros.
- Si el motor es GPU, el contenedor AIHub necesita acceso a la GPU del host (`--gpus all`).
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Si se usa `proxyPath`, montar también ese directorio en AIHub: `-v /ruta/proxy:/var/proxy/output`
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.

