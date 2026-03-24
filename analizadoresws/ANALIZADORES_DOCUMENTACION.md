# Documentación del Sistema de Analizadores

## Índice

1. [Visión General](#1-visión-general)
2. [Arquitectura](#2-arquitectura)
3. [Configuración e Inicialización](#3-configuración-e-inicialización)
4. [Registro del Analizador (`AltaAnalizador`)](#4-registro-del-analizador-altaanalizador)
5. [Ciclo de Vida de una Tarea](#5-ciclo-de-vida-de-una-tarea)
6. [Gestión de Threads](#6-gestión-de-threads)
7. [Comunicación con Videoma (Web Service)](#7-comunicación-con-videoma-web-service)
8. [Autenticación y Seguridad](#8-autenticación-y-seguridad)
9. [Catálogo de Analizadores](#9-catálogo-de-analizadores)
    - [MUFIN / AUDIOFINGERPRINT](#91-mufin--audiofingerprint)
    - [SUBTITULOS](#92-subtitulos)
    - [TRANSCRIPCIONESspeaker](#93-transcripciones)
    - [SPEAKERID NT](#94-speakerid-nt)
    - [OCR / OCR SD](#95-ocr--ocr-sd)
    - [WORD SPOTTING SD](#96-word-spotting-sd)
    - [ALPR](#97-alpr)
    - [QUALITY CONTROL](#98-quality-control)
    - [VIDEO ANALYTICS](#99-video-analytics)
    - [FACE RECOGNITION / FACE RECOGNITION NEURO](#910-face-recognition--face-recognition-neuro)
    - [TRANSLATE](#911-translate)
    - [SUMARIZACION](#912-sumarizacion)
    - [MEDIA DESCRIPTION](#913-media-description)
    - [SYMBOL ANALYSIS / TEXT ANALYSIS](#914-symbol-analysis--text-analysis)
10. [Manejo de Errores y Excepciones](#10-manejo-de-errores-y-excepciones)
11. [Alias de Motores (TraducirNombreMotor)](#11-alias-de-motores-traducirniombremotor)
12. [Proxy y Parámetros Globales](#12-proxy-y-parámetros-globales)
13. [Migración a RabbitMQ: Viabilidad y Plan de Cambio](#13-migración-a-rabbitmq-viabilidad-y-plan-de-cambio)

---

## 1. Visión General

El sistema de analizadores es un servicio Windows (`.exe` gestionado como servicio) que actúa como puente entre la plataforma **Videoma** y los distintos motores de análisis multimedia. Su responsabilidad es:

- Registrarse en Videoma como analizador de un tipo determinado.
- Consultar periódicamente si hay tareas pendientes asignadas.
- Ejecutar cada tarea en un hilo independiente llamando al motor correspondiente.
- Devolver los resultados (o el error) a Videoma al terminar.

Cada instancia del servicio gestiona **un único tipo de analizador**, cuyo tipo se determina en el fichero de configuración XML en el momento del arranque.

---

## 2. Arquitectura

```
┌────────────────────────────────────────────────────────────────┐
│                      CAnalizadoresBase                         │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Timer (OnTratarTareasTimerThread)                       │  │
│  │  → Consulta tareas pendientes en Videoma                 │  │
│  │  → Lanza threads libres con LanzarTareaThread            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────────────┐    │
│  │  Thread 0 │  │  Thread 1 │  │  Thread N (hasta MAX)    │    │
│  │  Tarea    │  │  Tarea    │  │  Tarea                   │    │
│  └─────┬─────┘  └─────┬─────┘  └────────────┬─────────────┘    │
│        │              │                     │                  │
│  ┌─────▼──────────────▼─────────────────────▼───────────────┐  │
│  │            Motor de Análisis (TaskVideoma_XXX)           │  │
│  │  MUFIN / OCR / Transcripción / SpeakerID / ALPR / ...    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │          WSDLVideomaService (Web Service SOAP)           │  │
│  │  PeticionPendienteAnalizadores / FinalizarExtraccion...  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

La clase principal es `CAnalizadoresBase` (definida en `AnalizadoresBase.h` / `AnalizadoresBase.cpp`). Hereda de `CIsiHttp` y contiene instancias de todos los motores de análisis posibles, aunque en tiempo de ejecución solo se activa el que corresponda según la configuración.

---

## 3. Configuración e Inicialización

### 3.1 Parámetros de entrada del constructor

```cpp
CAnalizadoresBase(
    CIsiLog^ apLog,                              // Logger
    TDatosServicioWeb^ apDatosServicioWeb,        // Host, usuario, password, apiKey, orden
    TListaDatosServicio^ listaServiciosXml,       // Lista de servicios leída del XML
    WSDLServiciosWebVideoma::WSDLVideomaService^ apwsServiciosVideoma  // WS Videoma
)
```

### 3.2 Flujo de inicialización

1. **Leer número de threads** desde `listaServiciosXml[0]->mNThreads`. Si es ≤ 0 se usa `MAXTHREADNUMBER`.
2. **Obtener token JWT** llamando a `GetAuthToken()` contra el endpoint `auth.php` de Videoma usando la `apiKey` configurada. Si falla, el servicio termina con `exit(-1)`.
3. **Crear Web Service** (`CrearWS`) apuntando a la URL de Videoma con el token JWT.
4. **Leer prioridad de tarea** del fichero XML de configuración local (`configuracion.xml`).
5. **Para cada servicio en `listaDatosServicio`**:
  - Traducir alias de motor (`TraducirNombreMotor`).
  - Llamar a `AltaAnalizador` para registrarse en Videoma.
  - Si el alta falla → `exit(-1)`.
  - Llamar a `PonerErroneasTareasAnalizador` para limpiar tareas que quedaron en curso en arranques anteriores.
  - Inicializar el objeto motor específico (p.ej. `CMufin2`, `CSpeakerID`, etc.) y configurar sus parámetros desde Videoma mediante `ObtenerParametroAnalizador`.
6. **Arrancar el timer** (`mTimerTareas`) que dispara `OnTratarTareasTimerThread` de forma periódica.

### 3.3 Fichero XML de configuración local

El fichero `configuracion.xml` (en el mismo directorio del ejecutable) controla:


| Campo                  | Descripción                                               |
| ---------------------- | --------------------------------------------------------- |
| `tiempoLeerTareas`     | Intervalo en ms del timer de consulta de tareas           |
| `tiempoMaxWaitProcess` | Tiempo máximo de espera de proceso externo (ms)           |
| `prioridadTarea`       | Prioridad con la que se registra el analizador en Videoma |


### 3.4 Estructura `TDatosServicioWeb`


| Campo          | Descripción                                          |
| -------------- | ---------------------------------------------------- |
| `Host`         | Hostname o IP del servidor Videoma                   |
| `Localizacion` | Ruta virtual en el servidor (p.ej. `videoma`)        |
| `Usuario`      | Usuario de autenticación SOAP                        |
| `Password`     | Password de autenticación SOAP                       |
| `ApiKey`       | Clave API para obtener el token JWT                  |
| `Orden`        | Número de orden del analizador (identificador local) |


### 3.5 Estructura `TListaDatosServicio` (por cada servicio)


| Campo       | Descripción                                                   |
| ----------- | ------------------------------------------------------------- |
| `mNombre`   | Tipo de analizador (p.ej. `"TRANSCRIPCIONES"`, `"OCR"`, ...)  |
| `mMotor`    | Motor específico dentro del tipo (p.ej. `"VOSK"`, `"OCR SD"`) |
| `mIdioma`   | Idioma por defecto o path auxiliar (según el analizador)      |
| `mUrlMotor` | URL del motor externo (si aplica)                             |
| `mUser`     | Usuario para el motor SaaS externo                            |
| `mCODE`     | Password/token para el motor SaaS externo                     |
| `mUsaSaas`  | Flag: `"1"` si usa SaaS externo, `"0"` si es local            |
| `mNThreads` | Número máximo de threads concurrentes                         |


---

## 4. Registro del Analizador (`AltaAnalizador`)

```cpp
bool AltaAnalizador(String^ tipo, int orden, TListaDatosServicio^ listaDatosServicio, int indice)
```

Este método registra el servicio en Videoma y obtiene su `analizadorId`. El flujo es:

1. **Obtener IP y hostname** de la máquina local.
2. **Consultar en Videoma** (`ObtenerDatosAnalizador`) si ya existe un analizador de ese `tipo` para esa IP/nombre y orden:
  - Si existe y `Alta == "1"` → obtiene `AnalizadorID` y devuelve `true`.
  - Si existe y `Alta == "0"` → el servicio está dado de baja, devuelve `false`.
3. Si no existe en Videoma → **insertar** (`InsertarAnalizador`) con estado `"ONLINE"` y luego **terminar con `exit(-1)`** para que el servicio se reinicie ya con el registro hecho.

El nombre del analizador en Videoma se forma como: `{hostname}_{orden}` (p.ej. `SERVIDOR01_1`).

> **Nota importante**: Si `AltaAnalizador` devuelve `false`, el servicio llama a `exit(-1)` y se apaga. Es responsabilidad del gestor de servicios de Windows reiniciarlo.

---

## 5. Ciclo de Vida de una Tarea

### 5.1 Detección de tareas (timer)

El método `OnTratarTareasTimerThread` se ejecuta periódicamente y realiza:

1. **Control de tiempo máximo** por thread: si un hilo lleva más de `TIEMPOMAXIMOTAREA` ms, se aborta y la tarea se marca como `ERRONEA` en Videoma.
2. **Gestión de tareas asíncronas** (solo para SPMATIC/NUANCE/SYMBOL ANALYSIS/TEXT ANALYSIS): llama a `TratarTareasEnCurso` para actualizar estados de tareas con ID externo.
3. **Búsqueda de thread libre** en el pool.
4. **Consulta de tarea pendiente** a Videoma: `PeticionPendienteAnalizadores(tipoDeAnalizador, analizadorId)` con un timeout de 30 segundos.
5. Si hay tarea pendiente:
  - Registra la hora de inicio del thread.
  - Llama a `CicloMonitorizacion` para marcar la tarea como `ENCURSO` en Videoma.
  - Lanza el thread con `LanzarTareaThread`.

### 5.2 Ejecución de la tarea (thread)

El método `LanzarTareaThread` es el punto de entrada de cada hilo:

1. Crea una instancia independiente del Web Service (`CrearWSTH`) para el hilo.
2. Identifica el tipo de analizador a partir de `listaDatosServicio[0]->mNombre`.
3. Llama al método `TaskVideoma_XXX` correspondiente.
4. Al terminar, llama a `FinalizarExtraccionAnalizadores2` (o `FinalizarExtraccionAnalizadores`) con:
  - El estado final: `"FINALIZADA"` o `"ERRONEA"`.
  - El último error (si lo hay).
  - El resultado del análisis en formato JSON o ruta de fichero.

### 5.3 Estados posibles de una tarea


| Estado       | Significado                                         |
| ------------ | --------------------------------------------------- |
| `ENCURSO`    | El hilo está ejecutando el análisis                 |
| `FINALIZADA` | Análisis completado con éxito                       |
| `ERRONEA`    | El análisis falló (timeout, excepción, error motor) |


---

## 6. Gestión de Threads

- El número máximo de threads simultáneos se define en `MAXTHREADNUMBER` (constante) y puede limitarse mediante `mNThreads` del XML.
- Se mantiene un array `threads[]` con los objetos `Thread` y un array paralelo `listaAnalisisThreads[]` con la petición que está procesando cada hilo.
- Cuando un hilo termina (estado `Stopped`), su petición se libera (`petid = -1`) y se crea un nuevo objeto `Thread` listo para la siguiente tarea.
- Un **Mutex** (`mut`) protege la sección crítica del timer para evitar condiciones de carrera al buscar threads libres.
- Un segundo **Mutex** (`mutLista`) protege la `listaIdsEnCurso` usada por los analizadores con confirmación asíncrona (SPMATIC, NUANCE, SYMBOL ANALYSIS, TEXT ANALYSIS).

---

## 7. Comunicación con Videoma (Web Service)

La comunicación se realiza mediante un **Web Service SOAP** (`WSDLVideomaService`). Los métodos principales usados son:


| Método WS                          | Descripción                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------- |
| `ObtenerDatosAnalizador`           | Consulta si el analizador está registrado y obtiene su ID                             |
| `InsertarAnalizador`               | Registra un nuevo analizador en Videoma                                               |
| `PeticionPendienteAnalizadores`    | Devuelve la siguiente tarea pendiente para este analizador                            |
| `CicloMonitorizacion`              | Notifica el cambio de estado de una tarea (p.ej. → `ENCURSO`)                         |
| `FinalizarExtraccionAnalizadores`  | Finaliza una tarea (sin resultado)                                                    |
| `FinalizarExtraccionAnalizadores2` | Finaliza una tarea enviando el resultado JSON                                         |
| `PonerErroneasTareasAnalizador`    | Marca como erróneas todas las tareas que estaban `ENCURSO` al reiniciar               |
| `ListaTareasAnalizador`            | Lista las tareas en un estado dado para este analizador                               |
| `ObtenerParametroAnalizador`       | Obtiene un parámetro de configuración del analizador desde Videoma                    |
| `ObtenerExtraInfoPeticion`         | Obtiene información extra de una petición (usado por SpeakerID: accion, ids, tiempos) |
| `BuscarKeywords`                   | Obtiene las keywords configuradas (usado por Word Spotting)                           |
| `AnalisisMedia`                    | Obtiene análisis previos de un media (usado por Translate/Sumarize)                   |
| `AnalisisMediaSumarizacion`        | Variante de `AnalisisMedia` para sumarización                                         |
| `ObtenerDiccionario`               | Obtiene diccionario por idioma (usado en Transcripciones VOCAPIA)                     |


### Estructura de `PeticionExtracion`


| Campo              | Descripción                                      |
| ------------------ | ------------------------------------------------ |
| `petid`            | ID interno de la petición en Videoma             |
| `identificador`    | ID del elemento media en Videoma                 |
| `localizacion`     | Ruta local (UNC) del fichero multimedia          |
| `localizacionhttp` | URL HTTP del fichero multimedia                  |
| `idioma`           | Idioma o parámetros adicionales (varía por tipo) |
| `tipo`             | Tipo de media (video, audio, etc.)               |


---

## 8. Autenticación y Seguridad

### 8.1 Token JWT

Al arrancar, el servicio obtiene un **token JWT** mediante una llamada `curl` al endpoint REST:

```
POST https://{host}/{localizacion}/includes/apirest/utilsws/auth.php
Form: apikey={apiKey}&servicio=ANALIZADOR
```

Si la llamada HTTPS falla, reintenta por HTTP. El token obtenido se adjunta a la URL del Web Service SOAP como query parameter:

```
http://{host}/{localizacion}/includes/utilsws/servicios.php?videoma_jwt_token={token}
```

### 8.2 Autenticación SOAP

Adicionalmente, las llamadas SOAP se autentican con usuario/password mediante el objeto `Autenticacion`:

```cpp
autentificacion->usuariows = usuario;
autentificacion->passwordws = password;
wsServiciosVideoma->AutenticacionValue = autentificacion;
```

El timeout del WS principal se establece a **3.600.000 ms (1 hora)** para operaciones largas. Para las consultas de peticiones pendientes se usa un timeout reducido de **30.000 ms (30 segundos)**.

### 8.3 Certificados SSL

Para conexiones HTTPS se acepta cualquier certificado (incluidos autofirmados) mediante un callback personalizado:

```cpp
ServicePointManager::ServerCertificateValidationCallback = returnTrueCallback; // siempre true
```

---

## 9. Catálogo de Analizadores

### 9.1 MUFIN / AUDIOFINGERPRINT

**Nombre en Videoma**: `MUFIN` o `AUDIOFINGERPRINT`  
**Clase**: `CMufin2`  
**Archivo**: `Mufin2.h` / `Mufin2.cpp`

Realiza identificación de huellas de audio contra un repositorio.

#### Parámetros obtenidos de Videoma (`ObtenerParametroAnalizador`)


| Parámetro                      | Nombre Videoma   | Descripción                         |
| ------------------------------ | ---------------- | ----------------------------------- |
| `repositorio`                  | `"repositorio"`  | Ruta del repositorio de huellas     |
| `urlMufinServer`               | `"baseUrl"`      | URL del servidor MUFIN (solo MUFIN) |
| `listaDatosServicio[0]->mUser` | `"usuario_afp"`  | Usuario AFP (solo MUFIN)            |
| `listaDatosServicio[0]->mCODE` | `"password_afp"` | Password AFP (solo MUFIN)           |


#### Parámetros de proxy

Se configura via `SetProxy(proxyurl, proxyuser, proxypwd)`.

#### Motor de análisis

Llama a `InsertarHuella("0", "0", localizacion, "false", 0)`.

#### Diferencia entre motores


| Motor              | Engine             | Fuente parámetros                             |
| ------------------ | ------------------ | --------------------------------------------- |
| `MUFIN`            | `AFPEngine::MUFIN` | Parámetros de `"MUFIN"` en Videoma            |
| `AUDIOFINGERPRINT` | `AFPEngine::CSF`   | Parámetros de `"AUDIOFINGERPRINT"` en Videoma |


---

### 9.2 SUBTITULOS

**Nombre en Videoma**: `SUBTITULOS`  
**Clase**: `ServicioSubtitulos`  
**Archivo**: `ServicioSubtitulos.h`

Extrae subtítulos incrustados en el fichero multimedia.

#### Motores soportados


| Motor    | Descripción                                               |
| -------- | --------------------------------------------------------- |
| `DVBSUB` | Subtítulos DVB (requiere parámetro `thresold2` de OCR SD) |
| Otros    | Demás formatos de subtítulos soportados por FFmpeg        |


#### Llamada principal

```cpp
mServicioSubtitulosTh->inicio(peticionPendiente, motor);
```

---

### 9.3 TRANSCRIPCIONES

**Nombre en Videoma**: `TRANSCRIPCIONES`  
**Clases**: `ServicioTranscripcion`, `CVosk`, `CWhisperS2T`  
**Archivos**: `ServicioTranscripcion.h`, `vosk.h`, `WisperS2T.h`

Transcripción de audio a texto. Soporta múltiples motores.

#### Motores disponibles


| Motor en XML                 | Motor efectivo                | Clase                   | Descripción                                |
| ---------------------------- | ----------------------------- | ----------------------- | ------------------------------------------ |
| `VOCAPIA` / `TRANSCRIPCION3` | `VOCAPIA`                     | `ServicioTranscripcion` | Motor SaaS Vocapia/SpeechMatic Premium     |
| `SPMATIC` / `TRANSCRIPCION4` | `VOCAPIA` (de cara a Videoma) | `ServicioTranscripcion` | SpeechMatic, asíncrono                     |
| `NUANCE` / `TRANSCRIPCION2`  | `NUANCE`                      | `ServicioTranscripcion` | Motor SaaS Nuance                          |
| `MS` / `TRANSCRIPCION1`      | `MS`                          | `ServicioTranscripcion` | Motor Microsoft                            |
| `VOSK` / `TRANSCRIPCION5`    | `VOSK`                        | `CVosk`                 | Motor local Vosk (offline)                 |
| `WHISPER` / `WISPER`         | `WHISPER`                     | `CWhisperS2T`           | Motor Whisper (API REST)                   |
| `WHISPERX` / `WISPERX`       | `WHISPERX`                    | `CWhisperS2T`           | Motor WhisperX (API REST, con diarización) |


#### Parámetros en Videoma (VOCAPIA/SpeechMatic)


| Parámetro Videoma                                         | Descripción                  |
| --------------------------------------------------------- | ---------------------------- |
| `TRANSCRIPCIONES VOCAPIA > url_saas`                      | URL del servicio SaaS        |
| `TRANSCRIPCIONES VOCAPIA > usuario_saas`                  | Usuario SaaS                 |
| `TRANSCRIPCIONES VOCAPIA > password_saas`                 | Password SaaS                |
| `TRANSCRIPCIONES VOCAPIA > segment_boundary_sensitivityr` | Sensibilidad de segmentación |
| `TRANSCRIPCIONES VOCAPIA > new_speaker_sensitivity`       | Sensibilidad nuevo locutor   |
| `TRANSCRIPCIONES VOCAPIA > Operating Mode`                | Modo de operación            |
| `TRANSCRIPCIONES VOCAPIA > summary_content_type`          | Tipo de contenido resumen    |
| `TRANSCRIPCIONES VOCAPIA > summaryLength`                 | Longitud del resumen         |
| `TRANSCRIPCIONES VOCAPIA > summaryType`                   | Tipo de resumen              |


#### Parámetros en Videoma (NUANCE)


| Parámetro Videoma                        | Descripción         |
| ---------------------------------------- | ------------------- |
| `TRANSCRIPCIONES NUANCE > url_saas`      | URL del servicio    |
| `TRANSCRIPCIONES NUANCE > usuario_saas`  | Usuario SaaS        |
| `TRANSCRIPCIONES NUANCE > password_saas` | Password SaaS       |
| `TRANSCRIPCIONES NUANCE > Temp URL`      | URL temporal        |
| `TRANSCRIPCIONES NUANCE > Temp path`     | Path temporal local |


#### Comportamiento asíncrono (SPMATIC / NUANCE)

Estos motores devuelven un ID de tarea externo. El servicio mantiene una `listaIdsEnCurso` y en cada ciclo del timer llama a `TratarTareasEnCurso` / `TratarTareasEnCursoNuance` para consultar el estado y notificar a Videoma cuando terminan.

#### Llamada principal (VOSK)

```cpp
mVoskTh->GetTrancription(localizacion, motor, idioma, resultadoAnalisis);
```

#### Llamada principal (WHISPER / WHISPERX)

```cpp
mWhisper->EnviarFilePOSTCurlWhisper(motor, urlmotor, localizacion,
    localizacionhttp, pathTemp, idioma, pURL, pUser, pPwd, resultadoAnalisis);
```

El fichero temporal de salida se guarda en `.\temp\{nombre}.srt`. Para WHISPERX, el nombre del fichero se sanea eliminando caracteres especiales (`[^a-zA-Z0-9-_]`).

---

### 9.4 SPEAKERID NT

**Nombre en Videoma**: `SPEAKERID NT`  
**Clase**: `CSpeakerID`  
**Archivo**: `SpeakerID.h` / `SpeakerID.cpp`

Identificación, verificación y diarización de locutor mediante huellas de voz (BioVox).

#### Parámetros en Videoma


| Parámetro Videoma                    | Campo clase   | Descripción                         |
| ------------------------------------ | ------------- | ----------------------------------- |
| `SPEAKERID NT > repositorio_voces`   | `repositorio` | Path donde se almacenan los modelos |
| `SPEAKERID NT > umbral_verificacion` | `min_score`   | Score mínimo para verificación      |


#### Información extra de la petición (`ObtenerExtraInfoPeticion`)


| Campo            | Descripción                                           |
| ---------------- | ----------------------------------------------------- |
| `accionanalisis` | Tipo de tarea: `"verify"`, `"identify"` o `"diarize"` |
| `extrainfo[]`    | Array de IDs de locutores de Videoma                  |
| `ctin`           | Tiempo de inicio del segmento a analizar              |
| `ctout`          | Tiempo de fin del segmento a analizar                 |


#### Acciones disponibles


| Acción     | Método clase              | Descripción                                   |
| ---------- | ------------------------- | --------------------------------------------- |
| `verify`   | `CSpeakerID::verificar`   | Verifica si el audio corresponde a un locutor |
| `identify` | `CSpeakerID::identificar` | Identifica el locutor entre los conocidos     |
| `diarize`  | `CSpeakerID::diarizar`    | Segmenta el audio por locutor                 |


---

### 9.5 OCR / OCR SD

**Nombre en Videoma**: `OCR` (documentos) o `OCR SD` (vídeo fotograma a fotograma)  
**Clase**: `COcr`  
**Archivo**: `Ocr.h` / `Ocr.cpp`

Extracción de texto por reconocimiento óptico de caracteres.

#### Diferencia entre modos


| Motor    | Descripción                                                          |
| -------- | -------------------------------------------------------------------- |
| `OCR SD` | OCR sobre fotogramas de vídeo, lanzado localmente                    |
| `OCR`    | OCR de documentos mediante `ExtraerOCR_FineReader` del WS de Videoma |


#### Parámetros en Videoma (OCR SD)


| Parámetro Videoma    | Campo clase  | Descripción                              |
| -------------------- | ------------ | ---------------------------------------- |
| `OCR SD > threads`   | `Threads`    | Número de threads del motor              |
| `OCR SD > fps`       | `FPS`        | Fotogramas por segundo a analizar        |
| `OCR SD > puntoX`    | `point_X`    | Coordenada X del ROI                     |
| `OCR SD > puntoY`    | `point_Y`    | Coordenada Y del ROI                     |
| `OCR SD > width`     | `width`      | Ancho del ROI                            |
| `OCR SD > height`    | `height`     | Alto del ROI                             |
| `OCR SD > thresold2` | (subtítulos) | Umbral para detección de idioma (DVBSUB) |


#### Llamada principal (OCR SD)

```cpp
mServicioOcrTh->AnalizarFicheroOcr(localizacion, idioma, resultadoAnalisis);
```

#### Llamada principal (OCR documentos)

```cpp
wsServiciosVideoma_th->ExtraerOCR_FineReader(identificador); // Timeout: 2 horas
```

---

### 9.6 WORD SPOTTING SD

**Nombre en Videoma**: `WORD SPOTTING SD`  
**Clase**: `CWordSpotting`  
**Archivo**: `WordSpotting.h` / `WordSpotting.cpp`

Búsqueda de palabras clave en audio.

#### Fuente de keywords

Las keywords se obtienen de Videoma mediante `BuscarKeywords` filtrando por `TipoKeyword = "TEXTO"`.

#### Parámetros en Videoma


| Parámetro Videoma                      | Descripción                            |
| -------------------------------------- | -------------------------------------- |
| `WORD SPOTTING SD > umbral_aceptacion` | Umbral de confianza para aceptar match |


#### Llamada principal

```cpp
mServicioWordSpotting->AnalizarFicheroWrdSPT(
    listaKeywords, "WORD SPOTTING SD", idioma, umbral, localizacion, resultadoAnalisis);
```

---

### 9.7 ALPR

**Nombre en Videoma**: `ALPR`  
**Clase**: `CAlpr`  
**Archivo**: `Alpr.h` / `Alpr.cpp`

Reconocimiento automático de matrículas de vehículos.

#### Parámetros en Videoma


| Parámetro Videoma        | Campo clase       | Descripción                             |
| ------------------------ | ----------------- | --------------------------------------- |
| `ALPR > max_plates`      | `thresold_plates` | Número máximo de matrículas a detectar  |
| `ALPR > platecountry`    | `country`         | País de las matrículas (código ISO)     |
| `ALPR > analyzeVehicule` | `analyzeVehicule` | Flag: analizar también tipo de vehículo |


#### Motores soportados


| Motor en XML | Método clase            | Descripción            |
| ------------ | ----------------------- | ---------------------- |
| `VAXTOR`     | `AnalizarAlprExeVaxtor` | Motor Vaxtor           |
| Otros        | `AnalizarAlprExe`       | Motor OpenALPR u otros |


---

### 9.8 QUALITY CONTROL

**Nombre en Videoma**: `QUALITY CONTROL`  
**Clase**: `CQualityControl`  
**Archivo**: `QualityControl.h` / `QualityControl.cpp`

Control de calidad técnica del contenido multimedia mediante FFmpeg. Detecta múltiples tipos de incidencias en una sola pasada.

#### Parámetros en Videoma


| Parámetro Videoma                             | Campo clase               | Descripción                                         |
| --------------------------------------------- | ------------------------- | --------------------------------------------------- |
| `QUALITY CONTROL > blackdetect_duration`      | `blackdetect_duration`    | Duración mínima de negro para ser detectado         |
| `QUALITY CONTROL > blackdetect_threshold`     | `blackdetect_threshold`   | Umbral de luminancia para considerar negro          |
| `QUALITY CONTROL > silencedetect_duration`    | `silencedetect_duration`  | Duración mínima de silencio                         |
| `QUALITY CONTROL > silencedetect_threshold`   | `silencedetect_threshold` | Umbral de decibelios para silencio                  |
| `QUALITY CONTROL > freezedetect_duration`     | `freezedetect_duration`   | Duración mínima de imagen congelada                 |
| `QUALITY CONTROL > freezedetect_threshold`    | `freezedetect_threshold`  | Umbral de cambio de pixel para freeze               |
| `QUALITY CONTROL > loudness_input_integrated` | `loudness_integrated`     | Nivel integrado de loudness (EBU R128)              |
| `QUALITY CONTROL > loudness_true_peak`        | `loudness_true_peak`      | True peak de loudness                               |
| `QUALITY CONTROL > loudness_lra`              | `loudness_lra`            | Loudness Range (LRA)                                |
| `QUALITY CONTROL > blackframe_amount`         | `blackframe_amount`       | % de píxeles negros para considerar fotograma negro |
| `QUALITY CONTROL > blackframe_threshold`      | `blackframe_threshold`    | Umbral de pixel negro                               |
| `QUALITY CONTROL > scenedetect_threshold`     | `scenedetect_threshold`   | Umbral de cambio de escena (0-1)                    |


#### Llamada principal

```cpp
mServicioQCTh->AnalizarQCMix(localizacion, listQC, /* todos los parámetros */, resultadoAnalisis);
```

El campo `idioma` de la petición actúa como filtro: si especifica análisis concretos (p.ej. `"blackdetect,loudness"`), solo se ejecutan esos. Si está vacío se ejecutan todos.

---

### 9.9 VIDEO ANALYTICS

**Nombre en Videoma**: `VIDEO ANALYTICS`  
**Clase**: `CObjectDetection`  
**Archivo**: `ObjectDetection.h`

Detección de objetos en vídeo.

#### Parámetros en Videoma


| Parámetro Videoma              | Campo clase  | Descripción                                    |
| ------------------------------ | ------------ | ---------------------------------------------- |
| `VIDEO ANALYTICS > CONFIDENCE` | `confidence` | Umbral de confianza para aceptar detección (%) |
| `VIDEO ANALYTICS > MODEL`      | `config`     | Modelo/configuración del detector              |
| `VIDEO ANALYTICS > PERIOD`     | `period`     | Periodo de muestreo de fotogramas              |
| `VIDEO ANALYTICS > DISTANCE`   | `distance`   | Distancia mínima entre detecciones             |
| `VIDEO ANALYTICS > PROXY_PATH` | `proxyPath`  | Path del ejecutable proxy                      |
| `VIDEO ANALYTICS > PROXY_URL`  | `ProxyUrl`   | URL del servicio proxy                         |


#### Filtros de clase

El campo `idioma` de la petición se usa como filtro de clases de objetos a detectar (p.ej. `"person,car"`).

#### Llamada principal

```cpp
mObjectDetectionTh->DetectObjects(localizacion, resultadoAnalisis);
```

---

### 9.10 FACE RECOGNITION / FACE RECOGNITION NEURO

**Nombre en Videoma**: `FACE RECOGNITION` o `FACE RECOGNITION NEURO`  
**Clase**: `CFaceRecognition`  
**Archivo**: `FaceRecognition.h` / `FaceRecognition.cpp`

Reconocimiento facial en vídeo.

#### Parámetros en Videoma (FACE RECOGNITION / DLIB)


| Parámetro Videoma               | Campo clase  | Descripción                              |
| ------------------------------- | ------------ | ---------------------------------------- |
| `FACE RECOGNITION > ACTION`     | `action`     | Acción: detect, recognize, etc.          |
| `FACE RECOGNITION > DATABASE`   | `database`   | Base de datos de rostros                 |
| `FACE RECOGNITION > THRESHOLD1` | `threshold1` | Umbral de detección                      |
| `FACE RECOGNITION > THRESHOLD2` | `threshold2` | Umbral de reconocimiento                 |
| `FACE RECOGNITION > DISTANCE`   | `distance`   | Métrica de distancia facial              |
| `FACE RECOGNITION > FPS`        | `fps`        | Fotogramas por segundo a analizar        |
| `FACE RECOGNITION > FILE`       | `file`       | Fichero de configuración adicional       |
| `FACE RECOGNITION > baseUrl`    | `mUrlMotor`  | URL del motor de reconocimiento (en XML) |


#### Parámetros en Videoma (FACE RECOGNITION NEURO)

Mismos parámetros que FACE RECOGNITION pero bajo el grupo `"FACE RECOGNITION NEURO"`, y además:


| Parámetro Videoma                     | Campo clase  | Descripción               |
| ------------------------------------- | ------------ | ------------------------- |
| `FACE RECOGNITION NEURO > THRESHOLD1` | `threshold1` | Umbral 1 específico NEURO |
| `FACE RECOGNITION NEURO > THRESHOLD2` | `threshold2` | Umbral 2 específico NEURO |


#### Motores según nombre del analizador


| Nombre contiene   | Engine    | Método dispatcher                  |
| ----------------- | --------- | ---------------------------------- |
| `NEURO`           | `"neuro"` | `dispatcher(...)` con engine neuro |
| `ONNX`            | ONNX      | `dispatcherOnnx(...)`              |
| Ninguno (default) | `"dlib"`  | `dispatcher(...)` con engine dlib  |


---

### 9.11 TRANSLATE

**Nombre en Videoma**: `TRANSLATE`  
**Clase**: `CTranslate`  
**Archivo**: `Translate.h`

Traducción automática de texto/subtítulos entre idiomas.

#### Parámetros en Videoma


| Parámetro Videoma      | Campo clase  | Descripción                           |
| ---------------------- | ------------ | ------------------------------------- |
| `TRANSLATE > engine`   | `engine`     | Motor de traducción (p.ej. `SYSTRAN`) |
| `TRANSLATE > url`      | `urlBase`    | URL base del servicio                 |
| `TRANSLATE > usuario`  | `useridSaas` | Usuario/ID del servicio SaaS          |
| `TRANSLATE > password` | `tokenSaas`  | Password/token del servicio SaaS      |


#### Formato del campo `idioma` de la petición

```
[TIPO_ANALISIS_ORIGEN];IDIOMA_ORIGEN;IDIOMA_DESTINO
```

Ejemplo: `[TRANSCRIPCIONES VOCAPIA];Spanish (es-ES);English (en-US)`

El sistema extrae el análisis de origen de Videoma (`AnalisisMedia`) y lo traduce al idioma de destino. El resultado se guarda en `.\temp\srtOut.srt`.

#### Llamada principal

```cpp
mTranslateTh->GetTranslate(localizacion, engine, urlBase, getUrl,
    useridSaas, tokenSaas, idiomaIn, idiomaOut, idioma, resultadoAnalisis, analisis);
```

---

### 9.12 SUMARIZACION

**Nombre en Videoma**: `SUMARIZACION`  
**Clase**: `CSumarize`  
**Archivo**: `Sumarize.h` / `Sumarize.cpp`

Generación automática de resúmenes de texto a partir de transcripciones.

#### Parámetros en Videoma


| Parámetro Videoma        | Campo clase | Descripción                           |
| ------------------------ | ----------- | ------------------------------------- |
| `SUMARIZACION > baseUrl` | `urlBase`   | URL base del servicio de sumarización |


El motor (`engine`) se toma de `listaDatosServicio[0]->mMotor` (configurado en XML).

#### Formato del campo `idioma` de la petición

```
[TIPO_ANALISIS_ORIGEN];IDIOMA_SALIDA
```

o con traducción implícita:

```
[TIPO_ANALISIS_ORIGEN];IDIOMA_ORIGEN_TO_IDIOMA_DESTINO
```

#### Llamada principal

```cpp
mSumarizeTh->GetSumarize(analisis, urlBase, engine, identificador, idiomaIn, resultadoAnalisis);
```

---

### 9.13 MEDIA DESCRIPTION

**Nombre en Videoma**: `MEDIA DESCRIPTION`  
**Clase**: `CMediaDescription`  
**Archivo**: `MediaDescription.h` / `MediaDescription.cpp`

Generación automática de descripción textual del contenido de un vídeo.

#### Parámetros en Videoma


| Parámetro Videoma             | Campo clase | Descripción                          |
| ----------------------------- | ----------- | ------------------------------------ |
| `MEDIA DESCRIPTION > baseUrl` | `urlBase`   | URL base del servicio de descripción |


El motor (`engine`) se toma de `listaDatosServicio[0]->mMotor` (configurado en XML).

#### Llamada principal

```cpp
mMediaDescriptionTh->GetMediaDescription(
    localizacion, urlBase, engine, identificador, idioma, resultadoAnalisis);
```

---

### 9.14 SYMBOL ANALYSIS / TEXT ANALYSIS

**Nombre en Videoma**: `SYMBOL ANALYSIS` o `TEXT ANALYSIS`  
**Clase**: `CSymbolsText`  
**Archivo**: `symbols.h` / `symbols.cpp`

Detección y búsqueda de símbolos gráficos o texto en imágenes/vídeo mediante un motor externo (Docker).

#### Parámetros en Videoma


| Parámetro Videoma | Analizador      | Descripción                                      |
| ----------------- | --------------- | ------------------------------------------------ |
| `baseUrl`         | Ambos           | URL del motor (si no está en XML se lee de aquí) |
| `Path evidence`   | Ambos           | Directorio destino de evidencias                 |
| `Threshold`       | SYMBOL ANALYSIS | Umbral de confianza (0.0 - 1.0, defecto: 0.9)    |


El campo `mIdioma` del XML se usa como **directorio local parseado** por el Docker (defecto:  
`C:\vicomtech\hcrime-symbols_4.3-1\media` para SYMBOL ANALYSIS,  
`C:\vicomtech\hcrime-text_4.3-1\media` para TEXT ANALYSIS).

#### Diferencia entre modos


| Modo            | `isSymbols` | Operación disponible                           |
| --------------- | ----------- | ---------------------------------------------- |
| SYMBOL ANALYSIS | `true`      | `DetectSymbols` o `SearchSymbols` según idioma |
| TEXT ANALYSIS   | `false`     | `DetectText`                                   |


#### Lógica de selección de operación (SYMBOL ANALYSIS)

- Si `idioma == ""` o contiene `"DETECT"` → `DetectSymbols`
- En caso contrario → `SearchSymbols` (búsqueda de símbolo específico)

#### Comportamiento asíncrono

Al igual que SPMATIC/NUANCE, usa `listaIdsEnCurso` + `TratarTareasEnCurso` en el timer para gestionar tareas de larga duración y notificar a Videoma cuando terminan.

---

## 10. Manejo de Errores y Excepciones

### En el timer (`OnTratarTareasTimerThread`)


| Excepción              | Acción                                                      |
| ---------------------- | ----------------------------------------------------------- |
| `WebException`         | Intento de reconexión (`TratarExcepcion`) - reconecta el WS |
| `OutOfMemoryException` | `exit(-1)` inmediato                                        |
| `System::Exception`    | `exit(-1)` con log del step donde ocurrió                   |


### En el thread de tarea (`LanzarTareaThread`)


| Excepción                                 | Acción                                                |
| ----------------------------------------- | ----------------------------------------------------- |
| `WebException` (timeout FACE RECOGNITION) | La tarea puede continuar; se ajusta el timeout a 500s |
| `WebException` (otros)                    | Tarea marcada como `ERRONEA` en Videoma               |
| `System::Exception`                       | Tarea marcada como `ERRONEA` con mensaje de error     |
| Excepción desconocida                     | Tarea marcada como `ERRONEA`                          |


### Reconexión al Web Service

Si se produce un error de conexión, `TratarExcepcion` intenta recrear el Web Service. Si la reconexión falla, llama a `Liberar()` y `exit(-1)`.

---

## 11. Alias de Motores (`TraducirNombreMotor`)

Para mantener compatibilidad con versiones anteriores, algunos nombres de motor se traducen automáticamente al arrancar:


| Nombre original                       | Nombre efectivo |
| ------------------------------------- | --------------- |
| `AFP1` (en nombre)                    | `MUFIN`         |
| `AFP1` (en motor)                     | `MUFIN`         |
| `TRANSCRIPCION1`                      | `MS`            |
| `TRANSCRIPCION2`                      | `NUANCE`        |
| `TRANSCRIPCION3`                      | `VOCAPIA`       |
| `TRANSCRIPCION4`                      | `SPMATIC`       |
| `TRANSCRIPCION5` + `WHISPERX`         | `WHISPERX`      |
| `TRANSCRIPCION5` + `WHISPER`/`WISPER` | `WHISPER`       |
| `TRANSCRIPCION5` (resto)              | `VOSK`          |


---

## 12. Proxy y Parámetros Globales

Los parámetros de proxy se leen de Videoma al arrancar mediante `LeerParametrosAnalizador`:


| Parámetro Videoma        | Variable    | Descripción        |
| ------------------------ | ----------- | ------------------ |
| `TODOS > url_proxy`      | `proxyurl`  | URL del proxy HTTP |
| `TODOS > usuario_proxy`  | `proxyuser` | Usuario del proxy  |
| `TODOS > password_proxy` | `proxypwd`  | Password del proxy |


Si la URL del proxy no empieza por `"http://"`, se le añade automáticamente.

Los analizadores que soportan proxy reciben los valores mediante `SetProxy(proxyurl, proxyuser, proxypwd)`: MUFIN, AUDIOFINGERPRINT, SPEAKERID NT, FACE RECOGNITION y TRANSCRIPCIONES.

---

## 13. Migración a RabbitMQ: Viabilidad y Plan de Cambio

### 13.1 Contexto: el problema del polling

En la arquitectura actual, cada analizador consulta a Videoma periódicamente (cada N segundos) si hay tareas pendientes mediante `PeticionPendienteAnalizadores`. Esto implica:

- **Carga innecesaria en Videoma** cuando no hay tareas: queries a la base de datos que devuelven vacío.
- **Latencia de hasta N segundos** entre que Videoma encola una tarea y el analizador la empieza.
- **Escalado horizontal complicado**: si se arrancan varias instancias del mismo tipo de analizador, todas compiten por la misma tarea con el riesgo de colisión (Videoma lo gestiona internamente, pero implica lógica adicional en servidor).
- **Acoplamiento temporal**: el analizador necesita que Videoma esté disponible en cada ciclo del timer para seguir funcionando.

Migrar a **RabbitMQ** resuelve estos problemas: Videoma publica la tarea en una cola en el momento en que se crea, y el analizador la recibe de forma inmediata y reactiva, sin polling.

---

### 13.2 Valoración global de viabilidad


| Dimensión                          | Valoración     | Justificación                                                                  |
| ---------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| Viabilidad técnica                 | **Alta**       | El cambio es conceptualmente directo: sustituir el timer por un consumer       |
| Dificultad de implementación       | **Media-Alta** | El entorno C++/CLI añade fricción; hay que gestionar reconexión y acks         |
| Impacto en Videoma (lado servidor) | **Medio**      | Videoma necesita publicar en RabbitMQ además de (o en lugar de) escribir en BD |
| Riesgo de regresiónjwt             | **Medio**      | Los analizadores asíncronos (SPMATIC, SYMBOL ANALYSIS) tienen lógica especial  |
| Beneficio operacional              | **Alto**       | Menor latencia, mejor escalado horizontal, menor carga en BD de Videoma        |


---

### 13.3 Mapa de cambios: qué se elimina, qué se modifica, qué se añade

```
ARQUITECTURA ACTUAL                      ARQUITECTURA CON RABBITMQ
─────────────────────────────────────    ──────────────────────────────────────────
Timer (cada N segundos)               →  RabbitMQ Consumer (event-driven)
  └─ WS.PeticionPendienteAnalizadores →  RabbitMQ.BasicDeliver callback
  └─ gestión de threads libres        →  BasicQos(prefetchCount = nThreads)
  └─ control tiempo máximo por hilo   →  se mantiene (timer interno reducido)
  └─ TratarTareasEnCurso (async)      →  timer residual SOLO para async

WS.CicloMonitorizacion (ENCURSO)      →  se mantiene (WS o cola de estados)
WS.FinalizarExtraccionAnalizadores2   →  se mantiene (WS o cola de resultados)
WS.AltaAnalizador (al arrancar)       →  se mantiene (registro sigue vía WS)
WS.PonerErroneasTareasAnalizador      →  se mantiene (al arrancar)
```

---

### 13.4 Pseudocódigo de la migración

#### 13.4.1 Arranque del servicio (se mantiene casi igual)

```
// SIN CAMBIOS respecto al actual:
token = GetAuthToken()
CrearWS(usuario, password)
AltaAnalizador(tipo, orden)           // sigue usando el WS de Videoma
PonerErroneasTareasAnalizador()       // limpia tareas huérfanas al reiniciar
LeerParametrosAnalizador()            // proxy, etc.
InicializarMotor()                    // CMufin2, CSpeakerID, COcr, etc.

// NUEVO en lugar del timer principal:
RabbitMQ_Iniciar(tipo, nThreads)
```

#### 13.4.2 Inicialización del consumer RabbitMQ

```
función RabbitMQ_Iniciar(tipo, nThreads):

    // Nombre de la cola: uno por tipo de analizador
    // Videoma publica en esta cola cuando crea una tarea de ese tipo
    nombreCola = "ANALIZADOR_" + tipo   // ej: "ANALIZADOR_TRANSCRIPCIONES VOSK"

    conexion = RabbitMQ.CreateConnection(
        host     = parametro("rabbitmq_host"),    // leído de Videoma o XML
        puerto   = parametro("rabbitmq_port"),    // defecto: 5672
        usuario  = parametro("rabbitmq_user"),
        password = parametro("rabbitmq_password"),
        vhost    = parametro("rabbitmq_vhost")    // defecto: "/"
    )

    canal = conexion.CreateChannel()

    // Cola durable: sobrevive a reinicios del broker
    canal.QueueDeclare(
        queue    = nombreCola,
        durable  = true,
        exclusive = false,
        autoDelete = false
    )

    // CLAVE: limitar mensajes no confirmados = límite de concurrencia
    // Reemplaza directamente la lógica de "buscar thread libre"
    canal.BasicQos(prefetchCount = nThreads)

    // Registrar el callback que se dispara por cada mensaje
    canal.BasicConsume(
        queue    = nombreCola,
        autoAck  = false,        // confirmación manual (ver 13.4.3)
        consumer = OnMensajeRecibido
    )

    // Hilo bloqueante que mantiene viva la conexión
    mientras (servicio_activo):
        si (conexion.isOpen == false):
            esperar(5000ms)
            RabbitMQ_Iniciar(tipo, nThreads)   // reconexión
        esperar(1000ms)
```

#### 13.4.3 Callback de recepción de mensaje (reemplaza `OnTratarTareasTimerThread`)

```
función OnMensajeRecibido(deliveryTag, body):

    // El body del mensaje es el JSON equivalente a PeticionExtracion
    // Videoma serializa la tarea al publicar
    peticion = Deserializar<PeticionExtracion>(body)

    // Equivalente al CicloMonitorizacion actual
    WS.CicloMonitorizacion(analizadorId, "ANALIZANDO", peticion.petid, "ENCURSO", "")

    // Registrar tiempo de inicio (para control de tiempo máximo)
    tiempoInicio = DateTime.Now

    // Lanzar en un thread del pool (BasicQos garantiza que no hay más de
    // nThreads mensajes sin confirmar, así que el pool nunca se desborda)
    Thread.Start(función():
        intentar:
            resultado = EjecutarAnalisis(peticion)   // TaskVideoma_XXX actual
            estado    = motor.estado                  // "FINALIZADA" o "ERRONEA"
            error     = motor.msgUltimoError

            WS.FinalizarExtraccionAnalizadores2(
                peticion.petid, analizadorId, estado, error, resultado)

            // Confirmar al broker: el mensaje se descarta de la cola
            canal.BasicAck(deliveryTag)

        capturar excepción como ex:
            WS.FinalizarExtraccionAnalizadores2(
                peticion.petid, analizadorId, "ERRONEA", ex.Message, "")

            // Reject sin requeue: evita bucles infinitos en mensajes corruptos
            canal.BasicNack(deliveryTag, requeue = false)

            log.error("Tarea fallida petid=" + peticion.petid + ": " + ex.Message)
    )
```

#### 13.4.4 Control de tiempo máximo por tarea (timer residual)

El timer principal desaparece, pero se mantiene un **timer de watchdog** mucho más simple, solo para abortar tareas que superen `TIEMPOMAXIMOTAREA`:

```
Timer cada 30 segundos:
    para cada thread en ejecución:
        si (DateTime.Now - thread.tiempoInicio > TIEMPOMAXIMOTAREA):
            thread.Abort()
            WS.FinalizarExtraccionAnalizadores(
                thread.petid, analizadorId, "ERRONEA", "Tiempo máximo superado")
            canal.BasicNack(thread.deliveryTag, requeue = false)
            log.error("Timeout tarea petid=" + thread.petid)
```

#### 13.4.5 Analizadores asíncronos: SPMATIC, NUANCE, SYMBOL ANALYSIS, TEXT ANALYSIS

Estos analizadores tienen un patrón especial: envían la tarea a un motor externo y luego **sondean periódicamente** para saber si terminó. Esta lógica **no desaparece** con RabbitMQ; se mantiene en un timer secundario independiente:

```
// Al recibir el mensaje RabbitMQ:
función OnMensajeRecibido_Asincrono(deliveryTag, body):

    peticion = Deserializar<PeticionExtracion>(body)
    WS.CicloMonitorizacion(analizadorId, "ANALIZANDO", peticion.petid, "ENCURSO", "")

    // Enviar al motor externo (p.ej. SpeechMatic)
    idExterno = motor.EnviarTarea(peticion)

    si (idExterno < 0):
        WS.FinalizarExtraccionAnalizadores(..., "ERRONEA", ...)
        canal.BasicNack(deliveryTag, requeue = false)
        retornar

    // Guardar en lista de seguimiento (igual que ahora)
    listaIdsEnCurso.Add({ idExterno, peticion.petid, deliveryTag })
    // El deliveryTag se guarda para hacer el BasicAck cuando termine


// Timer secundario (sigue existiendo, igual que TratarTareasEnCurso):
Timer cada N segundos:
    para cada entrada en listaIdsEnCurso:
        estadoExterno = motor.ConsultarEstado(entrada.idExterno)
        si (estadoExterno == TERMINADO):
            resultado = motor.ObtenerResultado(entrada.idExterno)
            WS.FinalizarExtraccionAnalizadores2(
                entrada.petid, analizadorId, "FINALIZADA", "", resultado)
            canal.BasicAck(entrada.deliveryTag)   // ← único cambio vs. actual
            listaIdsEnCurso.Remove(entrada)
        si (estadoExterno == ERROR):
            WS.FinalizarExtraccionAnalizadores2(..., "ERRONEA", ...)
            canal.BasicNack(entrada.deliveryTag, requeue = false)
            listaIdsEnCurso.Remove(entrada)
```

> **Clave**: el único cambio respecto al timer actual en los analizadores asíncronos es añadir `BasicAck` o `BasicNack` al finalizar. La lógica de `TratarTareasEnCurso` permanece prácticamente intacta.

---

### 13.5 Cambios necesarios en Videoma (lado servidor)

Videoma actualmente escribe la tarea en su propia base de datos y espera que el analizador la consulte. Para la migración necesitaría:

```
// Lógica actual en Videoma al crear una tarea:
BD.INSERT INTO tareas (tipo, localizacion, idioma, estado, ...) → petid

// Lógica nueva (adicional o sustitutiva):
BD.INSERT INTO tareas (...) → petid       // puede mantenerse para trazabilidad

RabbitMQ.Publish(
    exchange   = "",
    routingKey = "ANALIZADOR_" + tipo,    // nombre de la cola destino
    body       = JSON {
        petid          = petid,
        identificador  = elemento_id,
        localizacion   = ruta_fichero,
        localizacionhttp = url_fichero,
        idioma         = idioma,
        tipo           = tipo_media
    },
    persistent = true   // el mensaje sobrevive a reinicios del broker
)
```

> Videoma puede mantener la escritura en BD en paralelo con la publicación en RabbitMQ durante un período de transición, lo que permite rollback sin pérdida de datos.

---

### 13.6 Gestión de la cola y del nombre de analizador

#### Nombrado de colas

La estrategia de nombrado de colas determina cómo se distribuyen las tareas entre instancias:


| Estrategia                    | Nombre de cola ejemplo                         | Comportamiento                                                                |
| ----------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| **Una cola por tipo**         | `ANALIZADOR_TRANSCRIPCIONES VOSK`              | Varias instancias comparten cola → balanceo automático (round-robin RabbitMQ) |
| **Una cola por instancia**    | `ANALIZADOR_TRANSCRIPCIONES VOSK_SERVIDOR01_1` | Videoma elige la instancia → requiere lógica de routing en Videoma            |
| **Cola por tipo + prioridad** | `ANALIZADOR_OCR_SD_HIGH` / `_NORMAL`           | Permite priorización con x-max-priority o colas separadas                     |


La estrategia más directa y que mejor encaja con el modelo actual (Videoma asigna por tipo) es **una cola por tipo**.

#### Registro (`AltaAnalizador`) sin cambios

El proceso de `AltaAnalizador` puede mantenerse íntegro. El `analizadorId` obtenido de Videoma sigue siendo necesario para llamar a `FinalizarExtraccionAnalizadores2` y `CicloMonitorizacion`, que continúan usando el WS SOAP.

---

### 13.7 Puntos de fricción y riesgos específicos

#### Entorno C++/CLI

El código es C++/CLI (.NET managed), no C++ estándar. Las opciones para conectar con RabbitMQ son:


| Opción                       | Pros                                                                                   | Contras                                                   |
| ---------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `**RabbitMQ.Client` (.NET)** | Librería oficial, rica en features, `#using <RabbitMQ.Client.dll>` funciona en C++/CLI | Sintaxis C++/CLI con genéricos .NET es más verbosa que C# |
| `**rabbitmq-c` (C nativo)**  | Ligero, sin dependencias .NET                                                          | Interop con C++/CLI (mixed-mode), más código de glue      |
| `**AMQP-CPP` (C++ moderno)** | API moderna orientada a callbacks                                                      | Requiere Boost.Asio o libuv; compilación compleja en VS   |


La opción `**RabbitMQ.Client` (.NET)** es la más viable para el entorno actual porque el proyecto ya usa el runtime .NET (System::Threading, System::Timers, etc.) y añadir un NuGet es trivial.

```cpp
// Ejemplo de conexión en C++/CLI con RabbitMQ.Client:
#using <RabbitMQ.Client.dll>
using namespace RabbitMQ::Client;

auto factory = gcnew ConnectionFactory();
factory->HostName = "rabbitmq-server";
factory->UserName = "user";
factory->Password = "pass";

IConnection^ conn = factory->CreateConnection();
IModel^ channel   = conn->CreateModel();

channel->QueueDeclare("ANALIZADOR_OCR_SD", true, false, false, nullptr);
channel->BasicQos(0, nThreads, false);

auto consumer = gcnew EventingBasicConsumer(channel);
consumer->Received += gcnew EventHandler<BasicDeliverEventArgs^>(
    this, &CAnalizadoresBase::OnMensajeRabbitMQ);
channel->BasicConsume("ANALIZADOR_OCR_SD", false, consumer);
```

#### Garantía de entrega y duplicados

Con `autoAck = false` y `BasicAck` manual, si el servicio se cae antes de confirmar, RabbitMQ reencola el mensaje y otra instancia (o la misma al reiniciar) lo procesa. Esto introduce la posibilidad de **tareas duplicadas** si el motor ya las procesó pero el `BasicAck` no llegó.

```
Mitigación:
→ Al arrancar, llamar a PonerErroneasTareasAnalizador() (ya existe)
→ Videoma puede detectar tareas FINALIZADA duplicadas por petid y descartarlas
→ Usar un flag de idempotencia: antes de procesar, consultar WS si petid ya está FINALIZADA
```

#### Reconexión al broker

```
// Patrón de reconexión robusta:
función MantenerConexionRabbitMQ():
    mientras (servicio_activo):
        intentar:
            si (conexion == null || conexion.isOpen == false):
                log("Intentando conectar a RabbitMQ...")
                conexion = factory.CreateConnection()
                canal    = conexion.CreateModel()
                RegistrarConsumer(canal)
                log("Conectado a RabbitMQ OK")
            esperar(5000ms)
        capturar ShutdownEventArgs:
            log("Conexión RabbitMQ caída, reintentando en 5s...")
            esperar(5000ms)
        capturar Exception como ex:
            log("Error RabbitMQ: " + ex.Message + ", reintentando en 10s...")
            esperar(10000ms)
```

#### Mensajes "poison" (tareas corruptas)

Si un mensaje causa una excepción no controlada repetidamente, sin gestión de dead-letter podría generar un bucle infinito de redelivery:

```
// Al declarar la cola, configurar Dead Letter Exchange:
argumentos["x-dead-letter-exchange"] = "ANALIZADOR_DLX"
argumentos["x-message-ttl"]          = 86400000  // 24 horas max en cola

// En BasicNack: requeue = false → el mensaje va al DLX en lugar de reintentar
canal.BasicNack(deliveryTag, requeue = false)
```

---

### 13.8 Resumen del esfuerzo estimado por componente


| Componente a cambiar                                      | Dificultad | Descripción del trabajo                                                            |
| --------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------- |
| Eliminar `mTimerTareas` y su handler                      | Baja       | Borrar el timer y su callback principal                                            |
| Añadir `RabbitMQ.Client` NuGet                            | Baja       | Instalar paquete y añadir `#using`                                                 |
| `RabbitMQ_Iniciar`: conexión + consumer                   | Media      | ~80 líneas de código nuevo en C++/CLI                                              |
| Adaptar `LanzarTareaThread` para BasicAck                 | Media      | Añadir `deliveryTag` como parámetro, `BasicAck`/`BasicNack` al final               |
| Adaptar analizadores asíncronos (SPMATIC, NUANCE, SYMBOL) | Media-Alta | Timer residual + `BasicAck` diferido, almacenar `deliveryTag` en `listaIdsEnCurso` |
| Reconexión robusta al broker                              | Media      | Hilo de watchdog para la conexión RabbitMQ                                         |
| Gestión de Dead Letter (mensajes veneno)                  | Baja-Media | Declarar DLX al crear la cola                                                      |
| Cambios en Videoma (publicar en RabbitMQ)                 | Media      | Añadir publisher en el flujo de creación de tarea en Videoma                       |
| Pruebas de integración y rollback                         | Alta       | Validar todos los tipos de analizador; plan de coexistencia polling+MQ             |


**Estimación total**: la migración del lado analizador representa aproximadamente **200-350 líneas de código nuevo/modificado** sobre una base de ~2200 líneas. El cambio de mayor riesgo es Videoma, no los analizadores.

---

### 13.9 Estrategia de migración recomendada (sin corte de servicio)

```
FASE 1 — Infraestructura (sin cambios en código):
  → Instalar y configurar RabbitMQ (con management plugin)
  → Definir colas, exchanges y políticas de DLX
  → Configurar usuarios y permisos

FASE 2 — Modo dual en Videoma:
  → Videoma publica en RabbitMQ Y sigue respondiendo al polling SOAP
  → Los analizadores siguen usando polling (sin cambios)
  → Verificar que los mensajes llegan correctamente al broker

FASE 3 — Analizadores síncronos migrados (uno a uno):
  → Migrar primero los más simples (OCR SD, ALPR, QC, VIDEO ANALYTICS)
  → Cada uno arranca con RabbitMQ consumer; se desactiva su tipo en el polling
  → Validar en entorno de pruebas, luego producción

FASE 4 — Analizadores asíncronos migrados:
  → TRANSCRIPCIONES SPMATIC/NUANCE
  → SYMBOL ANALYSIS / TEXT ANALYSIS
  → Requieren mantener el timer residual de TratarTareasEnCurso

FASE 5 — Desactivar polling en Videoma:
  → Una vez todos los analizadores usan RabbitMQ
  → Eliminar/deprecar PeticionPendienteAnalizadores en el WS SOAP
```

