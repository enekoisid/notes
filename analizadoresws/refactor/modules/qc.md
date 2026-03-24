## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_QC`

| Dato | Origen |
|---|---|
| `pathIn` | `peticionPendiente->localizacion` |
| `listQC` | `peticionPendiente->idioma` — lista de análisis a ejecutar (vacío = todos) |
| `blackdetect_duration` | `ObtenerParametroAnalizador("QUALITY CONTROL", "blackdetect_duration")` |
| `blackdetect_threshold` | `ObtenerParametroAnalizador("QUALITY CONTROL", "blackdetect_threshold")` |
| `silencedetect_duration` | `ObtenerParametroAnalizador("QUALITY CONTROL", "silencedetect_duration")` |
| `silencedetect_threshold` | `ObtenerParametroAnalizador("QUALITY CONTROL", "silencedetect_threshold")` |
| `freezedetect_duration` | `ObtenerParametroAnalizador("QUALITY CONTROL", "freezedetect_duration")` |
| `freezedetect_threshold` | `ObtenerParametroAnalizador("QUALITY CONTROL", "freezedetect_threshold")` |
| `loudness_integrated` | `ObtenerParametroAnalizador("QUALITY CONTROL", "loudness_input_integrated")` |
| `loudness_true_peak` | `ObtenerParametroAnalizador("QUALITY CONTROL", "loudness_true_peak")` |
| `loudness_lra` | `ObtenerParametroAnalizador("QUALITY CONTROL", "loudness_lra")` |
| `blackframe_amount` | `ObtenerParametroAnalizador("QUALITY CONTROL", "blackframe_amount")` |
| `blackframe_threshold` | `ObtenerParametroAnalizador("QUALITY CONTROL", "blackframe_threshold")` |
| `scenedetect_threshold` | `ObtenerParametroAnalizador("QUALITY CONTROL", "scenedetect_threshold")` |

### Dato de salida

- JSON con todos los resultados de QC → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

```json
{
  "blackdetect": [...],
  "freezedetect": [...],
  "blackframe": [...],
  "scenedetect": [...],
  "silencedetect": [...],
  "loudness": {}
}
```

### Flujo interno

```
AnalizarQCMix()
  └─ construye un único comando ffmpeg con filter_complex combinando
     solo los filtros cuyo umbral/duración ≠ "0" y que estén en listQC:
       blackdetect, freezedetect, blackframe, scenedetect,
       silencedetect, loudnorm
  └─ ffmpeg.exe -i pathIn -filter_complex "..." -f null -
       └─ parsea stdout/stderr → rellena jsonString
            ParseBlackDetect / ParseFreeze / ParseBlackFrame /
            ParseScene / ParseSilence / ParseLoudness
```

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra directamente como plugin dentro del contenedor AIHub. Su única dependencia externa es `ffmpeg`, que es nativo en Linux (`apt install -y ffmpeg`). No requiere ejecutables Windows ni estudio de compilación. Es uno de los módulos más sencillos de integrar en AIHub.

---

## 3. Contrato de la API del nuevo módulo

### `POST /qc/analizar`

**Request body (JSON):**
```json
{
  "pathIn": "/var/media/fichero.mp4",
  "listQC": "blackdetect,silencedetect,loudness",
  "blackdetect_duration": "2",
  "blackdetect_threshold": "0.1",
  "silencedetect_duration": "1",
  "silencedetect_threshold": "-50",
  "freezedetect_duration": "5",
  "freezedetect_threshold": "0.5",
  "loudness_integrated": "23",
  "loudness_true_peak": "-2",
  "loudness_lra": "7",
  "blackframe_amount": "98",
  "blackframe_threshold": "32",
  "scenedetect_threshold": "0.4"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `listQC`: lista separada por comas de los análisis a ejecutar. Si está vacío, se ejecutan todos. Los valores posibles son: `blackdetect`, `freezedetect`, `blackframe`, `scenedetect`, `silencedetect`, `loudness`.
>
> Cualquier parámetro con valor `"0"` desactiva ese análisis concreto.

**Response body (JSON) — éxito:**
```json
{
  "status": "ok",
  "resultado": "{\"blackdetect\":[...],\"freezedetect\":[...],\"blackframe\":[...],\"scenedetect\":[...],\"silencedetect\":[...],\"loudness\":{}}"
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

AIHub expone `POST /qc/analizar` y, al recibirlo, ejecuta un binario local pasándole los parámetros necesarios. El binario analiza el fichero con `ffmpeg`, escribe el JSON de resultado en un fichero compartido, y devuelve un exit code por stdout (0 = OK, >0 = error). AIHub comprueba el exit code, lee el JSON del fichero compartido, y lo retorna al llamante.

```
AIHub/
└── modules/
    └── qc/
        └── qc_analyzer             ← ejecutable Linux que recibe parámetros, invoca ffmpeg, y escribe resultado en JSON compartido (exit code por stdout)
```

Dependencia de sistema: `ffmpeg` (instalado en AIHub vía `apt install -y ffmpeg`).

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_QC` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con `pathIn`, `listQC` y todos los parámetros de umbral/duración
2. Hacer una llamada HTTP `POST /qc/analizar` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- `ffmpeg` se instala como paquete del sistema en el contenedor AIHub (`apt install -y ffmpeg`). No requiere ejecutables Windows.
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- El Factory llama al endpoint `POST /qc/analizar` de AIHub, que internamente ejecuta el binario `qc_analyzer` con los parámetros.
- El ejecutable invoca `ffmpeg` del sistema, parsea la salida, y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout.
- **No requiere estudio de compilación ni servicio Windows REST.** Es uno de los módulos más directos de integrar.