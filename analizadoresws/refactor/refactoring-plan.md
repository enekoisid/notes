# Plan de Accion: Refactorizacion Modular del Sistema de Analizadores

## 1. Resumen Ejecutivo

Este documento describe el plan de refactorizacion del sistema de analizadores, actualmente un servicio Windows monolitico basado en polling SOAP, hacia una arquitectura modular event-driven con RabbitMQ.

La refactorizacion tiene dos grandes ejes de trabajo:

1. **Centralizacion de los motores de analisis en AIHub**: cada motor de analisis se convierte en un ejecutable independiente que recibe parámetros y devuelve JSON. **AIHub** es un contenedor Linux centralizado que expone una API REST unificada. Al recibir una petición, AIHub ejecuta el binario correspondiente con los parámetros (si es Linux-compatible) o hace POST a un servicio REST en una máquina Windows (si no es portable). En ambos casos, AIHub espera el JSON de respuesta y lo devuelve al llamante.
2. **Desarrollo del Factory**: un nuevo componente que consume tareas de RabbitMQ, gestiona la configuracion, invoca a **AIHub** via REST (nunca directamente a modulos individuales), moldea la respuesta obtenida y la envia a Videoma via SOAP.

### Flujo objetivo

```
Videoma --POST--> SAV --> RabbitMQ
                              |
                    +---------v--------------------+
                    |      FACTORY                  |
                    |                               |
                    |  Tipo de tarea?               |
                    |   +- Settings --> Actualiza fichero de config local
                    |   +- Analisis --> Valida JWT
                    |                  > Lee settings del fichero
                    |                  > Invoca AIHub REST API
                    |                  > Recibe respuesta cruda
                    |                  > Moldea/adapta la respuesta
                    |                  > SOAP FinalizarExtraccion -> Videoma
                    +-----------------------------------+
                              |
                              v
              +-------------------------------+
              |        AIHub (Linux container) |
              |        API REST unificada      |
              |                               |
              |  /analyze/ocr                 |
              |  /analyze/vosk                |
              |  /analyze/alpr                |
              |  /analyze/translate            |
              |  /analyze/quality-control      |
              |  ... (un endpoint por tipo)    |
              |                               |
              |  +---------------------------+|
              |  | Ejecutables Linux         ||
              |  | AIHub los ejecuta con     ||
              |  | parametros; el ejecutable ||
              |  | escribe resultado en JSON ||
              |  | compartido (exit code por ||
              |  | stdout)                   ||
              |  +---------------------------+|
              |                               |
              |  +---------------------------+|
              |  | POST a REST Windows       ||
              |  | (para .exe no portables)  ||
              |  | AIHub envia params y      ||
              |  | espera JSON de respuesta  ||
              |  +---------------------------+|
              +-------------------------------+
                              |
                    (si el modulo es Windows-only)
                              |
                              v
              +-------------------------------+
              | Windows REST Service          |
              | (ejecuta .exe nativos Windows)|
              +-------------------------------+
```

### Principios de diseno

- **AIHub como punto unico de entrada**: el Factory solo conoce la URL de AIHub. AIHub ejecuta el binario local correspondiente o hace POST a la maquina Windows segun el modulo.
- **Ejecutables, no plugins compilados**: cada motor de analisis es un ejecutable independiente que recibe parametros, escribe el resultado en un fichero JSON compartido, y devuelve un codigo de salida por stdout. AIHub invoca el ejecutable, comprueba el exit code, y lee el JSON de resultado. Para modulos via REST Windows, el resultado viene directamente en la respuesta HTTP. Esto facilita el mantenimiento y el testing individual de cada modulo.
- **Factory como orquestador**: el Factory es el unico componente que conoce RabbitMQ, Videoma (SOAP) y el fichero de settings. Su responsabilidad es conectar las piezas y adaptar las respuestas.
- **Configuracion centralizada en fichero local**: actualizable en caliente via mensajes RabbitMQ.
- **Compatibilidad hacia atras**: la comunicacion de resultados con Videoma se mantiene por SOAP en esta fase.

---

## 2. Arquitectura Objetivo

### 2.1 Diagrama de componentes

```
+----------+     +------------------------------------------------------+
| VIDEOMA  |---->|              RABBITMQ                                 |
|          |     |  +---------------+  +----------------------+          |
|          |     |  | Cola: settings|  | Cola: tareas_analysis|          |
|          |     |  +------+--------+  +----------+-----------+          |
+----^-----+     +---------+----------------------+----------------------+
     |                     |                      |
     |                       +---------v--------------------------v-----------+
     |                       |                FACTORY                         |
     |                       |                                                |
     |                       |  +-----------------+  +-----------------+     |
     |                       |  | Settings Manager|  | JWT Manager     |     |
     |                       |  | (fichero local) |  | (cache+renewal) |     |
     |                       |  +-----------------+  +-----------------+     |
     |                       |                                                |
     |    SOAP               |  +-------------------------------------+      |
     |    (resultado         |  |         Task Dispatcher              |      |
     |     moldeado)         |  |  (pool de hilos, prefetch=maxHilos) |      |
     |                       |  +--------------+----------------------+      |
     |                       |                 |                             |
     |                       |  +--------------v----------------------+      |
     |<----------------------|--|       Module Invoker (por hilo)     |      |
     |                       |  |  - Resuelve tipo de tarea           |      |
     |                       |  |  - Llama a AIHub REST API           |--+   |
     |                       |  |  - Recibe respuesta cruda           |<-+   |
     |                       |  |  - Response Shaper: moldea respuesta|      |
     |                       |  |  - Envia resultado SOAP a Videoma   |      |
     |                       |  +--------------+----------------------+      |
     |                       +-----------------+-----------------------------+
                                               |
                                          HTTP REST (unica interfaz)
                                               |
                                               v
                +--------------------------------------------------------------+
                |                     AIHub (Linux container)                    |
                |              API REST unificada - /analyze/{tipo}              |
                |                                                              |
                |  Plugins Linux-nativos:                                       |
                |  +----------+ +---------------+ +------------------+         |
                |  | ccextract| | ffmpeg/ffprobe| | ProjectX.jar     |         |
                |  | (apt)    | | (apt)         | | (Java, cross-plat)|        |
                |  +----------+ +---------------+ +------------------+         |
                |                                                              |
                |  Plugins compilados desde .exe (estudio compilacion Linux):   |
                |  +----------+ +---------------+ +------------------+         |
                |  | vosk_cmd | | OCRNet        | | azureTranslator  |         |
                |  | (.so)    | | (.so)         | | (.so)            |         |
                |  +----------+ +---------------+ +------------------+         |
                |  +----------+ +---------------+ +------------------+         |
                |  | NXIndex  | | Systran       | | VideoAnalytics6  |         |
                |  | (.so)    | | (.so)         | | (.so)            |         |
                |  +----------+ +---------------+ +------------------+         |
                |                                                              |
                |  Proxy REST -> Windows (modulos no compilables):              |
                |  +----------+                                                |
                |  | subTTX   | --> HTTP --> Windows REST Service               |
                |  +----------+                                                |
                |                                                              |
                |  Modulos que llaman APIs externas (sin .exe local):           |
                |  +----------+ +---------------+ +------------------+         |
                |  | SpeakerID| | Face Recog.   | | Whisper/X        |         |
                |  | NT       | |               | | (ASR externo)    |         |
                |  +----------+ +---------------+ +------------------+         |
                |  +----------+ +---------------+ +------------------+         |
                |  | Summarize| | Media Descript.| | Imbolos Text/Sym|         |
                |  | (Llama)  | | (Llama)       | | (API externa)   |         |
                |  +----------+ +---------------+ +------------------+         |
                +--------------------------------------------------------------+
                                               |
                            (proxy para modulos Windows-only)
                                               |
                                               v
                +--------------------------------------------------------------+
                |              Windows REST Service                             |
                |  Ejecuta .exe que no se pueden compilar a Linux               |
                |  +----------+                                                |
                |  | subTTX   |                                                |
                |  | .exe     |                                                |
                |  +----------+                                                |
                +--------------------------------------------------------------+
```

### 2.2 Separacion de responsabilidades

| Componente | Conoce RabbitMQ | Conoce Videoma | Conoce AIHub | Responsabilidad |
|---|---|---|---|---|
| **Videoma** | Si (publica) | -- | No | Crea tareas y las publica en RabbitMQ |
| **Factory** | Si (consume) | Si (SOAP) | Si (URL REST unica) | Orquesta todo el flujo; invoca AIHub para cada tarea |
| **AIHub** | No | No | -- | API REST centralizada; enruta a plugins internos o proxea a Windows REST |
| **Windows REST** | No | No | No | Ejecuta .exe Windows-only; responde a peticiones de AIHub |

Esta separacion permite que AIHub sea invocable por el Factory de Videoma, pero tambien por cualquier otro servicio futuro que necesite analisis, sin modificacion alguna.

### 2.3 Tipos de mensajes en RabbitMQ

| Tipo de mensaje | Cola | Contenido | Accion del Factory |
|---|---|---|---|
| **Actualizacion de settings** | `settings_update` | JSON con configuracion parcial o completa | Escribe en el fichero de settings local |
| **Tarea de analisis** | `tareas_analysis` | JSON con `petid`, `tipo`, `localizacion`, `idioma`, etc. | Valida JWT -> lee settings -> invoca AIHub REST -> moldea respuesta -> SOAP a Videoma |

---

## 3. Eje 1: Centralizacion de Modulos en AIHub

### 3.1 Objetivo

Construir **AIHub**, un contenedor Linux que centraliza todos los motores de analisis como plugins internos y expone una API REST unificada. El Factory solo necesita conocer la URL de AIHub.

Cada plugin dentro de AIHub:

- Recibe parametros de entrada estandarizados (JSON via HTTP REST).
- Ejecuta el analisis (localmente o delegando a una API externa / Windows REST).
- Devuelve un resultado crudo en formato JSON.
- No tiene conocimiento de Videoma, RabbitMQ, ni del Factory.

### 3.2 Contrato de comunicacion Factory -> AIHub

AIHub expone endpoints REST por tipo de analisis:

```
ENTRADA:
---------
HTTP POST http://aihub:8080/analyze/{tipo}

Payload:
{
    "localizacion": "/ruta/al/fichero.mp4",       // ruta accesible por AIHub (volumen montado)
    "localizacion_http": "http://server/file.mp4", // alternativa HTTP
    "idioma": "es-ES",                             // o parametro especifico
    "engine": "VOSK",                              // motor concreto (si aplica)
    "params": {                                    // parametros especificos del plugin
        "fps": 1,
        "threshold": 0.8,
        ...
    }
}

SALIDA:
------
{
    "status": "OK" | "ERROR",
    "result": { ... },            // JSON con el resultado del analisis
    "error_message": "",          // vacio si OK
    "output_file": "/ruta/out.srt" // opcional, si el resultado es un fichero
}
```

### 3.3 Modo de invocacion: AIHub REST unificado

El Factory **solo invoca AIHub via HTTP REST**. No hay invocacion CLI directa ni gRPC. AIHub internamente decide como ejecutar cada analisis:

| Escenario dentro de AIHub | Descripcion | Ejemplo |
|---|---|---|
| **Ejecutable Linux nativo** | AIHub ejecuta un comando del sistema con parametros; el ejecutable escribe el resultado en un fichero JSON compartido y devuelve exit code (0=ok, >0=error) | ccextractor, ffmpeg, ffprobe, `java -jar ProjectX.jar` |
| **Ejecutable compilado desde .exe** | El .exe original tenia repo git y se compilo a binario Linux; AIHub lo ejecuta con parametros; el ejecutable escribe resultado en JSON compartido y devuelve exit code | vosk_cmd, OCRNet, azureTranslator, Systran, VideoAnalytics6, Isi_alprstream |
| **Ejecutable wrapper de API externa** | AIHub ejecuta un binario que llama a una API/servicio externo, espera la respuesta, y escribe resultado en JSON compartido; devuelve exit code | speakerid_analyzer, facerec_analyzer, whisper_analyzer, sumarize_analyzer, mediadesc_analyzer, simbolos_analyzer |
| **POST a Windows REST** | AIHub hace POST al servicio REST en una maquina Windows con los parametros y espera JSON de respuesta | Word Spotting (NXIndex/NXSearch/NXTextNormalizer), ALPR Vaxtor (vaxWinAnalyzer), Subtitulos DVBText (subTTX) |

### 3.4 Catalogo de modulos de AIHub

| # | Modulo | Clase actual | Modo de invocacion en AIHub | Portable a Linux | Repo Git | Cambios necesarios |
|---|---|---|---|---|---|---|
| 1 | ALPR (OpenALPR) | `CAlpr` | Ejecutable compilado a Linux | **SI** | https://gitlab.isid.com/backend/oldschool/openalpr.git | C/C++ CMake, excluir wrappers Windows |
| 2 | ALPR (Vaxtor) | `CAlpr` | **Proxy a Windows REST** | **NO** | https://gitlab.isid.com/backend/laboratory/vaxwinanalyzer.git | C++ nativo, windows.h + libVaxWrap.lib propietaria |
| 3 | Quality Control | `CQualityControl` | Ejecutable Linux nativo | SI | -- | ffmpeg via `apt install`, sin cambios |
| 4 | OCR SD | `COcr` | Ejecutable compilado a Linux | **SI** (casi directo) | https://gitlab.isid.com/backend/oldschool/ocrnet.git | C# .NET 6.0: cambiar OpenCvSharp4.Windows -> OpenCvSharp4.runtime.linux |
| 5 | Video Analytics | `CObjectDetection` | Ejecutable compilado a Linux | **SI** (con esfuerzo) | https://gitlab.isid.com/backend/videoanalytics6.git | C# .NET 6.0: cambiar OpenCvSharp4.runtime.win -> linux + reempaquetar GStreamer/FFmpeg |
| 6 | MUFIN / AudioFingerprint | `CMufin2` | Ejecutable wrapper (proxy a servidor MUFIN) | N/A | -- | Encapsular logica de huella |
| 7 | Subtitulos (PROJECTX, ISDBT, DVBSUB) | `ServicioSubtitulos` | Ejecutable Linux nativo | SI | -- | ccextractor via apt; ProjectX.jar (Java); dvb2str.ps1 (PowerShell Core) |
| 8 | Subtitulos (DVBText) | `ServicioSubtitulos` | **Proxy a Windows REST** | **NO** (origen desconocido) | -- | subTTX.exe: sin repo, Windows-only |
| 9 | Vosk | `CVosk` | Ejecutable compilado a Linux | **SI** (tras migracion) | https://gitlab.isid.com/backend/oldschool/voskconsole.git | C# .NET Framework 4.8 -> migrar .csproj a .NET 6+. Codigo y deps ya multiplataforma |
| 10 | Whisper / WhisperX | `CWhisperS2T` | Ejecutable wrapper de API externa | N/A (API REST) | -- | Ya es API REST externa |
| 11 | SpeakerID NT | `CSpeakerID` | Ejecutable wrapper de API externa | N/A (API REST) | -- | Tres operaciones (verify/identify/diarize) |
| 12 | Face Recognition | `CFaceRecognition` | Ejecutable wrapper de API externa | N/A (API REST) | -- | Tres motores (dlib/neuro/onnx) |
| 13 | Translate (Azure) | `CTranslate` | Ejecutable compilado a Linux | **SI** (tras migracion) | https://gitlab.isid.com/backend/oldschool/azure-translator.git | C# .NET Framework 4.6.1 -> .NET 6+. Quitar Colorful.Console y System.Drawing (solo decoracion) |
| 14 | Translate (Systran) | `CTranslate` | Ejecutable compilado a Linux | **SI** (tras migracion) | https://gitlab.isid.com/backend/oldschool/systran-translator.git | C# .NET Framework 4.8 -> .NET 6+. Codigo 100% portable, sin cambios |
| 15 | Sumarizacion | `CSumarize` | Ejecutable wrapper de API externa (Llama) | N/A (API REST) | -- | Llama server externo |
| 16 | Media Description | `CMediaDescription` | Ejecutable wrapper de API externa (Llama) | N/A (API REST) | -- | Llama server externo |
| 17 | Symbol / Text Analysis | `CSymbolsText` | Ejecutable wrapper de API externa | N/A (API REST) | -- | API externa (Vicomtech) |
| 18 | OCR Documentos | `COcr` (FineReader) | Ejecutable wrapper de Videoma WS | N/A | -- | Delega en WS FineReader de Videoma |
| 19 | Word Spotting (NXIndex, NXSearch, NXTextNormalizer) | `CWordSpotting` | **Proxy a Windows REST** | **NO** | https://gitlab.isid.com/backend/oldschool/nxindex.git etc. | C# .NET Framework 4.5.2 + Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| 20 | Transcripciones SaaS (Vocapia, SPMATIC, Nuance, MS) | `ServicioTranscripcion` | Ejecutable wrapper de API externa | N/A | -- | Encapsular logica de envio + polling asincrono |

### Resultado del estudio de portabilidad de ejecutables

**PORTABLES a Linux (7 de 10 ejecutables con repo):**

| Ejecutable | Framework | Cambios necesarios |
|---|---|---|
| OCRNet.exe | C# .NET 6.0 | Cambiar NuGet `OpenCvSharp4.Windows` -> `OpenCvSharp4.runtime.linux` |
| VideoAnalytics6.exe | C# .NET 6.0 | Cambiar `OpenCvSharp4.runtime.win` -> `linux` + reempaquetar DLLs GStreamer/FFmpeg |
| Isi_alprstream.exe | C/C++ CMake | Excluir wrappers Windows del build |
| vosk_cmd.exe | C# .NET Framework 4.8 | Migrar .csproj a .NET 6+. Codigo y deps ya multiplataforma |
| azureTranslator.exe | C# .NET Framework 4.6.1 | Migrar a .NET 6+, quitar Colorful.Console y System.Drawing (solo decoracion) |
| Systran.exe | C# .NET Framework 4.8 | Migrar .csproj a .NET 6+. Codigo 100% portable |

**NO PORTABLES a Linux (3 de 10 ejecutables + 1 desconocido):**

| Ejecutable | Framework | Bloqueante |
|---|---|---|
| vaxWinAnalyzer.exe | C++ nativo | windows.h, APIs Win32, libreria propietaria libVaxWrap.lib |
| NXIndex.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| NXSearch.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| NXTextNormalizer.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| subTTX.exe | Desconocido | Origen desconocido, sin repo |

**Modulos que NO necesitan ejecutables locales (llaman APIs externas):**
- SpeakerID NT, Face Recognition, Whisper/WhisperX (ASR externo), Summarize (Llama server), Media Description (Llama server), ImbolosText/Symbols (API externa)

### 3.5 Gestion de modulos asincronos

Algunos plugins de AIHub (los que corresponden a SPMATIC, NUANCE, SYMBOL ANALYSIS, TEXT ANALYSIS) tienen un patron asincrono: se envia la tarea y luego hay que sondear el estado.

Hay dos estrategias para manejar esto:

| Estrategia | Descripcion | Pros | Contras |
|---|---|---|---|
| **A) El plugin de AIHub absorbe la asincronia** | El plugin recibe la peticion, internamente hace el polling al motor SaaS, y devuelve el resultado cuando termina (HTTP long-poll) | El Factory no necesita logica especial; todos los plugins se tratan igual | El plugin necesita logica interna de polling; la conexion HTTP puede ser muy larga |
| **B) El Factory gestiona el polling** | El plugin devuelve un `external_id` inmediatamente. El Factory tiene un timer que sondea AIHub periodicamente hasta que este listo | El plugin es mas simple (solo envio y consulta) | El Factory necesita un timer residual y logica de Ack diferido en RabbitMQ |

**Recomendacion**: Estrategia A con timeout. El plugin de AIHub se encarga del ciclo completo y el Factory establece un timeout maximo de espera. Si AIHub no responde a tiempo, el Factory marca la tarea como ERRONEA. Esto simplifica enormemente el Factory y mantiene toda la logica de cada motor dentro de AIHub.

Para plugins con tiempos de proceso muy largos (horas), se puede implementar un endpoint `/status/{id}` en AIHub, y el Factory usa la Estrategia B solo para esos casos concretos.

### 3.6 Tareas de este eje

| Tarea | Descripcion | Entregable |
|---|---|---|
| E1.1 | Definir el contrato estandar REST de AIHub: esquemas JSON de entrada/salida por tipo de analisis | Documento de especificacion de API (OpenAPI/Swagger) |
| E1.2 | Crear el esqueleto de AIHub: contenedor Linux con framework REST y sistema de plugins | Repositorio AIHub con scaffolding + plugin de ejemplo |
| E1.3 | Integrar plugins Linux-nativos (ccextractor, ffmpeg, ProjectX) | Plugins funcionales en AIHub |
| E1.4 | Estudiar compilacion a Linux de cada .exe con repo git (ver tabla de ejecutables) | Informe de viabilidad por ejecutable |
| E1.5 | Compilar e integrar como plugins .so los ejecutables viables | Plugins compilados funcionando en AIHub |
| E1.6 | Implementar proxy REST a Windows para modulos no compilables (subTTX) | Proxy funcional en AIHub + servicio REST en Windows |
| E1.7 | Integrar plugins que llaman APIs externas (SpeakerID, Face Recog, Whisper, Llama, etc.) | Plugins proxy funcionales en AIHub |
| E1.8 | Para cada plugin, crear tests de contrato: dado un input conocido, validar que el output cumple el esquema esperado | Suite de tests por plugin |
| E1.9 | Para plugins asincronos: implementar la logica de polling interna (Estrategia A) o definir endpoint `/status` (Estrategia B) | Plugin funcional con asincronia resuelta |

---

## 4. Eje 2: Desarrollo del Factory

### 4.1 Objetivo

El Factory es el componente central que orquesta el flujo completo. Es un servicio que:

- Consume mensajes de RabbitMQ.
- Gestiona la configuracion local (fichero de settings).
- Valida y renueva el JWT de Videoma.
- Invoca a **AIHub** via REST segun el tipo de tarea.
- Moldea la respuesta cruda de AIHub al formato que espera Videoma.
- Envia el resultado a Videoma via SOAP.

### 4.2 Componentes internos del Factory

#### 4.2.1 RabbitMQ Consumer

**Que reemplaza**: el timer `OnTratarTareasTimerThread` y la llamada `PeticionPendienteAnalizadores`.

**Responsabilidades**:
- Conectar a RabbitMQ y declarar las colas (`tareas_analysis`, `settings_update`).
- Configurar `BasicQos(prefetchCount = maxHilos)` para limitar la concurrencia.
- Registrar callbacks de recepcion de mensajes.
- Gestionar reconexion robusta al broker (watchdog con backoff exponencial).
- Gestionar `BasicAck` (tarea completada) y `BasicNack` (tarea fallida, sin requeue) con Dead Letter Exchange.

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.1.1 | Anadir dependencia `RabbitMQ.Client` (.NET) al proyecto |
| F.1.2 | Implementar conexion, declaracion de colas (con DLX) y consumer con `autoAck = false` |
| F.1.3 | Implementar hilo de watchdog para reconexion automatica |
| F.1.4 | Implementar routing: si el mensaje es de tipo `settings` -> Settings Manager; si es de tipo `analysis` -> Task Dispatcher |

#### 4.2.2 Settings Manager

**Que reemplaza**: la lectura de `configuracion.xml` + las multiples llamadas a `ObtenerParametroAnalizador` durante la inicializacion.

**Responsabilidades**:
- Leer el fichero de settings local (JSON) al arrancar.
- Exponer un metodo `GetSettings(tipo, engine)` thread-safe para que los hilos de tarea obtengan los parametros que necesitan.
- Recibir actualizaciones de settings via mensajes RabbitMQ y aplicarlas al fichero (hot-reload sin reinicio).

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.2.1 | Definir el esquema del fichero de settings unificado (ver seccion 5) |
| F.2.2 | Implementar `SettingsManager` con lectura al arrancar, acceso thread-safe (read-write lock) y escritura en caliente |
| F.2.3 | Implementar consumer de la cola `settings_update` |
| F.2.4 | Crear herramienta de migracion que genere el fichero de settings a partir de la configuracion actual (XML + parametros de Videoma) |

#### 4.2.3 JWT Manager

**Que reemplaza**: la llamada a `GetAuthToken()` al arrancar + la recreacion del WS en `TratarExcepcion`.

**Responsabilidades**:
- Obtener el token JWT al arrancar (`POST auth.php`).
- Cachear el token y controlar su expiracion.
- Renovar proactivamente antes de que expire.
- Exponer un metodo `GetValidToken()` que los hilos de tarea usan antes de llamar a SOAP.

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.3.1 | Implementar `JwtManager` con obtencion, cache, control de expiracion y renovacion automatica |
| F.3.2 | Integrar con el Task Dispatcher: cada hilo llama a `GetValidToken()` antes de invocar SOAP |

#### 4.2.4 Task Dispatcher (pool de hilos)

**Que reemplaza**: la logica de busqueda de thread libre en `OnTratarTareasTimerThread` + `LanzarTareaThread`.

**Responsabilidades**:
- Recibir mensajes del consumer RabbitMQ (callback).
- Lanzar cada tarea en un hilo del pool.
- La concurrencia esta regulada por `BasicQos(prefetchCount)`: RabbitMQ no entrega mas mensajes de los que hay hilos disponibles, asi que no necesita buscar hilos libres manualmente.
- Controlar timeout maximo por tarea (timer watchdog).

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.4.1 | Implementar el callback `OnMensajeRecibido` que lanza un hilo por tarea |
| F.4.2 | Implementar timer watchdog para abortar tareas que superen `TIEMPOMAXIMOTAREA` |
| F.4.3 | Implementar `BasicAck` al completar y `BasicNack` al fallar |

#### 4.2.5 Module Invoker (invocacion via AIHub)

**Que reemplaza**: los metodos `TaskVideoma_XXX` dentro de `LanzarTareaThread`.

**Responsabilidades**:
- Dado el tipo de tarea y el engine, construir la URL de AIHub correspondiente (`/analyze/{tipo}`).
- Construir el payload de entrada a partir de los datos de la tarea (de RabbitMQ) y los settings (del fichero).
- Invocar a AIHub via HTTP REST.
- Recibir la respuesta cruda.

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.5.1 | Implementar un registro de tipos de analisis: diccionario `{tipo+engine} -> {endpoint AIHub, timeout}` leido del fichero de settings |
| F.5.2 | Implementar invocador HTTP (POST JSON a AIHub, esperar respuesta) |
| F.5.3 | Implementar logica de retry configurable por tipo de analisis (con backoff) |

#### 4.2.6 Response Shaper (moldeo de respuesta)

**Que reemplaza**: la logica dispersa en cada `TaskVideoma_XXX` que formatea el resultado antes de llamar a `FinalizarExtraccionAnalizadores2`.

**Responsabilidades**:
- Recibir la respuesta cruda de AIHub.
- Transformarla al formato que Videoma espera (JSON especifico de cada tipo de analizador).
- Llamar a `FinalizarExtraccionAnalizadores2` via SOAP con el resultado moldeado, el estado (`FINALIZADA`/`ERRONEA`) y el mensaje de error si aplica.

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.6.1 | Documentar el formato de salida que Videoma espera para cada tipo de analizador (reverse-engineering del codigo actual de cada `TaskVideoma_XXX`) |
| F.6.2 | Implementar un `ResponseShaper` por tipo de analizador, o un shaper generico con reglas configurables |
| F.6.3 | Implementar la llamada SOAP de finalizacion (`FinalizarExtraccionAnalizadores2`) con creacion de WS por hilo (equivalente al actual `CrearWSTH`) |
| F.6.4 | Implementar la llamada de cambio de estado (`CicloMonitorizacion` -> `ENCURSO`) al inicio de cada tarea |

#### 4.2.7 Arranque del servicio

**Que reemplaza**: el constructor de `CAnalizadoresBase` y la secuencia de inicializacion actual.

**Flujo de arranque nuevo**:

```
1. Leer fichero de settings local
2. Obtener JWT (JwtManager.Initialize())
3. Crear WS SOAP principal
4. AltaAnalizador en Videoma (se mantiene via SOAP)
5. PonerErroneasTareasAnalizador (limpia tareas huerfanas)
6. Verificar conectividad con AIHub (health check)
7. Conectar a RabbitMQ (Consumer.Initialize())
8. Arrancar timer watchdog de timeout
9. Listo: el Factory queda a la espera de mensajes
```

**Tareas**:

| Tarea | Descripcion |
|---|---|
| F.7.1 | Implementar la secuencia de arranque con los nuevos componentes |
| F.7.2 | Mantener `AltaAnalizador` y `PonerErroneasTareasAnalizador` via SOAP (sin cambios respecto al actual) |
| F.7.3 | Implementar health check de AIHub al arrancar |

---

## 5. Estructura del Fichero de Settings

```json
{
  "global": {
    "rabbitmq": {
      "host": "rabbitmq-server",
      "port": 5672,
      "user": "analizador_user",
      "password": "***",
      "vhost": "/"
    },
    "videoma": {
      "host": "videoma-server",
      "localizacion": "videoma",
      "usuario_ws": "ws_user",
      "password_ws": "***",
      "api_key": "***"
    },
    "aihub": {
      "url": "http://aihub:8080",
      "health_endpoint": "/health",
      "timeout_default_ms": 300000
    },
    "windows_rest": {
      "url": "http://windows-machine:9090",
      "comment": "Fallback para modulos .exe que no se pueden compilar a Linux"
    },
    "proxy": {
      "url": "",
      "user": "",
      "password": ""
    },
    "max_threads": 4,
    "tiempo_maximo_tarea_ms": 3600000,
    "prioridad_tarea": 0
  },
  "modules": {
    "ALPR": {
      "aihub_endpoint": "/analyze/alpr",
      "timeout_ms": 120000,
      "engines": {
        "VAXTOR": { "params": {} },
        "OPENALPR": { "params": {} }
      },
      "params": {
        "max_plates": 10,
        "plate_country": "eu",
        "analyze_vehicle": false
      }
    },
    "TRANSCRIPCIONES": {
      "engines": {
        "VOSK": {
          "aihub_endpoint": "/analyze/transcripcion/vosk",
          "timeout_ms": 300000,
          "params": { "idioma_defecto": "es" }
        },
        "WHISPER": {
          "aihub_endpoint": "/analyze/transcripcion/whisper",
          "timeout_ms": 600000,
          "params": { "path_temp": "./temp" }
        },
        "WHISPERX": {
          "aihub_endpoint": "/analyze/transcripcion/whisperx",
          "timeout_ms": 600000,
          "params": { "path_temp": "./temp" }
        },
        "VOCAPIA": {
          "aihub_endpoint": "/analyze/transcripcion/vocapia",
          "timeout_ms": 1800000,
          "params": {
            "segment_boundary_sensitivity": "0.5",
            "new_speaker_sensitivity": "0.5"
          }
        }
      }
    },
    "OCR SD": {
      "aihub_endpoint": "/analyze/ocr-sd",
      "timeout_ms": 300000,
      "params": {
        "threads": 2,
        "fps": 1,
        "punto_x": 0, "punto_y": 0,
        "width": 0, "height": 0
      }
    },
    "SPEAKERID NT": {
      "aihub_endpoint": "/analyze/speakerid",
      "timeout_ms": 300000,
      "params": {
        "repositorio_voces": "/data/voces",
        "umbral_verificacion": 0.6
      }
    },
    "QUALITY CONTROL": {
      "aihub_endpoint": "/analyze/quality-control",
      "timeout_ms": 600000,
      "params": {
        "blackdetect_duration": "2.0",
        "blackdetect_threshold": "0.98",
        "silencedetect_duration": "2.0",
        "silencedetect_threshold": "-60",
        "freezedetect_duration": "5.0",
        "freezedetect_threshold": "0.003"
      }
    },
    "FACE RECOGNITION": {
      "aihub_endpoint": "/analyze/face-recognition",
      "timeout_ms": 500000,
      "engines": {
        "dlib": { "params": {} },
        "neuro": { "params": {} },
        "onnx": { "params": {} }
      },
      "params": {
        "action": "detect",
        "database": "/data/faces",
        "threshold1": 0.6,
        "threshold2": 0.5,
        "fps": 1
      }
    },
    "SUBTITULOS": {
      "aihub_endpoint": "/analyze/subtitulos",
      "timeout_ms": 300000,
      "comment": "AIHub usa ccextractor/ProjectX nativos; subTTX via proxy Windows REST",
      "params": {}
    },
    "TRANSLATE": {
      "aihub_endpoint": "/analyze/translate",
      "timeout_ms": 300000,
      "engines": {
        "AZURE": { "params": {} },
        "SYSTRAN": { "params": {} }
      },
      "params": {}
    },
    "SUMMARIZE": {
      "aihub_endpoint": "/analyze/summarize",
      "timeout_ms": 600000,
      "params": {}
    },
    "MEDIA DESCRIPTION": {
      "aihub_endpoint": "/analyze/media-description",
      "timeout_ms": 600000,
      "params": {}
    },
    "VIDEO ANALYTICS": {
      "aihub_endpoint": "/analyze/video-analytics",
      "timeout_ms": 600000,
      "params": {}
    }
  }
}
```

Todas las entradas de `modules` apuntan a endpoints de AIHub (`aihub_endpoint`). La URL base de AIHub y la URL de fallback del servicio REST Windows se configuran en `global`. El Factory construye la URL completa como `global.aihub.url + module.aihub_endpoint`.

---

## 6. Plan de Ejecucion por Fases

### Fase 0 -- Diseno y preparacion

| Tarea | Descripcion | Entregable |
|---|---|---|
| F0.1 | Instalar y configurar RabbitMQ (management plugin, DLX, usuarios, permisos) | Broker operativo |
| F0.2 | Definir contrato estandar REST de AIHub: esquemas JSON de entrada y salida por tipo de analisis (tarea E1.1) | Especificacion OpenAPI/Swagger |
| F0.3 | Definir esquema del fichero de settings (tarea F.2.1) | Esquema JSON validado |
| F0.4 | Crear esqueleto de AIHub: contenedor Linux con framework REST y sistema de plugins (tarea E1.2) | Repositorio AIHub con scaffolding |
| F0.5 | Documentar el formato de respuesta que Videoma espera por cada tipo de analizador (tarea F.6.1) | Documento de mapeo response |

**Criterio de completitud**: RabbitMQ operativo, contratos definidos, esqueleto de AIHub funcional, esquemas aprobados por el equipo.

### Fase 1 -- Factory funcional + AIHub con plugin piloto

| Tarea | Descripcion | Entregable |
|---|---|---|
| F1.1 | Implementar Settings Manager (tareas F.2.1-F.2.4) | Componente funcional + fichero de settings de ejemplo |
| F1.2 | Implementar JWT Manager (tareas F.3.1-F.3.2) | Componente funcional |
| F1.3 | Implementar RabbitMQ Consumer (tareas F.1.1-F.1.4) | Componente funcional con reconexion |
| F1.4 | Implementar Task Dispatcher (tareas F.4.1-F.4.3) | Pool de hilos con Ack/Nack |
| F1.5 | Implementar Module Invoker -- modo HTTP contra AIHub (tareas F.5.1-F.5.2) | Invocador HTTP funcional |
| F1.6 | Implementar Response Shaper + SOAP de finalizacion (tareas F.6.2-F.6.4) | Shaper + comunicacion SOAP |
| F1.7 | Implementar arranque del servicio con health check de AIHub (tareas F.7.1-F.7.3) | Servicio arranca y se conecta |
| F1.8 | Integrar plugin piloto en AIHub (Quality Control recomendado: usa ffmpeg nativo Linux) | Plugin QC funcional en AIHub |
| F1.9 | Test end-to-end: Videoma -> RabbitMQ -> Factory -> AIHub /analyze/quality-control -> resultado SOAP -> Videoma | Flujo completo validado |

**Criterio de completitud**: una tarea de Quality Control publicada en RabbitMQ se procesa correctamente de extremo a extremo via AIHub.

### Fase 2 -- Integracion de plugins Linux-nativos y APIs externas en AIHub

Integrar en AIHub los plugins que no requieren compilacion de .exe:

| Orden | Modulo AIHub | Tipo | Notas |
|---|---|---|---|
| 1 | Quality Control | Linux-nativo (ffmpeg) | Ya hecho en Fase 1 |
| 2 | Subtitulos (ccextractor + ProjectX) | Linux-nativo | ccextractor via apt; ProjectX.jar cross-plat |
| 3 | Whisper / WhisperX | Ejecutable wrapper de API externa | Ya son API REST; solo estandarizar E/S |
| 4 | SpeakerID NT | Ejecutable wrapper de API externa | Tres operaciones |
| 5 | Face Recognition | Ejecutable wrapper de API externa | Tres motores |
| 6 | Translate (si usa solo SaaS) | Ejecutable wrapper de API externa | Wrapper sobre SaaS |
| 7 | Sumarizacion | Ejecutable wrapper de API externa (Llama) | Wrapper sobre Llama server |
| 8 | Media Description | Ejecutable wrapper de API externa (Llama) | Wrapper sobre Llama server |
| 9 | Symbol / Text Analysis | Ejecutable wrapper de API externa | API Vicomtech |
| 10 | MUFIN / AudioFingerprint | Proxy a servidor MUFIN | Encapsular logica de huella |
| 11 | OCR Documentos | Proxy a Videoma WS FineReader | Delegacion |
| 12 | Video Analytics (si ya es servicio externo) | Ejecutable wrapper de API externa | Estandarizar contrato |

**Por cada plugin**:
- Implementar el plugin en AIHub con el contrato REST definido en Fase 0.
- Crear tests de contrato.
- Implementar el Response Shaper especifico en el Factory.
- Test de integracion end-to-end con el Factory.

### Fase 3 -- Estudio de compilacion de .exe a Linux e integracion en AIHub

| Tarea | Descripcion | Entregable |
|---|---|---|
| F3.1 | Compilar e integrar los 6 ejecutables portables como plugins Linux en AIHub | Plugins compilados funcionando |
| F3.2 | Implementar proxy Windows REST en AIHub para los 4 ejecutables no portables + subTTX | Proxy funcional |
| F3.3 | Desplegar servicio REST en maquina Windows para ejecutar .exe no portables | Servicio Windows REST operativo |

**Resultado del estudio de portabilidad (completado):**

**PORTABLES a Linux (6 ejecutables) - compilar e integrar en AIHub:**

| Ejecutable | Framework | Cambios necesarios |
|---|---|---|
| OCRNet.exe | C# .NET 6.0 | Cambiar NuGet OpenCvSharp4.Windows -> OpenCvSharp4.runtime.linux |
| VideoAnalytics6.exe | C# .NET 6.0 | Cambiar OpenCvSharp4.runtime.win -> linux + reempaquetar GStreamer/FFmpeg |
| Isi_alprstream.exe | C/C++ CMake | Excluir wrappers Windows del build |
| vosk_cmd.exe | C# .NET Framework 4.8 | Migrar .csproj a .NET 6+. Codigo y deps ya multiplataforma |
| azureTranslator.exe | C# .NET Framework 4.6.1 | Migrar a .NET 6+, quitar Colorful.Console y System.Drawing (solo decoracion) |
| Systran.exe | C# .NET Framework 4.8 | Migrar .csproj a .NET 6+. Codigo 100% portable |

**NO PORTABLES a Linux (4 ejecutables + 1 desconocido) - Windows REST obligatorio:**

| Ejecutable | Framework | Bloqueante |
|---|---|---|
| vaxWinAnalyzer.exe | C++ nativo | windows.h, APIs Win32, libreria propietaria libVaxWrap.lib |
| NXIndex.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| NXSearch.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| NXTextNormalizer.exe | C# .NET Framework 4.5.2 | Nexidia.Workbench.NET.dll propietaria (sin version Linux) |
| subTTX.exe | Desconocido | Origen desconocido, sin repo |

### Fase 4 -- Modulos asincronos

| Orden | Modulo AIHub | Complejidad | Notas |
|---|---|---|---|
| 1 | Transcripciones Vocapia/SPMATIC | Alta | El plugin de AIHub absorbe el polling al SaaS (Estrategia A) |
| 2 | Transcripciones Nuance | Alta | Idem Vocapia |
| 3 | Transcripciones MS | Media | Verificar si es sincrono o asincrono |
| 4 | Symbol Analysis | Media-Alta | Vicomtech; encapsular polling |
| 5 | Text Analysis | Media-Alta | Mismo motor que Symbol, diferente operacion |

### Fase 5 -- Cambios en Videoma (en paralelo con Fases 3-4)

| Tarea | Descripcion |
|---|---|
| F5.1 | Implementar publisher RabbitMQ en Videoma: al crear una tarea, publicar en `tareas_analysis` |
| F5.2 | Implementar publisher de settings: al modificar parametros de analizadores, publicar en `settings_update` |
| F5.3 | Modo dual: Videoma sigue respondiendo al polling SOAP mientras coexistan analizadores legacy |
| F5.4 | Desactivar polling una vez todos los analizadores esten migrados |

### Fase 6 -- Estabilizacion y cierre

| Tarea | Descripcion |
|---|---|
| F6.1 | Pruebas de carga: multiples instancias del Factory consumiendo de la misma cola, todas contra AIHub |
| F6.2 | Pruebas de resiliencia: caida de AIHub, caida del broker, caida de Windows REST, mensajes poison, reconexiones |
| F6.3 | Monitorizacion: alertas en RabbitMQ Management (colas acumuladas, DLX, consumers caidos) + health check de AIHub |
| F6.4 | Documentacion final: arquitectura AIHub, guia de creacion de nuevos plugins, runbook operativo |
| F6.5 | Eliminar codigo legacy: timer de polling, `OnTratarTareasTimerThread`, instanciacion directa de motores en `CAnalizadoresBase` |

---

## 7. Riesgos y Mitigaciones

| Riesgo | Impacto | Probabilidad | Mitigacion |
|---|---|---|---|
| **Accesibilidad de ficheros**: AIHub es un contenedor Linux y no tiene acceso directo a rutas UNC/Windows del fichero multimedia | Alto | Alta | Montar volumenes compartidos (NFS/CIFS) en el contenedor AIHub o usar `localizacionhttp` (descarga HTTP). Definir en settings que modo usa cada modulo. |
| **Viabilidad de compilacion .exe a Linux**: algunos ejecutables pueden depender de APIs Windows (WinAPI, COM, .NET Framework) que impiden la compilacion a Linux | Alto | Media-Alta | Analizar dependencias de cada repo en Fase 3. Para los no viables, mantener proxy a Windows REST como fallback permanente. |
| **Rendimiento de proxy Windows REST**: la latencia adicional de proxear via red a una maquina Windows puede impactar modulos con alta frecuencia de uso | Medio | Media | Benchmarking en Fase 3. Priorizar compilacion Linux para modulos de alto uso (Vosk, OCR, ALPR). |
| **Tareas duplicadas** tras caida del Factory antes de BasicAck | Medio | Media | `PonerErroneasTareasAnalizador` al arrancar + idempotencia en Videoma (descartar FINALIZADA duplicadas por petid) |
| **Mensajes poison** que causan bucle de fallos | Alto | Baja | Dead Letter Exchange configurado; `BasicNack(requeue=false)`; monitorizacion de cola DLX |
| **Perdida de conexion RabbitMQ** | Alto | Media | Watchdog con reconexion automatica y backoff exponencial |
| **AIHub no responde** (caido o lento) | Alto | Media | Timeout configurable por tipo de analisis en settings; retry con backoff; health check al arrancar; circuit breaker opcional |
| **Formato de respuesta de AIHub no coincide con lo que Videoma espera** | Medio | Media | Response Shaper por tipo de analizador; tests de contrato obligatorios por plugin antes de integrar |
| **Regresion en modulos asincronos** | Alto | Media | Migrarlos en fase posterior; tests exhaustivos; modo dual en Videoma hasta validar |
| **subTTX.exe sin repositorio**: no se puede compilar ni auditar; dependencia opaca | Medio | Alta | Investigar origen. Si no se localiza el codigo fuente, mantener en Windows REST indefinidamente y documentar como deuda tecnica. |
| **Disponibilidad de la maquina Windows**: si la maquina Windows que ejecuta el servicio REST cae, los modulos Windows-only quedan inoperativos | Alto | Media | Monitorizacion activa del servicio Windows REST; redundancia si es critico; priorizar compilacion a Linux para reducir dependencia |

---

## 8. Estimacion de Esfuerzo

| Fase | Descripcion | Esfuerzo estimado |
|---|---|---|
| Fase 0 | Diseno, infraestructura, contratos, esqueleto AIHub | 2-3 semanas |
| Fase 1 | Factory funcional + AIHub con plugin piloto | 2-3 semanas |
| Fase 2 | Plugins Linux-nativos y APIs externas en AIHub (12 plugins) | 3-4 semanas |
| Fase 3 | Estudio compilacion .exe + integracion en AIHub o proxy Windows | 3-5 semanas |
| Fase 4 | Plugins asincronos | 2-3 semanas |
| Fase 5 | Cambios en Videoma (en paralelo) | 1-2 semanas |
| Fase 6 | Estabilizacion, pruebas, documentacion | 2-3 semanas |
| **Total** | | **15-23 semanas** |

---

## 9. Criterios de Aceptacion Global

- Toda tarea publicada por Videoma en RabbitMQ es procesada por exactamente un Factory, sin perdida ni duplicacion en operacion normal.
- AIHub expone una API REST unificada que permite invocar cualquier tipo de analisis con un contrato estandar de entrada/salida.
- Un nuevo plugin se puede anadir a AIHub e integrarlo con el Factory unicamente registrandolo en el fichero de settings (nuevo `aihub_endpoint`), sin modificar codigo del Factory.
- Los ejecutables .exe con repositorio git han sido evaluados para compilacion a Linux; los viables funcionan como plugins nativos en AIHub.
- Los modulos Windows-only (subTTX y cualquier .exe no compilable) son accesibles via proxy REST a la maquina Windows.
- La actualizacion de settings via RabbitMQ se refleja sin reiniciar el servicio.
- El Factory se recupera automaticamente de caidas del broker RabbitMQ, de AIHub y del servicio Windows REST.
- El tiempo de latencia entre la publicacion de una tarea y el inicio de su procesamiento es inferior a 1 segundo.
- Todos los tipos de analizador del catalogo actual estan migrados y operativos como plugins de AIHub.
- El formato de resultado enviado a Videoma via SOAP es identico al actual para cada tipo de analizador (no requiere cambios en el lado receptor de Videoma).
