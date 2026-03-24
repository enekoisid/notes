## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_SimbolosText`


| Dato        | Origen                                                                     |
| ----------- | -------------------------------------------------------------------------- |
| `petid`     | `peticionPendiente->petid`                                                 |
| `pathIn`    | `peticionPendiente->localizacion`                                          |
| `idioma`    | `peticionPendiente->idioma`                                                |
| `isSymbols` | propiedad interna: `true` si `SYMBOL ANALYSIS`, `false` si `TEXT ANALYSIS` |


### Dato de salida

- `TaskVideoma_SimbolosText` devuelve `""` inmediatamente. El resultado real llega de forma **asíncrona**: un timer llama a `TratarTareasEnCurso`, que detecta el directorio de evidencias y llama a `FinalizarExtraccionAnalizadores2` con la ruta del directorio (`dirOut`).

### Flujo interno de `CSymbolsText`

```
TaskVideoma_SimbolosText()
  ├─ isSymbols=true  + idioma=="" o "DETECT"  →  DetectSymbols()
  │     ├─ copia fichero a localDir
  │     └─ POST {url}/DetectSymbols  →  devuelve UUID de tarea
  ├─ isSymbols=true  + otro idioma             →  SearchSymbols()
  │     ├─ copia fichero a localDir
  │     └─ POST {url}/SearchSymbols  →  devuelve UUID de tarea
  └─ isSymbols=false                           →  DetectText()
        ├─ copia fichero a localDir
        └─ POST {url}/DetectTexts  →  devuelve UUID de tarea

TratarTareasEnCurso() [timer]
  └─ comprueba si existe directorio de evidencias (por UUID)
       └─ si existe → copia a dirDest → FinalizarExtraccionAnalizadores2(dirOut)
```

> El servidor de Symbols/Text genera las evidencias en un directorio compartido que `analizadoresws` monitoriza por sistema de ficheros.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra como plugin en AIHub. La diferencia clave respecto a los otros analizadores es que **el polling asíncrono pasa al interior del plugin**: el plugin espera hasta que el servidor de Symbols/Text (Vicomtech) genera las evidencias y devuelve el resultado en la misma llamada HTTP. El servidor Vicomtech es accesible por red desde AIHub.

---

## 3. Contrato de la API del nuevo módulo

### `POST /simbolos/detectar`

Equivale a `DetectSymbols()` — detecta logos/símbolos sin repositorio previo.

**Request body (JSON):**

```json
{
  "petid": 12345,
  "pathIn": "/var/media/fichero.mp4"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`

---

### `POST /simbolos/buscar`

Equivale a `SearchSymbols()` — busca símbolos registrados en el repositorio.

**Request body (JSON):**

```json
{
  "petid": 12345,
  "pathIn": "/var/media/fichero.mp4"
}
```

---

### `POST /textos/detectar`

Equivale a `DetectText()`.

**Request body (JSON):**

```json
{
  "petid": 12345,
  "pathIn": "/var/media/fichero.mp4"
}
```

---

### Response body — éxito (los tres endpoints)

```json
{
  "status": "ok",
  "resultado": "/var/evidencias/uuid-tarea"
}
```

> `resultado` es la ruta **dentro del contenedor** al directorio de evidencias generado. `analizadoresws` recibe esta ruta y la pasa a `FinalizarExtraccionAnalizadores2` igual que ahora.

### Response body — error

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

---

### `POST /simbolos/registrar`

Para el enrolamiento actualmente en `ISIDWinService::InsertarHuellaSimbolo()`.

**Request body (JSON):**

```json
{
  "name": "nombre_del_simbolo",
  "type": "TYPE_LOGO",
  "pathSymbol": "/var/media/simbolo.png"
}
```

---

## 4. Decisión de endpoint en `analizadoresws`

`analizadoresws` decide qué endpoint llamar usando la misma lógica que ahora:


| Nombre XML        | `idioma`                   | Endpoint                  |
| ----------------- | -------------------------- | ------------------------- |
| `SYMBOL ANALYSIS` | `""` o contiene `"DETECT"` | `POST /simbolos/detectar` |
| `SYMBOL ANALYSIS` | otro valor                 | `POST /simbolos/buscar`   |
| `TEXT ANALYSIS`   | cualquiera                 | `POST /textos/detectar`   |


---

## 5. Estructura del nuevo módulo

AIHub expone los endpoints y, al recibirlos, ejecuta un binario local que gestiona el envío al servidor Vicomtech, el polling asíncrono interno, y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── simbolostext/
        └── simbolos_analyzer       ← ejecutable Linux que recibe parámetros, llama a la API Vicomtech (con polling interno), y escribe resultado en JSON compartido (exit code por stdout)
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

> La URL del servidor de Symbols/Text (`mUrlMotor`), `localDir` y `dirDest` son configuración interna del módulo.

---

## 6. Cambios en `analizadoresws`

### En `TaskVideoma_SimbolosText` (dentro de `AnalizadoresBase.cpp`)

1. Según nombre e idioma, construir el JSON y llamar al endpoint correspondiente
2. **Esperar la respuesta** (el módulo resuelve el polling internamente)
3. Extraer `resultado` (ruta del directorio de evidencias) de la respuesta
4. Pasar esa ruta a `FinalizarExtraccionAnalizadores2` tal y como lo hace `TratarTareasEnCurso` ahora

> `TratarTareasEnCurso` para SYMBOL/TEXT ANALYSIS deja de ser necesario en `analizadoresws`.

### En `ISIDWinService` (enrolamiento)

- `InsertarHuellaSimbolo()` pasa a llamar a `POST /simbolos/registrar`

---

## 7. Integración en AIHub

- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Montar el directorio de evidencias como volumen en AIHub: `-v /ruta/evidencias:/var/evidencias`
- El servidor de Symbols/Text (Vicomtech) es accesible por red desde AIHub, sin necesidad de volumen adicional.
- El Factory llama a los endpoints de AIHub, que internamente ejecutan el binario `simbolos_analyzer` con los parámetros.
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.
- **No requiere ejecutables Windows ni estudio de compilación.**

