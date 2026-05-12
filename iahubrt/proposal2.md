# IAHUBRT — Propuesta de diseño

**Autor**: Eneko Rebollo · **Versión**: 2.0 · **Fecha**: 2026-05-12

---

## 1. Idea en una frase

**IAHUBRT** es un servicio nuevo, externo a `ia-hub`, cuyo propósito es **emitir notificaciones MQTT a subscribers** cada vez que los workers de ia-hub detectan algo en un RTSP. Es el "broadcaster" del sistema.

1. Recibe por REST el alta de un canal (fuente RTSP + profile).
2. Abre una conexión SSE contra ia-hub para esa sesión y la mantiene viva mientras el canal esté `running`.
3. Normaliza cada evento que llega por el SSE (independientemente del modelo).
4. Publica por MQTT a los subscribers.

`ia-hub` gana dos workers nuevos — **`vaxtor-worker`** y **`innovatrix-worker`** — siguiendo el contrato estándar (`.claude/rules/adding-a-worker.md`). IAHUBRT no sabe nada de Vaxtor/Innovatrix; sólo conoce los endpoints de ia-hub.

> **Nomenclatura.** ia-hub usa **RabbitMQ (AMQP)** *internamente* para encolar tasks hacia los workers — no es el canal de respuesta a clientes. La respuesta a clientes la da ia-hub por **HTTP REST**, con SSE para streams (mismo patrón que el `/api/rag/stream` ya existente en `dispatcher.py:1762`). **MQTT** lo aporta IAHUBRT desde fuera para hablar con los subscribers externos.

---

## 2. Diagrama

```
                          ┌─────────────────────┐
                          │     Subscribers     │
                          └──────────▲──────────┘
                                     │ MQTT (pub/sub, fan-out)
                                     │
┌────────────────────────────────────┴─────────────────┐
│  IAHUBRT (servicio nuevo, repo propio)               │
│                                                      │
│   REST API ─────► Channel Manager (state machine)    │
│   /sources, /channels                                │
│       │                                              │
│       ▼                                              │
│   SSE consumer  ◄── POST /api/task/<pool>/stream-start
│       │              (text/event-stream)             │
│       ▼                                              │
│   Envelope normalizer ──► [Mosquitto]                │
└──────────────────────────────────────────────────────┘
                  │
                  │ HTTP (Dispatcher API)
                  ▼
         ┌─────────────────────────────────────────┐
         │   ia-hub Dispatcher (existente)         │
         │                                         │
         │   workers:                              │
         │     • vaxtor-worker      (new)          │
         │     • innovatrix-worker  (new)          │
         │     • face / ocr / …                    │
         └────────────────┬────────────────────────┘
                          │ RTSP (vía path interno de MediaMTX en IAHUBRT — ver §3)
                          ▼
                    [cámaras IP]
```

**Lo que vive en cada lado**:

| En `ia-hub` (cambios) | En `IAHUBRT` (nuevo) |
|---|---|
| `vaxtor-worker` — supervisor del ejecutable Vaxtor, expone `stream-start` con SSE | REST API pública (`/api/sources`, `/api/channels`) |
| `innovatrix-worker` — controla Innovatrix por REST + consume su pub/sub ZMQ, expone `stream-start` con SSE | State machine de canales (waiting/running/stopped/error) |
| Nuevas rutas en Dispatcher: `POST /api/task/{vaxtor,innovatrix}/stream-start` | Cliente HTTP que abre SSE contra el Dispatcher de ia-hub |
| Pools en `concurrency.json`: `vaxtor_pool`, `innovatrix_pool` | Normalizer + publicador MQTT (Mosquitto) |
| Concurrencia v2 ya existente cubre los nuevos pools | MediaMTX sidecar + Path manager (ingest RTSP — ver §3) + métricas Prometheus + dashboard Grafana |

> **Implicación**: el control de capacidad de los modelos lo hace **ia-hub** (concurrency v2). IAHUBRT recibe el equivalente a "no hay slot" y deja el canal en `waiting`. Cero reimplementación de semáforos.

---

## 3. Ingest de streams — MediaMTX

IAHUBRT despliega **MediaMTX** como sidecar — un media server FOSS (Apache 2.0, binario Go, ~30 MB de RAM en idle) cuya única función en este middleware es **encapsular la cámara**. En lugar de pasarle a los workers de `ia-hub` la URL RTSP cruda de la cámara — con todos sus problemas (red distinta, NAT, credenciales acopladas) — IAHUBRT registra el feed en MediaMTX y le entrega a los workers una URL **interna y estable**. Lo que pase entre cámara y MediaMTX es problema de IAHUBRT; hacia los workers la forma es siempre la misma: `rtsp://mediamtx:8554/source-{uuid}`. El codec por defecto es passthrough del original — ver §3.4 para la política completa y la nota sobre transcoding opcional a H.264.

```
   [cámara IP]
        │ RTSP / RTMP / SRT / WHIP
        ▼
┌──────────────────────────────────────┐
│ IAHUBRT                              │
│                                      │       ┌──────────────────────┐
│   ┌─────────────────────────┐        │       │ ia-hub Dispatcher    │
│   │ MediaMTX (sidecar)      │────────┼──────►│                      │
│   │                         │  RTSP  │       │   vaxtor-worker      │
│   │ rtsp://mediamtx:8554/   │        │       │   innovatrix-worker  │
│   │   source-{uuid}         │        │       │                      │
│   │ (codec passthrough;     │        │       └──────────────────────┘
│   │  ver §3.4)              │        │
│   └────────────▲────────────┘        │
│                │ REST mgmt API       │
│   ┌────────────┴────────────┐        │
│   │ Path manager            │        │
│   │ (alta/baja paths)       │        │
│   └────────────▲────────────┘        │
│                │ reacciona al        │
│                │ estado del channel  │
│   ┌────────────┴────────────┐        │
│   │ Channel state machine   │        │
│   └─────────────────────────┘        │
└──────────────────────────────────────┘
```

> **Resumen mental**: MediaMTX es un proxy/relay RTSP gestionado por IAHUBRT. Los workers de `ia-hub` siempre reciben una URL del mismo formato, independientemente de la red o las particularidades de la cámara real. Por defecto los frames se entregan en el codec original (passthrough/remuxing); transcoding a H.264 queda como opción debatible para cuando se confirme qué acepta cada analizador (ver §3.4). Si iahub o cualquier otro consumidor externo quisiera en el futuro consumir el feed por otros protocolos (HLS, WebRTC...), MediaMTX lo soporta out-of-the-box; pero eso queda fuera del alcance de este middleware — IAHUBRT sólo se compromete con el RTSP interno hacia los workers.

### 3.1 Ciclo de vida del path

El path en MediaMTX es la representación interna del feed. Su ciclo de vida lo gestiona IAHUBRT vía la **REST API de MediaMTX** (`POST /v3/config/paths/add/{name}`, `delete/{name}`, `GET /v3/paths/get/{name}`) y depende del estado de los channels asociados al source:

| Trigger | Acción IAHUBRT | Acción MediaMTX | Resultado |
|---|---|---|---|
| `POST /api/sources { url, ingest_mode }` | Persiste el Source en BD | (ninguna) | Source registrado pero sin consumir recursos |
| Primer channel para un source pasa a `running` | `POST /v3/config/paths/add/source-{uuid}` con `source = <url externa>` (pull) o config de `publisher` (push) | Path activo; MediaMTX recibe el feed | Stream operativo, consumible por todos los protocolos |
| Channel pasa a `stopped` siendo el último activo del source | `POST /v3/config/paths/delete/source-{uuid}` | Path eliminado | Cero consumo de CPU/ancho de banda por feeds huérfanos |
| Channel pasa a `running` con el path ya desregistrado | Reinscribe el path (mismo `add`) | Path activo de nuevo | Reactivación limpia |
| `DELETE /api/sources/{id}` | 409 si quedan channels asociados | (sin cambios) | El usuario primero detiene/borra los channels |
| `POST /api/sources/{id}/probe` | Consulta `GET /v3/paths/get/...` si el path existe; si no, registro temporal + probe + desregistro | (path temporal si hace falta) | Devuelve codec/fps/resolución reales que ve MediaMTX |

> **Regla de oro**: *un path vive mientras tenga al menos un channel activo*. Esto evita feeds huérfanos consumiendo recursos y simplifica el debugging — si un path está activo, alguien lo está procesando. Reactivar es barato (`POST` a MediaMTX, latencia ~100ms hasta que el feed esté disponible).

### 3.2 Modos de ingest — cómo llega el stream a MediaMTX

El campo `ingest_mode` del `Source` declara la dirección del flujo. Tres escenarios cubren prácticamente todos los casos reales:

#### Modo A — Pull (default)

MediaMTX abre la conexión saliente hacia el RTSP de la cámara y mantiene el feed activo.

```
[cámara IP] ◄── RTSP pull ── [MediaMTX en IAHUBRT]
```

**Cuándo usarlo**: la cámara está en una red alcanzable desde el host de IAHUBRT (misma LAN, VPN site-to-site, peering VPC).

**Pros**: cero infraestructura adicional. Modo nativo de la mayoría de cámaras IP. Plug-and-play.

**Contras**: el host de IAHUBRT debe poder llegar a la cámara (puerto 554 abierto, red routable). No funciona si la cámara está detrás de NAT sin VPN.

#### Modo B — Push

La cámara — o un agente edge corriendo dentro de su red — abre la conexión hacia IAHUBRT y publica el stream contra un path pre-acordado.

```
[cámara IP] ── push (RTSP/RTMP/SRT/WHIP) ──► [MediaMTX en IAHUBRT]
   o
[agente ffmpeg en red de la cámara] ── pull cámara ── push hacia IAHUBRT
```

**Cuándo usarlo**: la cámara está detrás de NAT o en una red sin inbound desde IAHUBRT (instalaciones de cliente sin VPN, dispositivos móviles, drones, cámaras en redes de socios).

**Protocolos soportados por MediaMTX**: **RTSP** (con `ANNOUNCE`), **RTMP**, **SRT** (recomendado sobre internet por su resiliencia frente a pérdida de paquetes), **WHIP** (WebRTC).

**Pros**: NAT traversal automático; sólo se necesita salida del lado de la cámara.

**Contras**: pocas cámaras IP soportan modos push nativos — normalmente requiere un agente intermedio (ffmpeg, mediamtx-edge, rtsp-simple-server en modo edge). SRT/WHIP necesitan autenticación con token.

#### Modo C — Híbrido

Una misma instancia de IAHUBRT puede combinar paths en pull y en push. Una instalación realista en cliente final puede tener cámaras locales (pull, misma LAN) + cámaras de sucursal (push vía SRT atravesando internet). El modo se declara por `Source`; los workers de `ia-hub` que consumen el RTSP interno **no se enteran del modo** — abren la URL y leen frames, igual que con una cámara local.

### 3.3 URL de consumo para los workers

Sea cual sea el modo de ingest (§3.2), IAHUBRT siempre entrega a `ia-hub` la misma forma de URL al arrancar el `stream-start` de un channel:

```
rtsp://mediamtx:8554/source-{source_uuid}
```

Propiedades garantizadas de esa URL:

| Propiedad | Valor | Notas |
|---|---|---|
| Esquema | `rtsp://` | Lo que ya saben consumir el binario Vaxtor y el cliente RTSP de Innovatrix |
| Host | `mediamtx` (DNS interno del compose de IAHUBRT) | Si `ia-hub` corre en otro host, en el `stream-start` se sustituye por la IP/DNS del host de IAHUBRT |
| Puerto | `8554` | Puerto estándar de RTSP en MediaMTX |
| Path | `source-{source_uuid}` | UUID del `Source` en BD de IAHUBRT; estable mientras el Source exista |
| Codec | Passthrough del original | Por defecto remuxing sin re-encodear (ver §3.4). Transcoding a H.264 queda como opción si lo pide la integración con Vaxtor/Innovatrix |
| Credenciales | Ninguna en la URL en fase 1 | Mismo perímetro de confianza que el resto del compose. Si se expone fuera, se activa `authMethod` en MediaMTX con `password_file` |

Los workers de `ia-hub` **no se enteran** de nada del modo (pull/push), de la red de la cámara real, ni del codec original. Para ellos sólo hay un `rtsp://` que abrir y leer frames — el mismo contrato del `vaxtor-worker` / `innovatrix-worker` descrito en `.claude/rules/adding-a-worker.md`, sin cambios.

### 3.4 Política de codec — remuxing por defecto, transcoding opcional

Convención del middleware en fase 1: **MediaMTX hace passthrough del codec que mande la cámara, sin re-encodear**. La URL interna que IAHUBRT entrega a los workers sirve los frames tal cual los emitió la cámara, sólo cambiando el envoltorio del contenedor (remuxing) cuando hace falta.

Razón: Vaxtor e Innovatrix aceptan varios codecs RTSP comunes — pendiente confirmar con cada fabricante el listado exacto, pero el supuesto de trabajo es que **H.264 lo aceptan con seguridad y otros codecs (H.265, MJPEG) probablemente también**. Forzar siempre transcoding a H.264 sería un consumo de CPU/GPU innecesario para el caso normal.

- **Coste de remuxing**: despreciable. MediaMTX sólo reenvasa los bytes de la cámara para el cliente RTSP del worker.
- **Recomendación operativa**: configurar las cámaras en H.264 cuando se pueda elegir. Es el codec más interoperable y el de menos sorpresas.

> **Nota de posibilidad — transcoding a H.264 (a debatir durante la implementación)**. Si al integrar Vaxtor o Innovatrix se descubre que un codec concreto no es aceptado por alguno de los dos (por ejemplo, H.265 sí pero AV1 no), MediaMTX soporta transcoding al ingest mediante un `runOnReady` que arranca ffmpeg para ese path y republica en el codec compatible. Es un mecanismo conocido y barato de añadir; queda **anotado como opción, no activado por defecto**. La decisión final (qué codecs forzar, si hace falta GPU para NVENC/QuickSync, etc.) se cierra cuando se conozca el listado real de codecs que aceptan Vaxtor e Innovatrix.

---

## 4. Flujo principal — suscripción a un RTSP en vivo

Mismo patrón que el `/api/rag/stream` ya existente en ia-hub: la respuesta es un `text/event-stream` que se mantiene abierto durante toda la vida de la sesión, y el slot de concurrencia queda reservado mientras el stream esté vivo. El flujo end-to-end se ilustra con Vaxtor en §4.1; la variante Innovatrix (idéntica desde fuera, distinta sólo en cómo el worker produce los eventos SSE) se detalla en §4.2.

### 4.1 Flujo end-to-end (ejemplo Vaxtor)

```
1. POST /api/sources              → Source { id, url: "rtsp://10.1.1.1/…" }
2. POST /api/channels             → Channel { source_id, profile: "vaxtor:lpr-eu" }
3. PATCH /api/channels/{id} desired_state=running
4. IAHUBRT registra el path en MediaMTX si todavía no existe
   (POST /v3/config/paths/add/source-{uuid}). Espera a que esté listo.
5. IAHUBRT abre una conexión SSE contra ia-hub:
       POST /api/task/vaxtor/stream-start
       { rtsp_url: "rtsp://mediamtx:8554/source-{uuid}", channel_id }
       Accept: text/event-stream
6. Dispatcher adquiere slot de vaxtor_pool (concurrency v2, igual que
   acquire_rag_slot()) y proxea el stream del worker.
7. vaxtor-worker abre el RTSP del path de MediaMTX y emite un evento
   SSE por cada detección:
       event: detection
       data:  { event_type, payload_específico, timestamp }
8. IAHUBRT lee cada evento del stream, lo normaliza en el envelope
   común y publica:
       mqtt://iahubrt/channel/{channel_id}/events
9. Para parar: PATCH desired_state=stopped → IAHUBRT cierra la
   conexión SSE → worker detecta cierre → libera el RTSP del path
   de MediaMTX → libera slot. Si es el último channel activo del
   source, IAHUBRT también desregistra el path en MediaMTX.
```

> **Por qué SSE y no inventar un exchange en RabbitMQ.** El SSE ya está implementado y probado en producción para `/api/rag/stream` (ver `dispatcher.py:1762`). Reusamos el mismo patrón con el mismo modelo de slot (gate mantenido durante toda la conexión). Cero infra nueva en ia-hub.
>
> **Si IAHUBRT se cae** mientras un canal está activo, la conexión SSE se rompe, el worker libera el slot y el RTSP. Al reiniciar, IAHUBRT reconcilia: para cada canal con `desired_state=running`, abre un nuevo SSE → arranca sesión limpia. No hay buffer de eventos perdidos durante la caída, pero tampoco hace falta: lo que importa es que el stream esté vivo, no recuperar la detección de hace 3 segundos.

### 4.2 Variante Innovatrix — traducción ZMQ → SSE

Los pasos 1–9 de §4.1 son idénticos cambiando `vaxtor` por `innovatrix` en las rutas (`/api/task/innovatrix/stream-start`) y el nombre del pool (`innovatrix_pool`). La diferencia real vive **dentro del worker, en el paso 7** — cómo `innovatrix-worker` produce los eventos SSE que IAHUBRT lee. Mientras `vaxtor-worker` lee de un subproceso, `innovatrix-worker` traduce un pub/sub ZeroMQ a SSE.

Detalle del paso 7 en Innovatrix:

```
7a. Al recibir POST /api/innovatrix/stream-start, el worker hace
    POST /watch a la API REST de Innovatrix con la rtsp_url y task_id.
    Innovatrix arranca el análisis sobre esa cámara y empieza a
    publicar detecciones en su socket ZeroMQ PUB.

7b. El worker abre un socket ZeroMQ SUB (pyzmq, modo async) conectado
    al endpoint del PUB de Innovatrix, y filtra por task_id para
    recibir sólo los eventos del channel actual.

7c. En un bucle async (`async for msg in sub`), por cada mensaje
    recibido emite un evento SSE en el mismo formato que vaxtor-worker:
        event: detection
        data:  { event_type, payload, timestamp }
    Cada yield del async generator se escribe al StreamingResponse al
    instante, sin buffering — IAHUBRT recibe el evento milisegundos
    después de que Innovatrix lo publique.

7d. Al cerrar IAHUBRT la conexión SSE (asyncio.CancelledError en el
    generador), el worker corre limpieza simétrica:
      - POST /unwatch a Innovatrix para que deje de procesar la cámara.
      - sub.close() + ctx.term() para liberar el socket ZMQ.
    El slot del pool en el Dispatcher se libera automáticamente al
    cerrarse el StreamingResponse.
```

> Desde fuera, IAHUBRT no se entera de la diferencia: lee los mismos `event: detection` por SSE y los normaliza al mismo envelope MQTT, vengan de un subproceso (Vaxtor) o de un socket ZMQ (Innovatrix). Esta uniformidad de cara al exterior es lo que permite que el contrato del worker definido en `.claude/rules/adding-a-worker.md` sirva para ambos sin variantes.

> **Dependencia clave**: `pyzmq` con su API async (`zmq.asyncio`). Es la forma estándar de integrar ZeroMQ en un servidor asyncio/FastAPI sin bloquear el event loop. No se necesita ningún proceso o thread adicional — un socket SUB por sesión activa basta.

---

## 5. Modelo de datos (IAHUBRT)

Dos tablas:

```
Source   { id, name, url, ingest_mode, credentials_ref?, metadata }
Channel  { id, source_id, profile, state, desired_state, ia_hub_task_id?, last_error, stats }
```

`ingest_mode` ∈ `{pull, push}`, default `pull`. Determina cómo IAHUBRT registra el path en MediaMTX (ver §3.2). En `pull`, IAHUBRT le dice a MediaMTX que tire de `url`. En `push`, IAHUBRT prepara el path para que el publisher externo (la cámara o un agente edge) suba el stream contra él; en este caso `url` se interpreta como la URL pública que entregaremos al publisher para que se conecte.

`profile` es un string opaco como `"vaxtor:lpr-eu"` o `"innovatrix:people-counter"`. Lo entiende ia-hub; IAHUBRT solo lo enruta. Si más adelante queremos CRUD de profiles, lo añadimos — en fase 1 son strings que ia-hub valida.

**Estados del canal**: `waiting → running → stopped` | `error`. Separamos `state` y `desired_state` para que un reaper pueda converger el sistema sin lógica adhoc en cada endpoint.

### Motor de base de datos

**SQLite por defecto, Postgres opcional**. El dataset es pequeño (sources + channels, escritura puntual), no justifica un servicio de DB separado en fase 1. Configurable por env:

```
# Default — fichero local persistido en volumen Docker
DATABASE_URL=sqlite:////data/iahubrt.db

# Opcional — para HA o despliegues multi-replica
DATABASE_URL=postgresql+psycopg://user:pass@host:5432/iahubrt
```

Requiere un ORM con soporte de migraciones que cubra los dos motores. Compose monta `/data` como volumen named para que el `.db` sobreviva reinicios. Pasar a Postgres más adelante es cambiar la URL de conexión y correr las migraciones — sin tocar código de modelo. Ver §12 para librerías concretas según el lenguaje elegido.

---

## 6. API REST

```
# Sources (fuentes RTSP)
POST   /api/sources                   { name, url, ingest_mode?, credentials?, metadata? }
GET    /api/sources
DELETE /api/sources/{id}              409 si tiene channels asociados
POST   /api/sources/{id}/probe        ping RTSP (vía MediaMTX); devuelve codec/fps/resolución

# Channels (suscripción live)
POST   /api/channels                  { source_id, profile }
GET    /api/channels
PATCH  /api/channels/{id}             { desired_state }
DELETE /api/channels/{id}

# Operación
GET    /health
GET    /metrics
```

Mismo formato de errores que ia-hub (RFC 7807, `application/problem+json` con `code` taxonomizado). IAHUBRT implementa el mismo contrato; ver §12 para la librería/módulo a usar en cada lenguaje.

---

## 7. MQTT — topics y payload

### Topics

```
iahubrt/system/status                       heartbeat (retain)
iahubrt/system/events                       eventos atípicos (pool down, reinicio)
iahubrt/channel/{channel_id}/state          estado actual del canal (retain)
iahubrt/channel/{channel_id}/events         detecciones del canal
```

`retain: true` en los topics de `…/state` para que un subscriber nuevo conozca el estado actual sin esperar a la próxima transición.

### Payload normalizado (envelope común)

```json
{
  "envelope_version": "1.0",
  "event_id":   "uuid",
  "event_type": "detection | tracking_update | state_change | error | stats",
  "channel_id": "uuid",
  "source_id":  "uuid",
  "profile":    "vaxtor:lpr-eu",
  "task_id":    "uuid (correlación con ia-hub)",
  "timestamp":  "2026-05-11T12:33:01.234Z",
  "severity":   "info | warn | error",
  "payload":    { /* específico del worker — bbox, plate, confidence… */ }
}
```

El envelope lo construye **IAHUBRT** a partir del evento SSE que recibe del worker. El worker solo manda `event_type` + `payload`; el resto lo enriquece IAHUBRT con lo que sabe del canal. Así el payload es **idéntico independientemente del worker** — requisito original cubierto.

---

## 8. Observabilidad

Métricas Prometheus mínimas (todas con prefijo `iahubrt_`):

```
iahubrt_channels_total{state}                      gauge
iahubrt_channel_state_transitions_total{from,to}   counter
iahubrt_events_published_total{event_type}         counter
iahubrt_events_dropped_total{reason}               counter
iahubrt_event_publish_duration_seconds             histogram
iahubrt_iahub_call_errors_total{code}              counter
iahubrt_api_request_duration_seconds{route}        histogram
iahubrt_sse_reconnects_total{pool}                 counter
```

La saturación de pools la expone **ia-hub** directamente (ya tiene métricas para eso). El dashboard de IAHUBRT enlaza al de ia-hub para esa fila.

**Dashboard inicial**:
- Fila 1 — Estado global: # canales por estado, eventos/s, reconexiones SSE.
- Fila 2 — Salud: errores hablando con ia-hub, eventos dropeados, latencia de publicación.
- Fila 3 — API: requests/s por ruta, latencia p95, 5xx rate.
- Link directo al dashboard de pools/concurrencia de ia-hub.

### Historial de eventos en OpenSearch

Cada evento publicado a MQTT se duplica como **log estructurado JSON** que Vector (la pipeline de logs ya existente en ia-hub) enruta al índice `iahubrt-events-*` en OpenSearch.

- Reusa la infra Vector + OpenSearch — sin código de cliente OpenSearch en IAHUBRT.
- Index template propio en `config/opensearch/iahubrt-events.json`, con keyword en `channel_id`, `event_type`, `pool`, `severity` para queries rápidas en Discover.
- Política ISM propia (propuesta inicial: 30 días, separada de `iahub-logs-*` para poder afinarla independientemente).
- **Recuperación de datos**: vía Dashboards Discover, filtrando por `channel_id:"<uuid>" AND @timestamp:[...]`. Sin endpoint REST de query en fase 1 — si aparece un cliente que lo necesite, añadimos `GET /api/channels/{id}/events?from=...&to=...&limit=...` como wrapper sobre OpenSearch (≈ 30 líneas).
- **Recuperación para subscribers offline**: ortogonal. Si hace falta, se activa `persistence true` + `QoS 1` + sesiones persistentes en Mosquitto. Esto resuelve "subscriber se reconecta y le entregan lo que perdió" sin tocar OpenSearch.

---

## 9. Qué cambia en `ia-hub`

Dos workers nuevos siguiendo el contrato de `.claude/rules/adding-a-worker.md`:

### `vaxtor-worker`
Un único endpoint:
- `POST /api/vaxtor/stream-start` — devuelve `text/event-stream`. Abre el RTSP y emite un evento SSE por cada detección. Cuando IAHUBRT cierra la conexión, libera el RTSP. Sigue el patrón del `embedding-worker` para `/api/rag/stream`.

Pool: `vaxtor_pool` con `limit: N` en `concurrency.json`. El slot se mantiene durante toda la conexión SSE.

**Integración interna**: Vaxtor se distribuye como un **ejecutable** al que se le pasa la URL RTSP. El worker actúa de supervisor de subproceso: lanza el binario en `stream-start`, captura su output (stdout / fichero de eventos — a confirmar en implementación, ver §10), parsea cada detección y la emite por el SSE. Al cerrarse la conexión SSE, el worker envía `SIGTERM` al subproceso y espera con timeout. La librería estándar (`asyncio.create_subprocess_exec`) basta.

### `innovatrix-worker`
Un único endpoint `POST /api/innovatrix/stream-start` con la misma semántica que Vaxtor de cara a IAHUBRT.

**Integración interna**: Innovatrix se controla en dos pasos:
1. **REST a la API de Innovatrix** para indicarle "vigila esta cámara" (el worker hace el `POST` cuando recibe `stream-start`).
2. **Suscripción ZeroMQ** a la publicación de eventos de Innovatrix (dependencia: `pyzmq`). El worker mantiene el socket SUB abierto y reenvía cada evento por el SSE.

Al cerrar el SSE, el worker hace el `POST` de "deja de vigilar esta cámara" a Innovatrix y cierra el socket ZMQ.

> Desde fuera, los dos workers son idénticos: URL RTSP entra, eventos SSE salen, el slot del pool se mantiene durante la sesión. La diferencia (subproceso vs API REST + ZMQ) queda contenida dentro del worker.

### Cambios en Dispatcher
- Constantes `VAXTOR_WORKER_URL` y `INNOVATRIX_WORKER_URL`.
- Rutas `POST /api/task/vaxtor/stream-start` y `/api/task/innovatrix/stream-start`: copian la lógica de `proxy_rag_stream` (adquieren slot, hacen proxy con `StreamingResponse`, sueltan slot al cierre).
- Pools `vaxtor_pool` e `innovatrix_pool` en `concurrency.json`, con su `limit`.

> No hay queues de RabbitMQ nuevas: los `stream-start` son streaming HTTP puro, no pasan por encolado.

> **Lo que NO cambia**: si el pool está saturado, ia-hub bloquea la adquisición del slot y devuelve un código de saturación; IAHUBRT lo ve como "no se pudo arrancar" y mantiene el canal en `waiting` reintentando.

---

## 10. Decisiones cerradas y pendientes

### Cerradas

- ✅ **MQTT es el único canal hacia subscribers.** Si es MQTT, es MQTT.
- ✅ **Comunicación IAHUBRT ↔ ia-hub vía SSE.** Mismo patrón que `/api/rag/stream`. El slot del pool se mantiene durante toda la conexión y se libera al cierre.
- ✅ **Alcance fase 1: sólo RTSP en vivo.** Vídeo suelto descartado — para pruebas usamos un RTSP de test.
- ✅ **Transporte de eventos de Innovatrix: ZeroMQ.** Dependencia del worker: `pyzmq`.
- ✅ **Broker MQTT: Mosquitto.** Imagen oficial en compose, persistencia en volumen, sin TLS en fase 1 (LAN privada).
- ✅ **Auth fase 1.** `HTTPBasic` en la REST de IAHUBRT (single user/password por env). Mosquitto con `password_file` usando las mismas credenciales. La llamada IAHUBRT → ia-hub Dispatcher va sin auth (red Docker interna, mismo perímetro de confianza que hoy). Fase 2 → migración a JWT integrado con `iahubcms`.
- ✅ **Persistencia de eventos: lite, vía OpenSearch.** Cada evento publicado a MQTT se duplica como log estructurado que Vector enruta al índice `iahubrt-events-*` en OpenSearch (reusa la pipeline ya existente). Sin tabla de eventos en Postgres, sin endpoint REST de query en fase 1. Retention 30 días por ISM. Cobertura para los dos casos: debug operacional (Discover) y recuperación de datos perdidos (query por `channel_id` y rango temporal). Si más adelante hace falta entregar mensajes a subscribers que estaban offline, se activa `QoS 1` + sesiones persistentes en Mosquitto sin tocar OpenSearch.
- ✅ **Ingest de streams centralizado en MediaMTX dentro de IAHUBRT** (sidecar Go, Apache 2.0). IAHUBRT gestiona el ciclo de vida de los paths vía la REST API de MediaMTX y entrega a `ia-hub` una URL RTSP interna estable (`rtsp://mediamtx:8554/source-{uuid}`) en lugar de la URL cruda de la cámara. Detalles en §3.
- ✅ **Política de path: vive mientras haya ≥1 channel activo para el source.** El último `running → stopped` desregistra el path en MediaMTX (`POST /v3/config/paths/delete/...`); un nuevo `stopped → running` lo registra de nuevo. Sin feeds huérfanos consumiendo CPU/ancho de banda.
- ✅ **Tres modos de ingest soportados**: `pull` (default; MediaMTX → cámara), `push` (cámara/agente edge → MediaMTX, vía RTSP/RTMP/SRT/WHIP), `híbrido` (mezcla por source). Se declara en el campo `Source.ingest_mode`. Los consumidores aguas abajo no se enteran del modo.
- ✅ **Remuxing por defecto, transcoding opcional.** MediaMTX hace passthrough del codec original (coste CPU despreciable). H.264 es el codec recomendado al configurar la cámara cuando se pueda elegir. Transcoding a H.264 al ingest queda como mecanismo disponible (ffmpeg vía `runOnReady`) por si la integración con Vaxtor/Innovatrix demuestra que hace falta forzar un codec concreto — a debatir durante la implementación, no activado por defecto.

### Pendientes

1. **Formato de salida del ejecutable de Vaxtor.** ¿stdout línea a línea? ¿fichero de eventos rotatorio? Define cómo parsea el `vaxtor-worker`. Verificar con la documentación del fabricante (deberes externos).

---

## 11. Propuesta de stack para IAHUBRT (Python y C#)

> El cuerpo del documento es agnóstico de lenguaje. Esta sección lista librerías concretas para los dos lenguajes candidatos; la decisión final se toma al iniciar el desarrollo.
>
> **Importante**: los dos workers que se añaden al repo `ia-hub` (`vaxtor-worker` e `innovatrix-worker`) van **obligatoriamente en Python**, porque siguen el contrato definido en `.claude/rules/adding-a-worker.md` y el resto del repo es Python. **Sólo IAHUBRT tiene libertad de elección de lenguaje.**

### Opción A — Python

| Pieza               | Librería / herramienta                                     | Notas                                                         |
| ------------------- | ---------------------------------------------------------- | ------------------------------------------------------------- |
| HTTP server         | **FastAPI**                                                | Async, OpenAPI automático, mismo stack que ia-hub.            |
| ORM + migraciones   | **SQLAlchemy 2.x** (async) + **Alembic**                   | Usar `batch_alter_table` para compatibilidad SQLite.          |
| Cliente HTTP / SSE  | **httpx**                                                  | Soporta streaming async (`aiter_bytes`) para consumir el SSE. |
| Cliente MQTT        | **asyncio-mqtt** (wrapper async de paho-mqtt)              | API limpia integrada en asyncio.                              |
| Validación          | **Pydantic v2**                                            | Viene con FastAPI.                                            |
| Config              | **pydantic-settings**                                      | Lee env + `.env` con tipos validados.                         |
| Métricas Prometheus | **prometheus-client**                                      | `/metrics` endpoint.                                          |
| Logging JSON        | Reusar el vendoreado `services/_shared/logging_setup.py`   | Misma pipeline Vector→OpenSearch que ia-hub.                  |
| Errores RFC 7807    | Reusar el vendoreado `services/_shared/problem_details.py` | Mismo catálogo de `code`.                                     |
| Linting / formato   | **ruff**                                                   | Coherencia con ia-hub.                                        |
| Tests               | **pytest** + **httpx.AsyncClient**                         | Patrón ya usado en ia-hub.                                    |

### Opción B — C# (.NET 8 LTS)

| Pieza | Librería / herramienta | Notas |
|---|---|---|
| HTTP server | **ASP.NET Core** Minimal APIs (o Controllers) | OpenAPI con Swashbuckle o NSwag. |
| ORM + migraciones | **Entity Framework Core** | EF migrations cubre SQLite y Postgres. |
| Cliente HTTP / SSE | **HttpClient** con `HttpCompletionOption.ResponseHeadersRead` + parser SSE, o **LaunchDarkly.EventSource** | .NET 9 traerá `System.Net.ServerSentEvents` nativo. |
| Cliente MQTT | **MQTTnet** | De-facto standard en .NET, activamente mantenido. |
| Validación | **FluentValidation** o `DataAnnotations` | |
| Config | **Microsoft.Extensions.Configuration** con `appsettings.json` + env | Patrón habitual del equipo. |
| Métricas Prometheus | **prometheus-net** | Middleware para ASP.NET Core. |
| Logging JSON | **Serilog** con sink Console (JSON) | Compatible con Vector→OpenSearch igual que el resto. |
| Errores RFC 7807 | `ProblemDetails` built-in en ASP.NET Core | Mapear códigos al catálogo de ia-hub manualmente. |
| Linting / formato | `dotnet format` + analizadores Roslyn | |
| Tests | **xUnit** + `WebApplicationFactory` | |
