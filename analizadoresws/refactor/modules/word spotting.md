## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_WordSpotting`

| Dato | Origen |
|---|---|
| `pathIn` | `peticionPendiente->localizacion` |
| `idioma` | `peticionPendiente->idioma` |
| `umbralAceptacion` | `ObtenerParametroAnalizador("WORD SPOTTING SD", "umbral_aceptacion")` |
| `keywords[]` | `wsServiciosVideoma_th->BuscarKeywords(TipoKeyword=TEXTO)` — lista de palabras a buscar |

### Dato de salida

- Contenido SRT con los resultados de búsqueda → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno

```
AnalizarFicheroWrdSPT()
  ├─ Normaliza keywords según idioma (números → texto, caracteres especiales)
  ├─ Escribe keywords normalizadas → wrdspt/pat/terms.txt
  ├─ Si idioma en-US: NXTextNormalizer.exe → normaliza terms.txt
  ├─ Si no existe .pat del fichero:
  │     NXIndex.exe -a {acoModel} pathIn → wrdspt/pat/fichero_idioma.pat
  └─ NXSearch.exe -p {proModel} -t umbral terms.txt fichero_idioma.pat → wrdspt/out.srt
       └─ lee out.srt → devuelve contenido SRT
```

**Modelos de idioma usados:**

| Idioma | Modelo acústico | Modelo de pronunciación |
|---|---|---|
| `es` | `Tele_LASpanish_010602.aco` | `Tele_LASpanish_010602_01.pro` |
| `en-US` | `Tele_NAEnglish_000607.aco` | `Tele_NAEnglish_000607_01.pro` |

> **Nota clave:** `analizadoresws` obtiene la lista de keywords de Videoma antes de llamar al módulo y la serializa en el request. El módulo **no necesita contactar con Videoma**.

---

## 2. Arquitectura propuesta: NO PORTABLE a Linux — requiere Windows REST

**Veredicto: NO, no es portable a Linux.**

Los tres ejecutables (`NXIndex.exe`, `NXSearch.exe`, `NXTextNormalizer.exe`) están en C# .NET Framework 4.5.2 y dependen de `Nexidia.Workbench.NET.dll`, una DLL propietaria de Nexidia sin versión Linux disponible. No es posible migrar a Linux. El módulo completo de Word Spotting se ejecuta obligatoriamente en una máquina Windows via REST.

---

## 3. Contrato de la API del nuevo módulo

### `POST /wordspotting/analizar`

**Request body (JSON):**
```json
{
  "pathIn": "/var/media/fichero.mp4",
  "idioma": "es",
  "umbralAceptacion": "0.5",
  "keywords": [
    "presidente",
    "economía",
    "elecciones"
  ]
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.mp4` → `/var/media/algo.mp4`
>
> `keywords`: array de strings generado por `analizadoresws` a partir de `wsServiciosVideoma->BuscarKeywords()` antes de llamar al endpoint.

**Response body (JSON) — éxito:**
```json
{
  "status": "ok",
  "resultado": "1\n00:00:05,000 --> 00:00:06,200\npresidente\n"
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

Al no ser portable a Linux, AIHub NO ejecuta ningún binario local para este módulo. En su lugar, AIHub hace POST al servicio REST en la máquina Windows con los parámetros y espera el JSON de respuesta.

```
Máquina Windows (servicio REST):
└── wordspotting/
    ├── WordSpottingService.exe        ← servicio REST que recibe peticiones de AIHub y ejecuta los .exe con los parámetros
    ├── NXIndex.exe                    ← https://gitlab.isid.com/backend/oldschool/nxindex.git
    ├── NXSearch.exe                   ← https://gitlab.isid.com/backend/oldschool/nxsearch.git
    ├── NXTextNormalizer.exe           ← https://gitlab.isid.com/backend/oldschool/nxtextnormalizer.git
    ├── Nexidia.Workbench.NET.dll      ← DLL propietaria (bloqueante para Linux)
    ├── license.lic
    ├── NAEnglish.tnr
    ├── Tele_LASpanish_010602.aco
    ├── Tele_LASpanish_010602_01.pro
    ├── Tele_NAEnglish_000607.aco
    ├── Tele_NAEnglish_000607_01.pro
    └── pat/                           ← caché de .pat generados
```

El servicio **no tiene dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_WordSpotting` (dentro de `AnalizadoresBase.cpp`)

1. Llamar a `wsServiciosVideoma->BuscarKeywords()` para obtener la lista de keywords (igual que ahora)
2. Construir el JSON de request con `pathIn`, `idioma`, `umbralAceptacion` y `keywords[]`
3. Hacer una llamada HTTP `POST /wordspotting/analizar` al nuevo servicio
4. Recibir la respuesta y extraer `resultado`
5. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- `NXIndex.exe`, `NXSearch.exe`, `NXTextNormalizer.exe`: **NO PORTABLES a Linux.** Dependen de `Nexidia.Workbench.NET.dll` (DLL propietaria sin versión Linux). Los tres se ejecutan obligatoriamente en una máquina Windows.
- AIHub recibe la petición del Factory, hace POST al servicio REST de Word Spotting en la máquina Windows con los parámetros, y espera el JSON de respuesta para devolverlo.
- Los `.pat` en caché se generan dentro del contenedor AIHub; si se quiere persistir entre reinicios, montar como volumen: `-v /ruta/pat:/app/wrdspt/pat`
- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.