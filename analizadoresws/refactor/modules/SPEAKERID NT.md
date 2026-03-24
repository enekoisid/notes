## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_SpeakerId`

Obtenidos desde Videoma vía `wsServiciosVideoma_th`:


| Dato                  | Origen                                                              |
| --------------------- | ------------------------------------------------------------------- |
| `motor`               | `listaDatosServicio[i]->mMotor` (XML)                               |
| `usaSaas`             | `listaDatosServicio[i]->mUsaSaas` (XML)                             |
| `repositorio_voces`   | `ObtenerParametroAnalizador("SPEAKERID NT", "repositorio_voces")`   |
| `umbral_verificacion` | `ObtenerParametroAnalizador("SPEAKERID NT", "umbral_verificacion")` |
| `tipoTarea`           | `petExtra->accionanalisis`                                          |
| `speakerIds[]`        | `petExtra->extrainfo`                                               |
| `tIn`, `tOut`         | `petExtra->ctin`, `petExtra->ctout`                                 |
| `pathIn`              | `peticionPendiente->localizacion`                                   |
| `proxy`               | `proxyurl`, `proxyuser`, `proxypwd`                                 |


### Dato de salida

- JSON string devuelto por `CSpeakerID::analizar()` → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Operaciones internas de `CSpeakerID`

- `verificar()` — verifica si un audio pertenece a un speaker concreto
- `identificar()` — identifica el speaker entre una lista
- `diarizar()` — diarización del audio
- `enrollBvp()` — enrolamiento (invocado desde `ISIDWinService`, no desde `TaskVideoma_SpeakerId`)

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra directamente como plugin dentro del contenedor AIHub. No requiere ejecutables Windows locales — toda la lógica de análisis (verificar, identificar, diarizar) se ejecuta mediante llamadas a servicios externos o código C++ compilable en Linux. El repositorio de voces se monta como volumen en AIHub.

---

## 3. Contrato de la API del nuevo módulo

### `POST /speakerid/analizar`

**Request body (JSON):**

```json
{
  "tipoTarea": "verify | identify | diarize",
  "pathIn": "ruta absoluta al fichero de audio/vídeo", //10.1.1.62/videos/samples/algo.mp4 -> /var/dir/algo.mp4
  "tIn": "segundos inicio (string)",
  "tOut": "segundos fin (string)",
  "speakerIds": ["id1", "id2"],
  "repositorio": "ruta al repositorio de voces", //10.1.1.62/videos/samples/vp -> /var/dir/vp
  "minScore": "0.7",
  "proxy": {
    "url": "",
    "user": "",
    "pwd": ""
  }
}
```

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "{ ...JSON original que devuelve analizar()... }"
}
```

**Response body (JSON) — error:**

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

### `POST /speakerid/enroll`

Para el enrolamiento que actualmente está en `ISIDWinService::InsertarHuellaSpeaker()`:

```json
{
  "tIn": "0",
  "tOut": "30",
  "urlMedia": "ruta al fichero",
  "name": "nombre del speaker"
}
```

---

## 4. Estructura del nuevo módulo

AIHub expone los endpoints y, al recibirlos, ejecuta un binario local pasándole los parámetros. El binario realiza las llamadas al servicio de reconocimiento de voz y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── speakerid/
        └── speakerid_analyzer      ← ejecutable Linux que recibe parámetros, llama al servicio de reconocimiento, y escribe resultado en JSON compartido (exit code por stdout)
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_SpeakerId` (dentro de `AnalizadoresBase.cpp`)

Reemplazar la lógica actual por:

1. Construir el JSON de request con los datos que antes se pasaban a `CSpeakerID`
2. Hacer una llamada HTTP `POST /speakerid/analizar` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string JSON tal y como se hace ahora → el resto del flujo (`FinalizarExtraccionAnalizadores2`) no cambia

### En `ISIDWinService` (enrolamiento)

- `InsertarHuellaSpeaker()` pasa a llamar a `POST /speakerid/enroll` en lugar de `mServicio->mServicioSpeaker->enrollBvp()`

---

## 6. Integración en AIHub

- Montar el directorio de audios y repositorio de voces como volumen en AIHub: `-v /ruta/voces:/data/repo`
- Montar el directorio de medios en AIHub: `-v /ruta/medios:/var/media`
- Las rutas en el JSON de request deben ser las rutas **dentro del contenedor AIHub**.
- El Factory llama a los endpoints de AIHub (`POST /speakerid/analizar`, `POST /speakerid/enroll`), que internamente ejecutan el binario `speakerid_analyzer` con los parámetros correspondientes.
- **No requiere ejecutables Windows ni estudio de compilación.**

