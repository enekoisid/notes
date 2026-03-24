## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_OCR`

**Motor `OCR SD`:**


| Dato               | Origen                                                 |
| ------------------ | ------------------------------------------------------ |
| `pathIn`           | `peticionPendiente->localizacion`                      |
| `idioma`           | `peticionPendiente->idioma`                            |
| `threads`          | `ObtenerParametroAnalizador("OCR SD", "threads")`      |
| `fps`              | `ObtenerParametroAnalizador("OCR SD", "fps")`          |
| `puntoX`, `puntoY` | `ObtenerParametroAnalizador("OCR SD", "puntoX/Y")`     |
| `width`, `height`  | `ObtenerParametroAnalizador("OCR SD", "width/height")` |


**Motor FineReader (cualquier otro valor):**


| Dato            | Origen                             |
| --------------- | ---------------------------------- |
| `identificador` | `peticionPendiente->identificador` |


### Dato de salida

- Contenido SRT como string → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno por motor

```
TaskVideoma_OCR()
  ├─ motor "OCR SD"        →  AnalizarFicheroOcr()
  │     └─ OCRNet.exe pathIn fps ocr/output/nombre.srt threads puntoX puntoY width height idioma intMotor
  │          └─ lee ocr/output/nombre.srt  →  devuelve contenido SRT
  └─ motor FineReader      →  wsServiciosVideoma->ExtraerOCR_FineReader(identificador)
        └─ llamada directa al web service de Videoma  →  devuelve resultado
```

> **Nota importante:** El motor FineReader no ejecuta ninguna lógica local — llama directamente al web service de Videoma pasándole el `identificador`. Este caso **no requiere módulo propio** ya que `analizadoresws` puede seguir llamando a Videoma directamente. El plan de extracción aplica **solo al motor OCR SD**.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — PORTABLE (casi directo)

**Veredicto: SÍ, portable a Linux.**

El proyecto `ocrnet` está en C# .NET 6.0 — ya es multiplataforma. El único cambio necesario es sustituir el paquete NuGet `OpenCvSharp4.Windows` por `OpenCvSharp4.runtime.linux`. No hay que tocar código fuente.

---

## 3. Contrato de la API del nuevo módulo

### `POST /ocr/analizar`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "idioma": "es",
  "threads": "4",
  "fps": "25",
  "puntoX": "0",
  "puntoY": "800",
  "width": "1920",
  "height": "120",
  "motor": "OCR SD | AZ"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `motor`: `OCR SD` usa `intMotor=0`, `AZ` usa `intMotor=1` internamente en `OCRNet.exe`.

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "1\n00:00:01,000 --> 00:00:03,500\nTexto detectado...\n"
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

AIHub expone `POST /ocr/analizar` y, al recibirlo, ejecuta directamente el binario `OCRNet` con los parámetros necesarios. El binario procesa la imagen/vídeo y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── ocr/
        ├── OCRNet                 ← compilado a Linux desde https://gitlab.isid.com/backend/oldschool/ocrnet.git (cambiar OpenCvSharp4.Windows → OpenCvSharp4.runtime.linux)
        └── output/                ← directorio temporal de resultados
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_OCR` (dentro de `AnalizadoresBase.cpp`)

```
if motor == "OCR SD" o "AZ":
  1. Construir JSON con pathIn, idioma, threads, fps, puntoX, puntoY, width, height, motor
  2. POST /ocr/analizar al nuevo servicio
  3. Extraer resultado y devolverlo

else (FineReader):
  → sin cambios, sigue llamando a wsServiciosVideoma->ExtraerOCR_FineReader() directamente
```

---

## 6. Integración en AIHub

- `OCRNet.exe`: **PORTABLE a Linux** (C# .NET 6.0). Cambiar el paquete NuGet `OpenCvSharp4.Windows` por `OpenCvSharp4.runtime.linux`. Se publica como binario Linux `OCRNet` y AIHub lo ejecuta directamente con los parámetros.
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.

