# Integración RabbitMQ: Worker Universal de Analizadores

## 1. Ciclo completo de una petición

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. Videoma crea la tarea en su BD                                          │
│     → serializa la tarea + JWT + analizadorId en un mensaje JSON            │
│     → publica el mensaje en RabbitMQ                                        │
│                                                                             │
│  ┌──────────┐   publish(msg)   ┌────────────────────┐                       │
│  │ Videoma  │ ──────────────►  │   RabbitMQ Broker  │                       │
│  │          │                  │   cola: ANALIZADOR │                       │
│  │          │  ◄────────────── │                    │                       │
│  └──────────┘   FinalizarEx2   └────────┬───────────┘                       │
│        ▲                                │ deliver(msg)                      │
│        │                                ▼                                   │
│        │                        ┌────────────────────┐                      │
│        │    WS SOAP             │  Worker Universal  │                      │
│        │  (JWT heredado)        │                    │                      │
│        └─────────────────────── │  1. extrae JWT     │                      │
│                                 │  2. crea WS        │                      │
│                                 │  3. dispatch tipo  │                      │
│                                 │  4. ejecuta motor  │                      │
│                                 │  5. devuelve res.  │                      │
│                                 └────────────────────┘                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Ciclo simplificado:**

```
Videoma ──► RabbitMQ ──► Worker ──► [Motor de análisis] ──► Videoma
   (POST tarea)          (consume)    (OCR, VOSK, etc.)     (WS resultado)
```

El Worker no sabe de antemano qué tipo de análisis va a ejecutar. Lee el tipo del mensaje, instancia el motor adecuado, ejecuta el análisis y devuelve el resultado a Videoma usando el JWT que llegó en el propio mensaje.

---

## 2. Estructura del mensaje RabbitMQ

Videoma publica en la cola un mensaje JSON con toda la información que el Worker necesita para ejecutar la tarea **sin conocimiento previo de configuración**:

```json
{
  "jwtToken":     "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "analizadorId": 42,
  "videomHost":   "videoma.empresa.com",
  "videomPath":   "videoma",
  "peticion": {
    "petid":            1234,
    "identificador":    5678,
    "localizacion":     "\\\\servidor\\media\\archivo.mp4",
    "localizacionhttp": "http://servidor/media/archivo.mp4",
    "idioma":           "es-ES",
    "tipo":             "VIDEO",
    "tipoAnalizador":   "TRANSCRIPCIONES VOSK"
  }
}
```

### Descripción de campos

| Campo                        | Origen     | Descripción                                                                    |
|------------------------------|------------|--------------------------------------------------------------------------------|
| `jwtToken`                   | Videoma    | JWT que el Worker usará para autenticarse en las llamadas WS de devolución     |
| `analizadorId`               | Videoma    | ID del analizador registrado en Videoma (necesario para `FinalizarExtraccion`) |
| `videomHost`                 | Videoma    | Host del servidor Videoma al que devolver resultados                           |
| `videomPath`                 | Videoma    | Path virtual de Videoma (p.ej. `"videoma"`)                                    |
| `peticion.petid`             | Videoma    | ID interno de la petición (identificador de tarea en Videoma)                  |
| `peticion.identificador`     | Videoma    | ID del elemento media en Videoma                                               |
| `peticion.localizacion`      | Videoma    | Ruta UNC local del fichero multimedia                                          |
| `peticion.localizacionhttp`  | Videoma    | URL HTTP del fichero (para motores que lo necesiten, p.ej. Whisper)            |
| `peticion.idioma`            | Videoma    | Idioma o parámetros adicionales según el tipo de analizador                    |
| `peticion.tipo`              | Videoma    | Tipo de media: `VIDEO`, `AUDIO`, `DOCUMENT`, etc.                              |
| `peticion.tipoAnalizador`    | Videoma    | Tipo de análisis a ejecutar (ver tabla de tipos en sección 6)                  |

> **Nota sobre `tipoAnalizador`**: es el campo clave que determina qué motor ejecuta el Worker. Debe coincidir exactamente con los nombres de tipo que ya usa el sistema actual (p.ej. `"TRANSCRIPCIONES VOSK"`, `"OCR SD"`, `"SPEAKERID NT"`, etc.).

---

## 3. Arquitectura del Worker

### Diferencia clave con el modelo actual

| Modelo actual (polling)                        | Modelo nuevo (Worker + RabbitMQ)                    |
|------------------------------------------------|-----------------------------------------------------|
| Un proceso por tipo de analizador              | **Un único proceso** que maneja cualquier tipo      |
| Se registra al arrancar para un tipo fijo      | No necesita `AltaAnalizador` (ID viene del mensaje) |
| Obtiene su propio JWT al arrancar              | Hereda el JWT del mensaje RabbitMQ                  |
| Timer consulta Videoma cada N segundos         | Consumer RabbitMQ recibe mensajes en tiempo real    |
| Thread pool propio con límite manual           | `BasicQos(prefetchCount)` controla la concurrencia  |

### Diagrama interno del Worker

```
┌──────────────────────────────────────────────────────────────────┐
│                        Worker Universal                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  RabbitMQ Consumer                                       │    │
│  │  cola: "ANALIZADOR"  (única cola para todos los tipos)   │    │
│  │  BasicQos(prefetchCount = MAX_THREADS)                   │    │
│  └────────────────────────┬─────────────────────────────────┘    │
│                           │ OnMensajeRecibido(msg)               │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Dispatcher                                              │    │
│  │  switch(msg.tipoAnalizador) {                            │    │
│  │    "OCR SD"              → TaskVideoma_OCR()             │    │
│  │    "TRANSCRIPCIONES VOSK"→ TaskVideoma_Vosk()            │    │
│  │    "SPEAKERID NT"        → TaskVideoma_SpeakerId()       │    │
│  │    "ALPR"                → TaskVideoma_ALPR()            │    │
│  │    ... (todos los tipos actuales)                        │    │
│  │  }                                                       │    │
│  └────────────────────────┬─────────────────────────────────┘    │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────────┐    │
│  │  Motor de análisis (instanciado por tarea, no al inicio) │    │
│  │  COcr / CVosk / CSpeakerID / CAlpr / CObjectDetection... │    │
│  └────────────────────────┬─────────────────────────────────┘    │
│                           │ resultado JSON                       │
│  ┌────────────────────────▼─────────────────────────────────┐    │
│  │  WS Client (instanciado por tarea con JWT heredado)      │    │
│  │  FinalizarExtraccionAnalizadores2(petid, analizadorId,   │    │
│  │                                   estado, error, result) │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Arranque del Worker

El arranque se simplifica drásticamente respecto al actual. Ya no hay registro en Videoma, ni obtención de JWT, ni inicialización de motores para un tipo fijo.

```
función Arrancar():

    // 1. Leer solo la configuración de conexión a RabbitMQ
    //    (del XML local o variables de entorno del contenedor/servicio)
    config = LeerConfiguracionLocal()
    //   config.rabbitHost, config.rabbitPort
    //   config.rabbitUser, config.rabbitPassword
    //   config.rabbitVhost
    //   config.maxThreads          ← límite de concurrencia
    //   config.nombreCola          ← "ANALIZADOR" (o configurable)
    //   config.tempDir             ← directorio temporal para ficheros intermedios

    log("Worker arrancando. Cola: " + config.nombreCola)

    // 2. Conectar a RabbitMQ y registrar el consumer
    RabbitMQ_Iniciar(config)

    // 3. El worker queda en escucha. No hay timer, no hay polling.
    log("Worker listo. Esperando mensajes...")
    EsperarIndefinidamente()
```

> No se llama a `GetAuthToken`, `AltaAnalizador`, `CrearWS`, `LeerParametrosAnalizador` ni `PonerErroneasTareasAnalizador` en el arranque. Todo esto ocurre **por mensaje**, usando el contexto que llega en cada uno.

---

## 5. Procesamiento de un mensaje

### 5.1 Recepción y validación

```
función OnMensajeRecibido(deliveryTag, bodyBytes):

    intentar:
        // Deserializar el mensaje
        msg = JSON.Deserializar<MensajeAnalizador>(bodyBytes)

        // Validaciones básicas antes de hacer nada
        si (msg.jwtToken == null || msg.jwtToken == ""):
            log.error("Mensaje sin JWT. petid=" + msg.peticion.petid)
            BasicNack(deliveryTag, requeue = false)   // mensaje inválido, a DLX
            retornar

        si (msg.analizadorId <= 0):
            log.error("Mensaje sin analizadorId. petid=" + msg.peticion.petid)
            BasicNack(deliveryTag, requeue = false)
            retornar

        si (msg.peticion.tipoAnalizador == null || msg.peticion.tipoAnalizador == ""):
            log.error("Mensaje sin tipoAnalizador. petid=" + msg.peticion.petid)
            BasicNack(deliveryTag, requeue = false)
            retornar

        // Lanzar el procesamiento en un thread del pool
        // BasicQos garantiza que no habrá más de maxThreads mensajes activos
        Thread.Start(() => ProcesarTarea(deliveryTag, msg))

    capturar Exception como ex:
        log.error("Error al deserializar mensaje: " + ex.Message)
        BasicNack(deliveryTag, requeue = false)
```

### 5.2 Creación del Web Service con JWT heredado

Cada mensaje lleva su propio JWT. El Worker crea una instancia del WS **por tarea**, no hay un WS compartido:

```
función CrearWSConJWT(msg) → WSDLVideomaService:

    ws = new WSDLVideomaService()

    // Construir la URL con el JWT del mensaje (mismo patrón que el actual)
    ws.Url = "http://{msg.videomHost}/{msg.videomPath}/includes/utilsws/servicios.php"
           + "?videoma_jwt_token=" + msg.jwtToken

    // Las credenciales SOAP también pueden ir en el JWT o ser fijas del worker
    autenticacion = new Autenticacion()
    autenticacion.usuariows  = ""    // vacío si el JWT es suficiente
    autenticacion.passwordws = ""
    ws.AutenticacionValue = autenticacion

    ws.Timeout = 3600000   // 1 hora, igual que ahora

    retornar ws
```

### 5.3 Notificación de inicio y ejecución

```
función ProcesarTarea(deliveryTag, msg):

    ws = CrearWSConJWT(msg)
    peticion = msg.peticion
    tiempoInicio = DateTime.Now

    intentar:
        // Notificar a Videoma que la tarea está en curso
        // (igual que CicloMonitorizacion actual)
        ws.CicloMonitorizacion(
            msg.analizadorId, "ANALIZANDO",
            peticion.petid, "ENCURSO", "", null)

        // Obtener parámetros del motor desde Videoma usando el JWT heredado
        // (igual que ObtenerParametroAnalizador actual - sin cambios)
        // → se hace dentro de cada TaskVideoma_XXX, no aquí

        // Dispatch al motor correspondiente
        [estado, error, resultado] = Dispatch(msg.peticion.tipoAnalizador, ws, peticion)

        // Devolver resultado a Videoma
        DevolverResultado(ws, msg.analizadorId, peticion.petid, estado, error, resultado)

        // Confirmar al broker: mensaje procesado OK
        BasicAck(deliveryTag)

    capturar TimeoutException:
        error = "Tiempo máximo de tarea superado"
        DevolverResultado(ws, msg.analizadorId, peticion.petid, "ERRONEA", error, "")
        BasicNack(deliveryTag, requeue = false)

    capturar Exception como ex:
        DevolverResultado(ws, msg.analizadorId, peticion.petid, "ERRONEA", ex.Message, "")
        BasicNack(deliveryTag, requeue = false)
        log.error("Error en tarea petid=" + peticion.petid + ": " + ex.Message)
```

---

## 6. Dispatch por tipo de analizador

El Dispatcher es el núcleo del Worker universal. Lee `tipoAnalizador` del mensaje y llama al método `TaskVideoma_XXX` correspondiente, que **permanece sin cambios internos** respecto al código actual.

```
función Dispatch(tipoAnalizador, ws, peticion) → (estado, error, resultado):

    motor = null

    según tipoAnalizador:

        caso "TRANSCRIPCIONES VOSK":
            motor = new CVosk(log)
            resultado = TaskVideoma_Vosk(
                peticion.tipoMotor, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "TRANSCRIPCIONES WHISPER":
        caso "TRANSCRIPCIONES WHISPERX":
            motor = new CWhisperS2T(log)
            si (tipoAnalizador.Contains("WHISPERX")):
                resultado = TaskVideoma_WhisperX(
                    peticion.tipoMotor, peticion.urlMotor, peticion,
                    proxyurl, proxyuser, proxypwd, motor)
            sino:
                resultado = TaskVideoma_Whisper(
                    peticion.tipoMotor, peticion.urlMotor, peticion,
                    proxyurl, proxypwd, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "TRANSCRIPCIONES VOCAPIA":
        caso "TRANSCRIPCIONES SPMATIC":
        caso "TRANSCRIPCIONES NUANCE":
            // Analizadores asíncronos: ver sección 6.1
            retornar Dispatch_Asincrono(tipoAnalizador, ws, peticion)

        caso "SPEAKERID NT":
            motor = new CSpeakerID(log, peticion.tipoMotor)
            resultado = TaskVideoma_SpeakerId(
                proxyurl, proxyuser, proxypwd,
                peticion.tipoMotor, peticion.usaSaas,
                ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "OCR SD":
        caso "OCR":
            motor = new COcr(log)
            resultado = TaskVideoma_OCR(
                peticion.tipoMotor, ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "ALPR":
            motor = new CAlpr(log, tiempoMaxWaitProcess)
            resultado = TaskVideoma_ALPR(
                peticion.tipoMotor, ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "QUALITY CONTROL":
            motor = new CQualityControl(log)
            resultado = TaskVideoma_QC(ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "VIDEO ANALYTICS":
            motor = new CObjectDetection(log, tiempoMaxWaitProcess)
            resultado = TaskVideoma_VideoAnalytics(
                peticion.tipoMotor, ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "FACE RECOGNITION":
        caso "FACE RECOGNITION NEURO":
        caso "FACE RECOGNITION ONNX":
            motor = new CFaceRecognition(log)
            resultado = TaskVideoma_FaceRecognition(
                tipoAnalizador, proxyurl, proxyuser, proxypwd,
                peticion.usaSaas, ws, peticion,
                peticion.urlMotor, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "SUBTITULOS":
            motor = new ServicioSubtitulos(log)
            resultado = TaskVideoma_Subtitulos(
                peticion.tipoMotor, ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "WORD SPOTTING SD":
            motor = new CWordSpotting(log)
            resultado = TaskVideoma_WordSpotting(ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "MUFIN":
        caso "AUDIOFINGERPRINT":
            motor = new CMufin2(log, tiempoMaxWaitProcess, peticion.tipoMotor)
            resultado = TaskVideoma_MufinAudioFingerPrint(
                tipoAnalizador, peticion.tipoMotor,
                proxyurl, proxyuser, proxypwd, ws, peticion, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "TRANSLATE":
            motor = new CTranslate(log, tiempoMaxWaitProcess)
            resultado = TaskVideoma_Translate(ws, peticion, tempDir, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "SUMARIZACION":
            motor = new CSumarize(log, tiempoMaxWaitProcess)
            resultado = TaskVideoma_Sumarize(ws, peticion, tempDir, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "MEDIA DESCRIPTION":
            motor = new CMediaDescription(log, tiempoMaxWaitProcess)
            resultado = TaskVideoma_MediaDescription(ws, peticion, tempDir, motor)
            retornar (motor.estado, motor.msgUltimoError, resultado)

        caso "SYMBOL ANALYSIS":
        caso "TEXT ANALYSIS":
            // Analizador asíncrono: ver sección 6.1
            retornar Dispatch_Asincrono(tipoAnalizador, ws, peticion)

        defecto:
            log.error("Tipo de analizador desconocido: " + tipoAnalizador)
            retornar ("ERRONEA", "Tipo de analizador no reconocido: " + tipoAnalizador, "")
```

### 6.1 Casos asíncronos (SPMATIC, NUANCE, SYMBOL ANALYSIS, TEXT ANALYSIS)

Estos analizadores envían la tarea a un motor externo y no tienen el resultado inmediatamente. El `deliveryTag` de RabbitMQ debe mantenerse **sin confirmar** hasta que el motor externo termine.

```
función Dispatch_Asincrono(tipoAnalizador, ws, peticion, deliveryTag, msg):

    // Enviar al motor externo y obtener un ID externo de seguimiento
    idExterno = motor.EnviarTarea(peticion)

    si (idExterno == null || idExterno == ""):
        retornar ("ERRONEA", "No se pudo enviar tarea al motor externo", "")

    // Registrar en la lista de seguimiento del Worker
    // (en lugar del listaIdsEnCurso actual, ahora también guarda deliveryTag y msg)
    listaAsync.Add({
        idExterno    : idExterno,
        petid        : peticion.petid,
        analizadorId : msg.analizadorId,
        deliveryTag  : deliveryTag,       // ← nuevo campo respecto al modelo actual
        ws           : ws,                // ← instancia WS con JWT del mensaje
        tiempoInicio : DateTime.Now,
        tipoAnalizador: tipoAnalizador
    })

    // El thread termina aquí SIN hacer BasicAck todavía
    // El timer de seguimiento (ver abajo) hará el BasicAck cuando el motor termine
    retornar null   // señal de "pendiente de confirmación asíncrona"


// Timer de seguimiento asíncrono (el único timer que subsiste en el Worker):
Timer cada 40 segundos:
    para cada entrada en listaAsync:

        estadoExterno = motor.ConsultarEstado(entrada.idExterno)

        si (estadoExterno == TERMINADO):
            resultado = motor.ObtenerResultado(entrada.idExterno)
            DevolverResultado(
                entrada.ws, entrada.analizadorId,
                entrada.petid, "FINALIZADA", "", resultado)
            BasicAck(entrada.deliveryTag)
            listaAsync.Remove(entrada)

        si (estadoExterno == ERROR):
            DevolverResultado(
                entrada.ws, entrada.analizadorId,
                entrada.petid, "ERRONEA", motor.msgError, "")
            BasicNack(entrada.deliveryTag, requeue = false)
            listaAsync.Remove(entrada)

        si (DateTime.Now - entrada.tiempoInicio > TIEMPOMAXIMOTAREA):
            DevolverResultado(
                entrada.ws, entrada.analizadorId,
                entrada.petid, "ERRONEA", "Timeout tarea asíncrona", "")
            BasicNack(entrada.deliveryTag, requeue = false)
            listaAsync.Remove(entrada)
```

---

## 7. Devolución de resultados a Videoma

La devolución es idéntica a la actual (`FinalizarExtraccionAnalizadores2`). La única diferencia es que el WS se crea con el JWT del mensaje en lugar del JWT global del servicio:

```
función DevolverResultado(ws, analizadorId, petid, estado, error, resultado):

    intentar:
        log("FinalizarExtraccion → petid:{petid} analizadorId:{analizadorId} estado:{estado}")

        ws.FinalizarExtraccionAnalizadores2(
            petid,        // ID de la petición en Videoma
            analizadorId, // ID del analizador registrado en Videoma (del mensaje)
            estado,       // "FINALIZADA" o "ERRONEA"
            error,        // descripción del error (vacío si OK)
            resultado     // JSON con los resultados del análisis
        )

    capturar WebException como ex:
        log.error("Error al devolver resultado a Videoma: " + ex.Message)
        // Si falla la devolución, el BasicAck no se hará,
        // RabbitMQ reentregará el mensaje y el Worker reintentará
        lanzar ex   // re-throw para que ProcesarTarea haga el BasicNack
```

> Los métodos `ObtenerParametroAnalizador` que cada `TaskVideoma_XXX` llama internamente también usarán el mismo WS instanciado con el JWT del mensaje, por lo que no requieren ningún cambio.

---

## 8. Pseudocódigo completo

```
═══════════════════════════════════════════════════════════════
 ARRANQUE DEL WORKER
═══════════════════════════════════════════════════════════════

config = LeerXML("configuracion.xml")
  ├─ rabbitHost, rabbitPort, rabbitUser, rabbitPwd, rabbitVhost
  ├─ nombreCola = "ANALIZADOR"
  ├─ maxThreads = 4
  └─ tempDir    = ".\temp"

conexion = RabbitMQ.CreateConnection(config.rabbit*)
canal    = conexion.CreateChannel()

canal.QueueDeclare(
  queue     = config.nombreCola,
  durable   = true,
  exclusive = false,
  autoDelete= false,
  arguments = {
    "x-dead-letter-exchange": "ANALIZADOR_DLX",  // mensajes rechazados
    "x-message-ttl": 86400000                     // 24h máximo en cola
  }
)

canal.BasicQos(prefetchCount = config.maxThreads)

consumer.Received += OnMensajeRecibido
canal.BasicConsume(config.nombreCola, autoAck=false, consumer)

log("Worker listo")
EsperarSiempre()


═══════════════════════════════════════════════════════════════
 RECEPCIÓN DE MENSAJE
═══════════════════════════════════════════════════════════════

función OnMensajeRecibido(deliveryTag, bodyBytes):

  msg = JSON.Parse(bodyBytes)

  // Validar campos mínimos
  si msg.jwtToken vacío O msg.analizadorId <= 0 O msg.peticion.tipoAnalizador vacío:
    BasicNack(deliveryTag, requeue=false)
    retornar

  Thread.Start(() =>

    ┌─────────────────────────────────────────────────────┐
    │ POR CADA MENSAJE (en su propio thread)              │
    │                                                     │
    │  1. ws = CrearWSConJWT(msg.jwtToken,                │
    │                        msg.videomHost,              │
    │                        msg.videomPath)              │
    │                                                     │
    │  2. ws.CicloMonitorizacion(                         │
    │        msg.analizadorId, "ANALIZANDO",              │
    │        msg.peticion.petid, "ENCURSO", "", null)     │
    │                                                     │
    │  3. [estado, error, result] =                       │
    │        Dispatch(msg.peticion.tipoAnalizador,        │
    │                 ws, msg.peticion, deliveryTag, msg) │
    │                                                     │
    │  4. si (resultado != ASINCRONO_PENDIENTE):          │
    │       ws.FinalizarExtraccionAnalizadores2(          │
    │         petid, analizadorId, estado, error, result) │
    │       BasicAck(deliveryTag)                         │
    │                                                     │
    └─────────────────────────────────────────────────────┘
  )


═══════════════════════════════════════════════════════════════
 DISPATCH (simplificado)
═══════════════════════════════════════════════════════════════

función Dispatch(tipo, ws, peticion, deliveryTag, msg):

  según tipo:
    "OCR SD"                   → TaskVideoma_OCR(...)
    "OCR"                      → TaskVideoma_OCR(...)
    "ALPR"                     → TaskVideoma_ALPR(...)
    "QUALITY CONTROL"          → TaskVideoma_QC(...)
    "VIDEO ANALYTICS"          → TaskVideoma_VideoAnalytics(...)
    "SPEAKERID NT"             → TaskVideoma_SpeakerId(...)
    "MUFIN" / "AUDIOFINGERPRINT" → TaskVideoma_MufinAudioFingerPrint(...)
    "SUBTITULOS"               → TaskVideoma_Subtitulos(...)
    "WORD SPOTTING SD"         → TaskVideoma_WordSpotting(...)
    "FACE RECOGNITION*"        → TaskVideoma_FaceRecognition(...)
    "TRANSCRIPCIONES VOSK"     → TaskVideoma_Vosk(...)
    "TRANSCRIPCIONES WHISPER"  → TaskVideoma_Whisper(...)
    "TRANSCRIPCIONES WHISPERX" → TaskVideoma_WhisperX(...)
    "TRANSLATE"                → TaskVideoma_Translate(...)
    "SUMARIZACION"             → TaskVideoma_Sumarize(...)
    "MEDIA DESCRIPTION"        → TaskVideoma_MediaDescription(...)

    // Asíncronos: el BasicAck lo hará el timer cuando termine el motor externo
    "TRANSCRIPCIONES SPMATIC"
    "TRANSCRIPCIONES VOCAPIA"
    "TRANSCRIPCIONES NUANCE"
    "SYMBOL ANALYSIS"
    "TEXT ANALYSIS"            → Dispatch_Asincrono(...)   // sin BasicAck inmediato


═══════════════════════════════════════════════════════════════
 TIMER DE SEGUIMIENTO ASÍNCRONO (único timer que subsiste)
═══════════════════════════════════════════════════════════════

Timer cada 40s:
  para cada entrada en listaAsync:
    estado = motor.ConsultarEstado(entrada.idExterno)

    si TERMINADO:
      ws.FinalizarExtraccionAnalizadores2(...)
      BasicAck(entrada.deliveryTag)
      listaAsync.Remove(entrada)

    si ERROR o TIMEOUT:
      ws.FinalizarExtraccionAnalizadores2(..., "ERRONEA", ...)
      BasicNack(entrada.deliveryTag, requeue=false)
      listaAsync.Remove(entrada)
```
---

## 9. El dispatch ya existe: qué hay que cambiar realmente

Una observación clave sobre el código actual: **el `if/else` de dispatch en `LanzarTareaThread` ya está completo y cubre todos los tipos**. El tipo del XML ejerce actualmente dos responsabilidades distintas que en el nuevo modelo se separan:

```
RESPONSABILIDAD 1 — en el timer (desaparece con RabbitMQ):
  tipoDeAnalizador = listaDatosServicio[i]->mNombre   ← del XML
  PeticionPendienteAnalizadores(tipoDeAnalizador, analizadorId)
  → filtra qué tipo de tarea pide a Videoma

RESPONSABILIDAD 2 — en LanzarTareaThread (se mantiene, solo cambia la fuente):
  if (listaDatosServicio[i]->mNombre->Contains("TRANSCRIPCIONES")) → TaskVideoma_Vosk / Whisper...
  if (listaDatosServicio[i]->mNombre->Contains("OCR"))             → TaskVideoma_OCR
  if (listaDatosServicio[i]->mNombre->Contains("SPEAKERID"))       → TaskVideoma_SpeakerId
  ...
```

Con RabbitMQ la **Responsabilidad 1 desaparece** (la tarea llega tipada en el mensaje). El único cambio estructural en `LanzarTareaThread` es sustituir de dónde lee el tipo:

```
// ANTES: el tipo viene del XML (porque el proceso arrancó configurado para un tipo fijo)
if (this->listaDatosServicio[i]->mNombre->Contains("TRANSCRIPCIONES"))

// DESPUÉS: el tipo viene del propio mensaje RabbitMQ
if (peticion->tipoAnalizador->Contains("TRANSCRIPCIONES"))
```

El cuerpo de cada rama del `if/else` no necesita cambios.

### Los tres campos del XML que hay que añadir al mensaje

Además del tipo, el dispatch usa actualmente tres campos del XML que eran implícitos porque el proceso solo recibía tareas del tipo configurado. Con el Worker universal, estos deben viajar en el mensaje:

| Campo XML                          | Dónde se usa en el dispatch                                              | Acción                              |
|------------------------------------|--------------------------------------------------------------------------|-------------------------------------|
| `listaDatosServicio[0]->mMotor`    | Subtipo de motor: `"VOSK"`, `"WHISPER"`, `"OCR SD"`, `"VAXTOR"`, etc.   | Añadir campo `motor` al mensaje     |
| `listaDatosServicio[0]->mUrlMotor` | URL del motor externo (Whisper, Face Recognition, Symbol Analysis)       | Añadir campo `urlMotor` al mensaje  |
| `listaDatosServicio[0]->mUsaSaas`  | Flag `"0"`/`"1"` para Face Recognition y Transcripciones con SaaS       | Añadir campo `usaSaas` al mensaje   |

El resto de parámetros específicos de motor (repositorios, umbrales, credenciales SaaS) ya se obtienen dentro de cada `TaskVideoma_XXX` mediante `ObtenerParametroAnalizador`, que usará el JWT heredado del mensaje sin ningún cambio.

### Estructura del mensaje con los campos añadidos

```json
{
  "jwtToken":     "eyJ...",
  "analizadorId": 42,
  "videomHost":   "videoma.empresa.com",
  "videomPath":   "videoma",
  "peticion": {
    "petid":            1234,
    "identificador":    5678,
    "localizacion":     "\\\\servidor\\media\\archivo.mp4",
    "localizacionhttp": "http://servidor/media/archivo.mp4",
    "idioma":           "es-ES",
    "tipo":             "VIDEO",
    "tipoAnalizador":   "TRANSCRIPCIONES VOSK",
    "motor":            "VOSK",
    "urlMotor":         "",
    "usaSaas":          "0"
  }
}
```

### Resumen real del esfuerzo de migración del Worker

| Cambio                                      | Dificultad | Detalle                                                                   |
|---------------------------------------------|------------|---------------------------------------------------------------------------|
| Sustituir fuente del tipo en el `if/else`   | Muy baja   | Buscar y reemplazar `listaDatosServicio[0]->mNombre` → `peticion->tipo*`  |
| Añadir consumer RabbitMQ                    | Media      | ~80 líneas nuevas, instanciar `RabbitMQ.Client`                           |
| Añadir `BasicAck`/`BasicNack` al finalizar  | Baja       | Dos líneas al cierre de cada rama del `if/else` de finalización           |
| Adaptar asíncronos (guardar `deliveryTag`)  | Media      | SPMATIC/NUANCE/SYMBOL: añadir `deliveryTag` a `listaIdsEnCurso`           |
| Eliminar `AltaAnalizador`, timer, `GetAuthToken` | Baja  | Borrar código, no añadir                                                  |
| Cambios en Videoma (publicar en RabbitMQ)   | Media      | El grueso del esfuerzo está en el lado de Videoma, no del Worker          |

---

## 10. Riesgos y puntos abiertos

### Expiración del JWT (pendiente de decisión)

El JWT que Videoma incluye en el mensaje debe tener **vida útil suficiente** para cubrir el análisis completo. Algunos analizadores pueden tardar horas (MUFIN, SPMATIC).

```
Opciones a definir con el equipo de Videoma:
  A) JWT de larga duración específico para el worker (p.ej. 24h, renovable)
     → Videoma genera un token especial "service token" al publicar en RabbitMQ
     → No es el JWT de sesión del usuario

  B) JWT de corta duración + mecanismo de refresco
     → El mensaje incluye también un "refresh token"
     → El Worker renueva el JWT antes de llamar a FinalizarExtraccionAnalizadores2
     → Añade complejidad al Worker

  C) JWT de sesión del usuario con expiración extendida automática
     → Videoma extiende la vida del token al crear la tarea

  RECOMENDACIÓN: opción A es la más simple y robusta para un worker de background.
```

### Garantía de entrega y reentrega

Si el Worker procesa la tarea pero falla antes de hacer `BasicAck`, RabbitMQ reentrega el mensaje al reiniciar. El Worker podría procesar la misma tarea dos veces.

```
Mitigación recomendada:
  → Antes de llamar a CicloMonitorizacion, consultar el estado actual de la tarea en Videoma
  → Si ya está en "FINALIZADA", hacer BasicAck y salir sin reanalizar

  función EsTareaYaFinalizada(ws, petid) → bool:
    tareas = ws.ListaTareasAnalizador(analizadorId, "FINALIZADA")
    retornar tareas.Any(t => t.peticionid == petid)
```

### Proxy por tarea vs. proxy global

Actualmente el proxy se lee una vez al arrancar. Con el Worker universal, el proxy podría venir en el mensaje o seguir siendo un parámetro global del Worker. Lo más simple es mantenerlo en el XML local del Worker.

### Mensajes en la cola al reiniciar Videoma

Si Videoma reinicia y los mensajes del RabbitMQ se habían publicado con `persistent = true` sobre una cola `durable`, los mensajes **no se pierden**. El Worker los procesará cuando Videoma vuelva a estar disponible. Esto es una mejora respecto al modelo actual donde las tareas quedaban en estado `ENCURSO` en la BD y había que limpiarlas manualmente.
