# 1. Análisis del estado actual

### Datos que consume `TaskVideoma_MediaDescription`


| Dato      | Origen                                                       |
| --------- | ------------------------------------------------------------ |
| `pathIn`  | `peticionPendiente->localizacion`                            |
| `idioma`  | `peticionPendiente->idioma`                                  |
| `engine`  | `listaDatosServicio[0]->mMotor` (XML)                        |
| `urlBase` | `ObtenerParametroAnalizador("MEDIA DESCRIPTION", "baseUrl")` |


### Dato de salida

- Resultado de la descripción como string → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno

```
TaskVideoma_MediaDescription()
  └─ GetMediaDescription()
       └─ engine "llama"  →  MediaDescriptionLlama()
             └─ curl POST {urlBase}/process-media
                  -F file=@pathIn
                  -F description_lang=idioma
                  →  devuelve resultado descripción
```

> A diferencia de Sumarize, **el fichero de media se envía directamente** al servidor Llama (no se preprocesa texto). El servidor recibe el fichero y genera la descripción.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra directamente como plugin en AIHub. Solo realiza llamadas HTTP al servidor Llama externo, enviándole el fichero de media. No requiere ejecutables locales ni dependencias Windows.

---

## 3. Contrato de la API del nuevo módulo

### `POST /mediadescription/describir`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.mp4",
  "idioma": "es"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `engine` y `urlBase` (URL del servidor Llama) son configuración interna del módulo.

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "{ ...resultado de la descripción... }"
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

AIHub expone `POST /mediadescription/describir` y, al recibirlo, ejecuta un binario local que envía el fichero al servidor Llama por HTTP y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── mediadescription/
        └── mediadesc_analyzer      ← ejecutable Linux que recibe parámetros, envía fichero al servidor Llama, y escribe resultado en JSON compartido (exit code por stdout)
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

> El servidor Llama (`urlBase`) es accesible por red desde AIHub. El fichero de media se accede desde el volumen montado en el contenedor AIHub.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_MediaDescription` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con `pathIn` e `idioma`
2. Hacer una llamada HTTP `POST /mediadescription/describir` al nuevo servicio
3. Recibir la respuesta y extraer `resultado`
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- El servidor Llama es accesible por red desde AIHub, sin necesidad de volumen adicional.
- El Factory llama al endpoint `POST /mediadescription/describir` de AIHub, que internamente ejecuta el binario `mediadesc_analyzer` con los parámetros.
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.
- **No requiere estudio de compilación ni servicio Windows REST.**

