---

# Plan de acción: Extracción de `TaskVideoma_Sumarize` a módulo independiente

## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_Sumarize`

| Dato | Origen |
|---|---|
| `idioma` | Parseado de `peticionPendiente->idioma` (formato `[TipoIn] idioma` o `[TipoIn] idiomaIn_TO_idiomaOut`) |
| `analisis[]` | `wsServiciosVideoma_th->AnalisisMediaSumarizacion(tipoIn, tipo, identificador, idioma)` — segmentos de texto con prompt a sumarizar |
| `engine` | `listaDatosServicio[0]->mMotor` (XML) |
| `urlBase` | `ObtenerParametroAnalizador("SUMARIZACION", "baseUrl")` |

### Dato de salida

- Resultado de la sumarización como string → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno

```
TaskVideoma_Sumarize()
  ├─ parsea idioma: "[TipoIn] idioma" → extrae idiomaIn (resuelve _TO_ si existe)
  ├─ obtiene texto: wsServiciosVideoma->AnalisisMediaSumarizacion()  →  analisis[]
  │     cada elemento: { texto, prompt }
  └─ GetSumarize()
       ├─ concatena texto de todos los elementos → temp/{petID}.txt
       ├─ extrae prompt del primer elemento
       └─ engine "llama"  →  SumarizeLlama()
             └─ curl POST {urlBase}/process-multipart
                  -F file=@temp/{petID}.txt
                  -F summary_lang=idioma
                  -F prompt=prompt
                  →  devuelve resultado sumarización
```

> **Nota:** Actualmente solo existe el motor `llama`. El servidor Llama es un servicio HTTP externo al que `analizadoresws` llama vía curl. **El módulo nuevo actúa de intermediario** entre `analizadoresws` y el servidor Llama, añadiendo el preprocesado del texto.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux)

Este módulo se integra directamente como plugin en AIHub. Solo realiza llamadas HTTP al servidor Llama externo. No requiere ejecutables locales ni dependencias Windows.

---

## 3. Contrato de la API del nuevo módulo

### `POST /sumarize/sumarizar`

**Request body (JSON):**
```json
{
  "idioma": "es",
  "analisis": [
    { "texto": "Segmento de texto uno...", "prompt": "Haz un resumen en el mismo idioma." },
    { "texto": "Segmento de texto dos...", "prompt": "" }
  ]
}
```

> `analisis`: array de segmentos generados por `analizadoresws` a partir de `wsServiciosVideoma->AnalisisMediaSumarizacion()` antes de llamar al endpoint. Solo se usa el `prompt` del primer elemento.
>
> `engine` y `urlBase` (URL del servidor Llama) son configuración interna del módulo.

**Response body (JSON) — éxito:**
```json
{
  "status": "ok",
  "resultado": "{ ...resultado de la sumarización... }"
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

AIHub expone `POST /sumarize/sumarizar` y, al recibirlo, ejecuta un binario local que preprocesa el texto, llama al servidor Llama por HTTP, y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── sumarize/
        └── sumarize_analyzer       ← ejecutable Linux que recibe parámetros, llama al servidor Llama, y escribe resultado en JSON compartido (exit code por stdout)
```

El ejecutable **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

> El servidor Llama (`urlBase`) es accesible por red desde AIHub. No requiere volúmenes ni ejecutables locales.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_Sumarize` (dentro de `AnalizadoresBase.cpp`)

1. Parsear `peticionPendiente->idioma` para extraer `idioma` (igual que ahora)
2. Llamar a `wsServiciosVideoma->AnalisisMediaSumarizacion()` para obtener `analisis[]` (igual que ahora)
3. Serializar `analisis[]` al formato del request JSON
4. Hacer una llamada HTTP `POST /sumarize/sumarizar` al nuevo servicio
5. Recibir la respuesta y extraer `resultado`
6. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- No requiere volumen de medios — el texto a sumarizar se envía directamente en el body del request.
- El servidor Llama es accesible por red desde AIHub, sin necesidad de volumen adicional.
- El Factory llama al endpoint `POST /sumarize/sumarizar` de AIHub, que internamente ejecuta el binario `sumarize_analyzer` con los parámetros.
- **No requiere estudio de compilación ni servicio Windows REST.**