# IAHUBRT — Propuesta de diseño

**Autor**: Eneko Rebollo · **Versión**: 1.0 · **Fecha**: 2026-05-11

---

## Resumen

**IAHUBRT** (IA-Hub Real-Time) es un servicio nuevo que recibe peticiones para vigilar feeds RTSP y publica por MQTT las detecciones que producen los analizadores Vaxtor e Innovatrix. Es el "broadcaster" del sistema: una capa fina entre los clientes externos (subscribers) y `ia-hub`, que es quien hace el trabajo de visión por computador.

La idea base se sostiene sobre tres elecciones:

1. **`ia-hub` absorbe Vaxtor e Innovatrix** como dos workers nuevos (`vaxtor-worker` e `innovatrix-worker`) siguiendo el contrato estándar del repo. Toda la complejidad de gestionar RTSP, capacidad de GPU y concurrencia vive ahí.
2. **IAHUBRT habla sólo con `ia-hub`**, vía REST. Para sesiones en vivo abre un canal SSE largo (mismo patrón que el `/api/rag/stream` que ya está en producción), y mantiene el slot del pool reservado durante toda la conexión.
3. **MQTT como único canal hacia los subscribers**. Cada evento se normaliza en un envelope común que es idéntico provenga del worker que provenga.

El resultado es un servicio pequeño, sin estado pesado, que se puede mantener con poca gente y que evoluciona independientemente de los cambios en los modelos.

---

## Arquitectura

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
                          │ RTSP (lo consume el worker)
                          ▼
                    [cámaras IP]
```

**Reparto de responsabilidades**:

| `ia-hub` (cambios incrementales) | `IAHUBRT` (servicio nuevo) |
|---|---|
| `vaxtor-worker`: supervisa el ejecutable Vaxtor, expone `stream-start` (SSE) | REST API pública (`/api/sources`, `/api/channels`) |
| `innovatrix-worker`: controla Innovatrix por REST y consume su pub/sub ZMQ, expone `stream-start` (SSE) | State machine de canales |
| Rutas nuevas en Dispatcher: `POST /api/task/{vaxtor,innovatrix}/stream-start` | Cliente HTTP que abre SSE contra `ia-hub` |
| Pools nuevos en `concurrency.json` con sus `limit` | Normalizer + publicador MQTT |
| Concurrencia v2 cubre los nuevos pools sin tocar el motor | Métricas Prometheus + dashboard Grafana |

Una consecuencia importante: **el control de capacidad lo hace `ia-hub`**. Cuando un pool está saturado, `ia-hub` no llega a despachar la sesión; IAHUBRT lo percibe como "no se pudo arrancar" y deja el canal en `waiting`, reintentando con backoff. Cero reimplementación de semáforos en IAHUBRT.

> Nota terminológica: `ia-hub` usa **RabbitMQ (AMQP)** internamente para encolar tasks. Eso no se expone al exterior. La comunicación con clientes — incluida la nuestra — va por **HTTP REST**, con SSE para streams. **MQTT** sólo aparece en IAHUBRT hacia los subscribers.

---

## Cómo funciona — vida de una suscripción

```
1. POST /api/sources              → Source { id, url: "rtsp://10.1.1.1/…" }
2. POST /api/channels             → Channel { source_id, profile: "vaxtor:lpr-eu" }
3. PATCH /api/channels/{id} desired_state=running

4. IAHUBRT abre una conexión SSE contra ia-hub:
       POST /api/task/vaxtor/stream-start
       { rtsp_url, channel_id }
       Accept: text/event-stream

5. ia-hub Dispatcher adquiere slot del pool (mismo mecanismo que el
   gate de RAG), proxea el stream desde el worker.

6. vaxtor-worker abre el RTSP y emite un evento SSE por cada detección:
       event: detection
       data:  { event_type, payload_específico, timestamp }

7. IAHUBRT lee cada evento, lo envuelve en el envelope común y publica:
       mqtt://iahubrt/channel/{channel_id}/events

8. Para detener: PATCH desired_state=stopped → IAHUBRT cierra el SSE
   → el worker detecta el cierre, libera RTSP y libera el slot.
```

Si IAHUBRT cae en mitad de una sesión, la conexión SSE se rompe y el worker libera todo. Al reiniciar, IAHUBRT reconcilia: para cada canal con `desired_state=running` abre un nuevo SSE, arrancando sesión limpia. No hay buffer de eventos perdidos durante la caída — lo que importa es que el stream esté vivo, no recuperar una detección de hace tres segundos.

---

## Modelo de datos

Dos entidades, ambas en una base de datos local:

```
Source   { id, name, url, credentials_ref?, metadata }
Channel  { id, source_id, profile, state, desired_state,
           ia_hub_task_id?, last_error, stats }
```

El campo `profile` es un string opaco — `"vaxtor:lpr-eu"`, `"innovatrix:people-counter"` — que `ia-hub` valida y entiende. IAHUBRT no necesita conocerlo, sólo enrutarlo. Si más adelante hace falta CRUD de profiles, se añade; en fase 1 son strings.

**Estados del canal**: `waiting → running → stopped` | `error`.

Separamos `state` (lo que el sistema ha conseguido) de `desired_state` (lo que el usuario quiere). Igual que en Kubernetes — el usuario expresa intención y un reaper converge el sistema sin lógica adhoc en cada endpoint.

**Motor de base de datos**: SQLite por defecto, Postgres opcional. Configurable por variable de entorno:

```
DATABASE_URL=sqlite:////data/iahubrt.db                   # default
DATABASE_URL=postgresql+psycopg://user:pass@host/iahubrt  # opcional
```

El dataset es pequeño (sources + channels, escritura puntual). SQLite no justifica un servicio de DB aparte en fase 1. Si más adelante hace falta multi-replica o backup en caliente, cambiar la URL y correr las migraciones — sin tocar código.

---

## API REST

```
# Sources (fuentes RTSP)
POST   /api/sources                  { name, url, credentials?, metadata? }
GET    /api/sources
DELETE /api/sources/{id}             409 si tiene channels asociados
POST   /api/sources/{id}/probe       ping RTSP, devuelve codec/fps/resolución

# Channels (suscripción live)
POST   /api/channels                 { source_id, profile }
GET    /api/channels
PATCH  /api/channels/{id}            { desired_state }
DELETE /api/channels/{id}            idempotente, pasa por stopped antes de borrar

# Operación
GET    /health
GET    /metrics                      Prometheus
```

**Contrato de errores**: RFC 7807 (`application/problem+json`), mismo formato que `ia-hub`. Cada respuesta de error trae un `code` taxonomizado para que los clientes ramifiquen sin parsear `detail`. Catálogo inicial: `SOURCE_NOT_FOUND`, `PROFILE_NOT_FOUND`, `CHANNEL_NOT_FOUND`, `SOURCE_PROBE_FAILED`, `POOL_UNAVAILABLE`, `POOL_SATURATED`, `INVALID_TRANSITION`.

---

## Notificaciones MQTT

**Topics**:

```
iahubrt/system/status                       heartbeat            (retain)
iahubrt/system/events                       eventos atípicos
iahubrt/channel/{channel_id}/state          estado del canal     (retain)
iahubrt/channel/{channel_id}/events         detecciones del canal
```

`retain: true` en los topics de `…/state` permite que un subscriber nuevo descubra el estado actual sin esperar a la próxima transición.

**Envelope común** — idéntico provenga del worker que provenga:

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

El envelope lo construye IAHUBRT a partir del evento SSE que recibe. El worker sólo aporta `event_type` y `payload`; lo demás lo enriquece IAHUBRT con lo que ya sabe del canal. Así se cumple el requisito original: **el payload externo es el mismo para Vaxtor y para Innovatrix**.

---

## Workers en `ia-hub`

Dos workers nuevos siguiendo el contrato documentado en `.claude/rules/adding-a-worker.md`. Desde fuera son idénticos (URL RTSP entra, eventos SSE salen, slot del pool mantenido durante la sesión). Por dentro, cada uno integra su producto a su manera.

### `vaxtor-worker`

Un único endpoint:
- `POST /api/vaxtor/stream-start` → devuelve `text/event-stream`, emite un evento SSE por cada detección.

Vaxtor se distribuye como un ejecutable al que se le pasa la URL RTSP. El worker actúa de **supervisor de subproceso**: lanza el binario al recibir `stream-start`, captura su output (stdout o fichero de eventos rotatorio — se confirmará en el sprint), parsea cada detección y la emite por el SSE. Al cerrarse la conexión, envía `SIGTERM` al subproceso y espera con timeout.

Pool: `vaxtor_pool` con `limit: N` en `concurrency.json`. El slot se mantiene durante toda la conexión SSE.

### `innovatrix-worker`

Un único endpoint `POST /api/innovatrix/stream-start` con la misma semántica que Vaxtor de cara a IAHUBRT.

Innovatrix se controla en dos pasos:
1. **REST a la API de Innovatrix** para indicarle "vigila esta cámara".
2. **Suscripción ZeroMQ** a su publicación de eventos (socket SUB). El worker mantiene el socket abierto y reenvía cada evento por el SSE.

Al cerrar el SSE, el worker hace el `POST` de "deja de vigilar esta cámara" a Innovatrix y cierra el socket ZMQ.

### Cambios en el Dispatcher de `ia-hub`

- Constantes `VAXTOR_WORKER_URL` y `INNOVATRIX_WORKER_URL`.
- Rutas `POST /api/task/vaxtor/stream-start` y `POST /api/task/innovatrix/stream-start`: copian la lógica de `proxy_rag_stream` (adquieren slot, hacen proxy con streaming, sueltan slot al cierre).
- Pools `vaxtor_pool` e `innovatrix_pool` en `concurrency.json`.

No hay queues nuevas en RabbitMQ — el patrón `stream-start` es streaming HTTP puro, no pasa por encolado.

---

## Observabilidad y persistencia

### Métricas Prometheus

Prefijo `iahubrt_`. Lo mínimo para tener un dashboard útil desde el día 1:

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

La saturación de pools la exponen las métricas existentes de `ia-hub`; el dashboard de IAHUBRT enlaza al de ia-hub para esa fila.

**Dashboard Grafana inicial**, tres filas:
- *Estado global*: # canales por estado, eventos/s, reconexiones SSE.
- *Salud*: errores hablando con `ia-hub`, eventos dropeados, latencia de publicación.
- *API*: requests/s por ruta, latencia p95, 5xx rate.

### Historial de eventos en OpenSearch

Cada evento publicado a MQTT se duplica como **log estructurado JSON** que Vector (la pipeline ya existente en `ia-hub`) enruta al índice `iahubrt-events-*` en OpenSearch. Reusa la infra de logs en producción — sin código de cliente OpenSearch en IAHUBRT.

- Index template propio (`config/opensearch/iahubrt-events.json`), con keyword en `channel_id`, `event_type`, `pool`, `severity` para queries rápidas en Discover.
- Política ISM propia, retención 30 días (afinable).
- **Recuperación de datos perdidos**: ops usa Discover en Dashboards filtrando por `channel_id` y rango temporal. Sin endpoint REST de query en fase 1 — se añadiría como wrapper sobre OpenSearch (~30 líneas) si aparece un cliente que lo necesite.
- **Subscribers offline**: ortogonal. Si hace falta, se activa `persistence true` + `QoS 1` + sesiones persistentes en Mosquitto. No requiere tocar OpenSearch.

---

## Decisiones operacionales

| Decisión | Elección | Motivo corto |
|---|---|---|
| Canal de salida hacia subscribers | **MQTT** únicamente | Es el requisito original; mantiene contrato simple |
| Comunicación IAHUBRT ↔ ia-hub | **SSE** (mismo patrón que `/api/rag/stream`) | Patrón en producción; slot reservado durante la conexión |
| Alcance fase 1 | Sólo RTSP en vivo | Pruebas con un RTSP de test; el caso "vídeo suelto" se valora en fase 2 |
| Transporte de eventos de Innovatrix | **ZeroMQ** | Confirmado por el fabricante |
| Broker MQTT | **Mosquitto** | Imagen oficial, persistencia en volumen; sin TLS en fase 1 (LAN privada) |
| Auth en fase 1 | **HTTPBasic** en REST + Mosquitto `password_file` | Mismas credenciales por env; suficiente en LAN. Fase 2: JWT integrado con `iahubcms` |
| Persistencia de eventos | **Lite, vía OpenSearch** | Reusa pipeline existente; cubre debug y recuperación de datos perdidos |
| Base de datos | **SQLite por defecto, Postgres opcional** | Dataset pequeño; configurable por env |

---

## Implementación — Python o C#

El cuerpo del documento es agnóstico de lenguaje. **Los dos workers que entran en `ia-hub` (`vaxtor-worker`, `innovatrix-worker`) van obligatoriamente en Python** porque el repo lo es por convención. **Sólo IAHUBRT tiene libertad de elección.**

### Opción A — Python

| Pieza | Librería |
|---|---|
| HTTP server | **FastAPI** |
| ORM + migraciones | **SQLAlchemy 2.x** (async) + **Alembic** |
| Cliente HTTP / SSE | **httpx** |
| Cliente MQTT | **asyncio-mqtt** |
| Validación / config | **Pydantic v2** + **pydantic-settings** |
| Métricas | **prometheus-client** |
| Logging JSON | Vendoreado de `services/_shared/logging_setup.py` |
| Errores RFC 7807 | Vendoreado de `services/_shared/problem_details.py` |
| Linting + tests | **ruff** + **pytest** |

*Pros*: máxima coherencia con `ia-hub`, módulos vendoreables, cero duplicación en logging/errores, mismo CI/CD.

### Opción B — C# (.NET 8 LTS)

| Pieza | Librería |
|---|---|
| HTTP server | **ASP.NET Core** Minimal APIs |
| ORM + migraciones | **Entity Framework Core** |
| Cliente HTTP / SSE | `HttpClient` con streaming + parser SSE (o `LaunchDarkly.EventSource`; .NET 9 trae `System.Net.ServerSentEvents` nativo) |
| Cliente MQTT | **MQTTnet** |
| Validación / config | **FluentValidation** + `appsettings.json` |
| Métricas | **prometheus-net** |
| Logging JSON | **Serilog** (sink Console JSON, lo recoge Vector) |
| Errores RFC 7807 | `ProblemDetails` built-in, mapeo de `code` manual |
| Linting + tests | `dotnet format` + **xUnit** + `WebApplicationFactory` |

*Pros*: lenguaje habitual del equipo, IDE potente, ASP.NET Core sólido para SSE de larga duración. *Contra*: hay que re-implementar el contrato RFC 7807 y el formato de log (trabajo limitado pero introduce duplicación).

---

## Preguntas abiertas para la reunión

1. **Lenguaje de IAHUBRT**: Python (coherencia) vs C# (familiaridad del equipo).
2. **Formato exacto del output del ejecutable de Vaxtor**: stdout línea a línea o fichero de eventos rotatorio. Condiciona el parser del `vaxtor-worker`. Deberes con el fabricante.
3. **Capacidad real de los pools**: ¿qué `limit` razonable ponemos en `concurrency.json` para Vaxtor e Innovatrix en el hardware actual? Necesitamos un número (o un mecanismo para descubrirlo).
4. **Política de retención de eventos**: 30 días en `iahubrt-events-*` como propuesta. ¿Cubre necesidades de auditoría o hay que extender?

Todo lo demás (arquitectura, contratos REST/MQTT, integración con `ia-hub`, persistencia, auth, broker, BBDD) está propuesto en este documento y queda abierto a discutir.
