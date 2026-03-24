## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_ALPR`


| Dato              | Origen                                                  |
| ----------------- | ------------------------------------------------------- |
| `pathIn`          | `peticionPendiente->localizacion`                       |
| `motor`           | `listaDatosServicio[i]->mMotor` (XML)                   |
| `country`         | `ObtenerParametroAnalizador("ALPR", "platecountry")`    |
| `thresoldPlates`  | `ObtenerParametroAnalizador("ALPR", "max_plates")`      |
| `analyzeVehicule` | `ObtenerParametroAnalizador("ALPR", "analyzeVehicule")` |


### Dato de salida

- JSON con matrículas detectadas como string → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno por motor

```
TaskVideoma_ALPR()
  ├─ motor contiene "VAXTOR"  →  AnalizarAlprExeVaxtor()
  │     └─ vaxWinAnalyzer.exe --URL=pathIn --pathOut=output/nombre.json --StateCodes=country --FpsRatio=fps
  │          └─ lee output/nombre.json → devuelve contenido JSON
  └─ resto de motores         →  AnalizarAlprExe()
        ├─ motor contiene "GPU" → flag -gpu, resto → -cpu
        └─ Isi_alprstream.exe pathIn output/nombre.json country analyzeVehicule thresold_plates -cpu/-gpu
             └─ lee output/nombre.json → devuelve contenido JSON
```

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — motor OpenALPR PORTABLE / motor Vaxtor NO PORTABLE

El motor se pasa en el request; el plugin decide internamente qué ejecutable lanzar.

- `Isi_alprstream.exe` (OpenALPR): **SÍ, portable a Linux.** Proyecto C/C++ con CMake. El core es multiplataforma. Solo hay que excluir los wrappers Windows del build.
- `vaxWinAnalyzer.exe` (Vaxtor): **NO portable a Linux.** C++ nativo con dependencias de `windows.h`, APIs Win32 y librería propietaria `libVaxWrap.lib` (sin versión Linux). Este motor se ejecuta obligatoriamente en una máquina Windows via REST.

---

## 3. Contrato de la API del nuevo módulo

### `POST /alpr/analizar`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "motor": "CPU | GPU | VAXTOR",
  "country": "EU",
  "thresoldPlates": "10",
  "analyzeVehicule": "1"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `analyzeVehicule`: `1` activa el análisis de vehículo asociado a la matrícula, `0` solo matrícula.
>
> `fps` (motor VAXTOR) es configuración interna del módulo (actualmente hardcodeado a 1).

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "{ ...JSON con matrículas detectadas... }"
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

AIHub expone `POST /alpr/analizar` y, al recibirlo:
- Si el motor es **OpenALPR**: ejecuta directamente el binario `Isi_alprstream` con los parámetros. El binario devuelve JSON.
- Si el motor es **VAXTOR**: hace POST al servicio REST en la máquina Windows con los parámetros. Espera el JSON de respuesta.

```
AIHub/
└── modules/
    └── alpr/
        ├── Isi_alprstream         ← compilado a Linux desde https://gitlab.isid.com/backend/oldschool/openalpr.git (C/C++ CMake, excluir wrappers Windows)
        └── output/                ← directorio temporal de resultados
```

> `vaxWinAnalyzer.exe` NO se incluye en AIHub — NO portable. Se accede via REST a máquina Windows.

Los ejecutables **no tienen dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_ALPR` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con `pathIn`, `motor`, `country`, `thresoldPlates` y `analyzeVehicule`
2. Hacer una llamada HTTP `POST /alpr/analizar` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- `Isi_alprstream.exe` (OpenALPR): **PORTABLE a Linux.** C/C++ con CMake, core multiplataforma. Excluir wrappers Windows del build. Se compila como binario Linux `Isi_alprstream` y AIHub lo ejecuta directamente con los parámetros.
- `vaxWinAnalyzer.exe` (Vaxtor): **NO PORTABLE a Linux.** Depende de `windows.h`, APIs Win32 y librería propietaria `libVaxWrap.lib`. El motor VAXTOR se ejecuta obligatoriamente en una máquina Windows. AIHub hace POST al servicio REST Windows con las peticiones `motor=VAXTOR` y espera el JSON de respuesta.
- Si el motor es GPU, el contenedor AIHub necesita acceso a la GPU del host (`--gpus all`).
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.

