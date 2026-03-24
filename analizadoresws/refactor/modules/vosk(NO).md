## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_Vosk`


| Dato     | Origen                            |
| -------- | --------------------------------- |
| `pathIn` | `peticionPendiente->localizacion` |
| `idioma` | `peticionPendiente->idioma`       |


### Dato de salida

- String con el contenido del `.srt` generado → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno de `CVosk`

```
GetTrancription()
  └─ VoskTranscriptor()
       ├─ ffmpeg.exe  →  extrae WAV mono 16kHz del fichero de entrada  →  temp/nombre.wav
       ├─ vosk_cmd.exe -i wav -o srt -l idioma -s 16000 -f srt  →  temp/nombre.srt
       └─ lee y devuelve el contenido del .srt
```

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — PORTABLE tras migración

**Veredicto: SÍ, portable a Linux.**

El proyecto `voskconsole` está en C# .NET Framework 4.8. El código y sus dependencias (Vosk, WebRtcVad) ya son multiplataforma. Solo requiere migrar el `.csproj` a .NET 6+. No hay que tocar código fuente.

---

## 3. Contrato de la API del nuevo módulo

### `POST /vosk/transcribir`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "idioma": "es"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "1\n00:00:01,000 --> 00:00:03,500\nTexto transcrito...\n"
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

AIHub expone `POST /vosk/transcribir` y, al recibirlo, ejecuta directamente los binarios `ffmpeg` (extracción WAV) y `vosk_cmd` (transcripción) con los parámetros necesarios. `vosk_cmd` devuelve el SRT resultante.

```
AIHub/
└── modules/
    └── vosk/
        ├── vosk_cmd                 ← compilado a Linux desde https://gitlab.isid.com/backend/oldschool/voskconsole.git (migrar .csproj a .NET 6+)
        ├── model-es/
        └── model-en/
```

Dependencia de sistema: `ffmpeg` (instalado en AIHub vía `apt install -y ffmpeg`).

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_Vosk` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con `pathIn` e `idioma`
2. Hacer una llamada HTTP `POST /vosk/transcribir` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo (`FinalizarExtraccionAnalizadores2`) no cambia

---

## 6. Integración en AIHub

- `ffmpeg`: se instala como paquete del sistema en AIHub (`apt install -y ffmpeg`).
- `vosk_cmd.exe`: **PORTABLE a Linux** (C# .NET Framework 4.8 → migrar a .NET 6+). El código y dependencias (Vosk, WebRtcVad) ya son multiplataforma. Solo hay que migrar el `.csproj`. Se publica como binario Linux `vosk_cmd` y AIHub lo ejecuta directamente con los parámetros.
- Los modelos de idioma se incluyen en la imagen AIHub o se montan como volumen.
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.

