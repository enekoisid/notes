## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_Translate`


| Dato         | Origen                                                                                                                       |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `idiomaIn`   | Parseado de `peticionPendiente->idioma` (formato `[TipoIn] idiomaIn;idiomaOut`)                                              |
| `idiomaOut`  | Parseado de `peticionPendiente->idioma`                                                                                      |
| `analisis[]` | `wsServiciosVideoma_th->AnalisisMedia(tipoIn, tipo, identificador, idiomaIn)` — segmentos de texto con timestamps a traducir |
| `engine`     | `ObtenerParametroAnalizador("TRANSLATE", "engine")`                                                                          |
| `urlBase`    | `ObtenerParametroAnalizador("TRANSLATE", "url")`                                                                             |
| `useridSaas` | `ObtenerParametroAnalizador("TRANSLATE", "usuario")`                                                                         |
| `tokenSaas`  | `ObtenerParametroAnalizador("TRANSLATE", "password")`                                                                        |


### Dato de salida

- Contenido SRT traducido como string → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno

```
TaskVideoma_Translate()
  ├─ parsea idioma: "[TipoIn] idiomaIn;idiomaOut"
  ├─ obtiene texto a traducir: wsServiciosVideoma->AnalisisMedia()  →  analisis[]
  └─ GetTranslate()
       ├─ engine "AZ"  →  AzureTranslator()
       │     ├─ FicheroJson(): serializa analisis[] → temp/jsonIn.json
       │     └─ azureTranslator.exe jsonIn.json srtOut.srt url pwd languageOut
       │          └─ lee srtOut.srt → devuelve SRT traducido
       └─ engine "ST"  →  SystranTranslator()
             ├─ FicheroSrt(): serializa analisis[] → temp/srtIn.srt
             └─ Systran.exe subscriptionKey postUrl getUrl srtIn.srt srtOut.srt source target idioma videoma
                  └─ lee srtOut.srt → devuelve SRT traducido
```

> **Nota clave:** `analizadoresws` obtiene el array `analisis[]` de Videoma antes de llamar al módulo y lo serializa en el request. El módulo **no necesita contactar con Videoma**.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — PORTABLE tras migración

**Veredicto: SÍ, ambos ejecutables son portables a Linux.**

- `azureTranslator.exe`: C# .NET Framework 4.6.1. Migrar a .NET 6+, quitar las dependencias `Colorful.Console` y `System.Drawing` (solo son decoración de consola). El código de traducción es 100% portable.
- `Systran.exe`: C# .NET Framework 4.8. Migrar el `.csproj` a .NET 6+. El código es 100% portable, no hay que tocar ni una línea.

---

## 3. Contrato de la API del nuevo módulo

### `POST /translate/traducir`

**Request body (JSON):**

```json
{
  "idiomaIn": "Spanish (es)",
  "idiomaOut": "English (en)",
  "analisis": [
    { "in": "0", "out": "3500", "text": "Texto original..." },
    { "in": "4000", "out": "7000", "text": "Más texto..." }
  ]
}
```

> `analisis`: array de segmentos con timestamps (ms) y texto, generados por `analizadoresws` a partir de `wsServiciosVideoma->AnalisisMedia()` antes de llamar al endpoint.
>
> `engine`, `urlBase`, `useridSaas`, `tokenSaas` son configuración interna del módulo.

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "1\n00:00:00,000 --> 00:00:03,500\nTranslated text...\n"
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

AIHub expone `POST /translate/traducir` y, al recibirlo, ejecuta directamente el binario correspondiente al motor (`azureTranslator` o `Systran`) con los parámetros necesarios. El binario realiza la traducción y escribe el resultado en un fichero JSON compartido. Devuelve exit code por stdout (0 = OK, >0 = error).

```
AIHub/
└── modules/
    └── translate/
        ├── azureTranslator          ← compilado a Linux desde https://gitlab.isid.com/backend/oldschool/azure-translator.git (migrar a .NET 6+, quitar Colorful.Console y System.Drawing)
        └── Systran                  ← compilado a Linux desde https://gitlab.isid.com/backend/oldschool/systran-translator.git (migrar .csproj a .NET 6+, sin cambios en código)
```

Los ejecutables **no tienen dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_Translate` (dentro de `AnalizadoresBase.cpp`)

1. Parsear `peticionPendiente->idioma` para extraer `idiomaIn` e `idiomaOut` (igual que ahora)
2. Llamar a `wsServiciosVideoma->AnalisisMedia()` para obtener `analisis[]` (igual que ahora)
3. Serializar `analisis[]` al formato del request JSON
4. Hacer una llamada HTTP `POST /translate/traducir` al nuevo servicio
5. Recibir la respuesta y extraer `resultado`
6. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- No requiere volumen de medios — el texto a traducir se envía directamente en el body del request.
- Las credenciales (`url`, `usuario`, `password`) se configuran en el plugin, no en el request.
- `azureTranslator.exe`: **PORTABLE a Linux** (C# .NET Framework 4.6.1 → migrar a .NET 6+). Quitar `Colorful.Console` y `System.Drawing` (solo decoración de consola). Código de traducción 100% portable.
- `Systran.exe`: **PORTABLE a Linux** (C# .NET Framework 4.8 → migrar .csproj a .NET 6+). Código 100% portable, sin tocar ni una línea.
- El Factory llama al endpoint `POST /translate/traducir` de AIHub, que internamente ejecuta el binario correspondiente al motor con los parámetros.

