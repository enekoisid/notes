## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_WhisperX`


| Dato               | Origen                                |
| ------------------ | ------------------------------------- |
| `pathIn`           | `peticionPendiente->localizacion`     |
| `localizacionhttp` | `peticionPendiente->localizacionhttp` |
| `idioma`           | `peticionPendiente->idioma`           |
| `proxy`            | `proxyurl`, `proxyuser`, `proxypwd`   |


### Dato de salida (ambos)

- String con el contenido del `.srt` → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno — WhisperX (`EnviarFilePOSTCurlWhisperX`)

```
├─ Si fichero no es mp3/wav  →  ffmpeg convierte a MP3
├─ Si fichero no existe local  →  ffmpeg descarga desde localizacionhttp
├─ Sanitiza nombre de fichero: regex [^a-zA-Z0-9-_]
├─ Parseo de idioma (igual que Whisper)
└─ PowerShell POST {urlWhisperX}/api/data
       body: { language, model, task, diarize=True, hf_token, output_format=srt }
       →  guarda SRT en fichero temp  →  lee y devuelve contenido
```

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra directamente como plugin en AIHub. Su única dependencia local es `ffmpeg` (nativo Linux). Los servidores ASR (Whisper y WhisperX) son servicios externos accesibles por red. No requiere ejecutables Windows.

---

## 3. Contrato de la API del nuevo módulo

### `POST /whisperx/transcribir`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "localizacionhttp": "http://servidor/fichero.mp4",
  "idioma": "es",
  "proxy": {
    "url": "",
    "user": "",
    "pwd": ""
  }
}
```

> Mismo contrato de entrada que `/whisper/transcribir`. La diferencia es interna: llama a `/api/data` con diarización y el servidor WhisperX tiene su propia URL de configuración.

---

### Response body — éxito (ambos endpoints)

```json
{
  "status": "ok",
  "resultado": "1\n00:00:01,000 --> 00:00:03,500\nTexto transcrito...\n"
}
```

### Response body — error (ambos endpoints)

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

---

## 4. Estructura del nuevo módulo

AIHub expone los endpoints y, al recibirlos, ejecuta un binario local pasándole los parámetros. El binario preprocesa el audio con `ffmpeg`, llama al servidor ASR correspondiente (Whisper o WhisperX) por HTTP, y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── whisper/
        └── whisper_analyzer        ← ejecutable Linux que recibe parámetros, llama a ffmpeg + servidor ASR, y escribe resultado en JSON compartido (exit code por stdout)
```

Dependencia de sistema: `ffmpeg` (instalado en AIHub vía `apt install -y ffmpeg`).

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

> Las URLs de los servidores ASR (`urlWhisper` y `urlWhisperX`) son configuración interna del módulo, no parámetros de la petición.

---

## 5. Cambios en `analizadoresws`

1. Construir el JSON de request con `pathIn`, `localizacionhttp`, `idioma` y `proxy`
2. Hacer una llamada HTTP `POST /whisperx/transcribir` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- `ffmpeg` se instala como paquete del sistema en el contenedor AIHub (`apt install -y ffmpeg`). No requiere ejecutables Windows.
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Los servidores Whisper ASR y WhisperX son accesibles por red desde AIHub, sin necesidad de volumen adicional.
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.
- El Factory llama al endpoint de AIHub, que internamente ejecuta el binario `whisper_analyzer` con los parámetros.
- **No requiere estudio de compilación ni servicio Windows REST.**

