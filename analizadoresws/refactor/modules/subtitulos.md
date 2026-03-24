## 1. Análisis del estado actual

### Datos que consume `TaskVideoma_Subtitulos`


| Dato                 | Origen                                                                         |
| -------------------- | ------------------------------------------------------------------------------ |
| `pathIn`             | `peticionPendiente->localizacion`                                              |
| `motor`              | `listaDatosServicio[i]->mMotor` (XML)                                          |
| `thresold2_language` | `ObtenerParametroAnalizador("OCR SD", "thresold2")` — solo para motor `DVBSUB` |


### Dato de salida

- Ruta al fichero `.srt` generado (`mServicioSubtitulosTh->resultadoAnalisis`) → lo recibe `AnalizadoresBase` y lo pasa a `FinalizarExtraccionAnalizadores2()`

### Flujo interno por motor

```
inicio()
  ├─ motor PROJECTX (+ opcional TLX)  →  inicioProjectX()
  │     └─ java -jar ProjectX.jar -ini parametros.ini -demux pathIn -out pathOut
  ├─ motor ISDBT                       →  inicioISDBText()
  │     └─ ccextractorwin.exe -autoprogram -out=srt -bom -latin1 pathIn
  ├─ motor DVBSUB                      →  inicioDVBSUB()
  │     ├─ ffprobe.exe  →  detecta stream de subtítulos y duración
  │     └─ powershell dvb2str.ps1 pathIn pathOut -l thresold2_language
  └─ motor DVBText (por defecto)       →  inicioDVBText()
        └─ subTTX.exe 888 pathIn pathOut
```

El `.srt` resultante se genera en el mismo directorio que el fichero de entrada, con el sufijo `[888].srt`.

---

## 2. Arquitectura propuesta: Plugin en AIHub (contenedor Linux) — parcialmente dependiente de Windows

Un único endpoint que recibe el `motor` en el request y decide internamente qué herramienta lanzar.

La mayoría de motores funcionan en Linux:
- **PROJECTX**: Java multiplataforma (`ProjectX.jar`)
- **ISDBT**: `ccextractor` disponible en Linux (`apt install -y ccextractor`)
- **DVBSUB**: `ffprobe` (Linux) + `dvb2str.ps1` (ejecutable con PowerShell Core en Linux: `apt install -y powershell`)

El motor **DVBText** depende de `subTTX.exe`, cuyo origen es **desconocido** (¿?). Si no se encuentra alternativa Linux, este motor concreto deberá redirigir a un servicio REST en una máquina Windows.

---

## 3. Contrato de la API del nuevo módulo

### `POST /subtitulos/extraer`

**Request body (JSON):**

```json
{
  "pathIn": "/var/media/fichero.ts",
  "motor": "PROJECTX | PROJECTX TLX | ISDBT | DVBSUB | DVBTEXT",
  "thresold2Language": "0.7"
}
```

> `pathIn`: ruta dentro del contenedor. Ej: `\\10.1.1.62\videos\samples\algo.ts` → `/var/media/algo.ts`
>
> `thresold2Language`: solo requerido cuando `motor = DVBSUB`. Para el resto puede omitirse o enviarse vacío.

**Response body (JSON) — éxito:**

```json
{
  "status": "ok",
  "resultado": "/var/media/fichero[888].srt"
}
```

> `resultado` es la ruta al `.srt` generado, en el mismo directorio que `pathIn`. `analizadoresws` la pasa a `FinalizarExtraccionAnalizadores2` igual que ahora.

**Response body (JSON) — error:**

```json
{
  "status": "error",
  "mensaje": "descripción del error"
}
```

---

## 4. Estructura del nuevo módulo

AIHub expone `POST /subtitulos/extraer` y, al recibirlo, ejecuta directamente la herramienta correspondiente al motor con los parámetros necesarios. Cada motor se invoca como ejecutable/comando del sistema. Para DVBText (subTTX.exe, no portable), AIHub hace POST a un servicio REST en una máquina Windows.

```
AIHub/
└── modules/
    └── subtitulos/
        ├── ProjectX/
        │   ├── ProjectX.jar              ← multiplataforma (Java), se ejecuta con `java -jar`
        │   └── parametros.ini
        └── DVBSub/
            └── dvb2str.ps1               ← se ejecuta con `pwsh` (PowerShell Core en Linux)
```

Dependencias de sistema en AIHub (se ejecutan directamente como comandos):
- `ffprobe` → `apt install -y ffmpeg` (incluye ffprobe)
- `ccextractor` → `apt install -y ccextractor`
- `java` → `apt install -y default-jre` (para `java -jar ProjectX.jar`)
- `pwsh` → `apt install -y powershell` (para `pwsh dvb2str.ps1`)
- `subTTX.exe` → **origen desconocido (¿?)**. Motor DVBText no integrable en Linux → AIHub redirige a REST Windows.

Los ejecutables **no tienen dependencia** de `AnalizadoresBase`, `WSDLServiciosWebVideoma` ni de ningún otro componente de `analizadoresws`.

---

## 5. Cambios en `analizadoresws`

### En `TaskVideoma_Subtitulos` (dentro de `AnalizadoresBase.cpp`)

1. Construir el JSON de request con `pathIn`, `motor` y (si es DVBSUB) `thresold2Language`
2. Hacer una llamada HTTP `POST /subtitulos/extraer` al nuevo servicio
3. Recibir la respuesta y extraer `resultado` (ruta al SRT)
4. Devolver el string tal y como se hace ahora → el resto del flujo no cambia

---

## 6. Integración en AIHub

- Montar el directorio de medios como volumen en AIHub: `-v /ruta/medios:/var/media`
- El `.srt` se genera en el mismo volumen montado, por lo que es accesible desde el Factory sin volumen adicional.
- Las rutas en el JSON deben ser las rutas **dentro del contenedor AIHub**.

### Compatibilidad por motor

| Motor | Herramienta | Disponible en Linux | Acción |
|-------|-------------|---------------------|--------|
| PROJECTX | `ProjectX.jar` | Si (Java multiplataforma) | `apt install -y default-jre` en AIHub |
| ISDBT | `ccextractor` | Si | `apt install -y ccextractor` en AIHub |
| DVBSUB | `ffprobe` + `dvb2str.ps1` | Si | `apt install -y ffmpeg powershell` en AIHub |
| **DVBText** | `subTTX.exe` | **No — origen desconocido (¿?)** | **Redirigir a servicio REST en máquina Windows** |

> Para el motor DVBText, al no existir el código fuente de `subTTX.exe`, el plugin AIHub deberá redirigir la petición a un servicio REST alojado en una máquina Windows con `subTTX.exe` instalado. El resto de motores funcionan nativamente en el contenedor Linux de AIHub.

