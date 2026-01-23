# Resultados de comparaciones (YOLO)

## Global settings (inferencia)
- `conf_score`: 0.5
- `imgsz`: 640

## Resultados y videos
```\\10.1.1.61\Furia\training yolo\```

> Nota: este documento describe **resultados cualitativos** (observación visual sobre vídeos).

## Resumen ejecutivo
| Comparación | Vídeo(s) | Resultado | Observaciones clave |
|---|---|---|---|
| `yolo11n` vs `yolo11s` | Calle (cámara fija) | Gana `yolo11s` | Mejor detección de `handbag`; en `person/car/truck` se observan resultados similares |
| `yolo11n` vs `yolo11n-objects365` | Calle (otro ángulo) | Empate (con matices) | Detecciones similares en `person/car`; el modelo con Objects365 detectó `handbag` en un caso |
| `yolo11n` vs `yolo11n-visdrone` | Pegasus (helicóptero de tráfico) | Gana `yolo11n-visdrone` | Mejor detección de coches; ninguno detectó el tractor del vídeo |
| `yolo11(visdrone)` vs `yolov5m(visdrone)` | 5 vídeos (mixtos) | Gana `yolo11(visdrone)` (global) | Mejor detección aérea de vehículos; `yolov5m` fue más estable en un vídeo de montaña |

---

## Comparación 1: `yolo11n` vs `yolo11s` (modelo base)
### Objetivo
Comparar el impacto del tamaño del modelo (`n` vs `s`) en detección sobre una escena urbana.

### Setup
- **Modelos**: `yolo11n` vs `yolo11s`
- **Entrada**: vídeo de cámara fija apuntando a una calle con coches y peatones
- **Parámetros**: ver “Global settings”

### Resultados (cualitativos)
- Se observa una **ligera mejora** en detección con `yolo11s`, sobre todo en la clase `handbag`.
- `yolo11n` **no detectó `handbag`** en ningún momento durante esta prueba.
- Para `person`, `car` y `truck`, ambos modelos dieron una respuesta **muy similar**.

### Conclusión
En este vídeo, usar un modelo más grande (`yolo11s`) aporta detecciones adicionales (p. ej. `handbag`) que el modelo pequeño (`yolo11n`) pasa por alto, sin diferencias claras en clases grandes (`person/car/truck`).


---

## Comparación 2: `yolo11n` vs `yolo11n-objects365`
### Objetivo
Evaluar si un modelo con soporte de clases de Objects365 aporta valor frente al modelo base en escenas urbanas.

### Setup
- **Modelos**: `yolo11n` (base Ultralytics) vs `yolo11n-objects365`
- **Entrada**: vídeo de una calle desde otro ángulo, con altura similar a un poste de luz
- **Parámetros**: ver “Global settings”

### Resultados (cualitativos)
- Las detecciones fueron **parecidas**; ambos detectaron a las mismas personas y los mismos coches.
- Diferencia destacable (aunque pequeña): `yolo11n-objects365` detectó `handbag` (bolso de una señora), mientras que el modelo base no lo detectó.

### Observaciones
- Si se necesitan clases que solo existen en Objects365 (p. ej. `suv`, u otras específicas), el modelo base no las podrá devolver.
- Por lo observado, el modelo base es capaz para tareas que no requieran clases exclusivas de Objects365.

### Conclusión
En este vídeo, `yolo11n-objects365` aporta alguna detección extra (`handbag`) y habilita clases adicionales; para un caso de uso que dependa de esas clases, conviene entrenar/usar un modelo alineado con ese dataset.

---

## Comparación 3: `yolo11n` vs `yolo11n-visdrone` (Pegasus)
### Objetivo
Comparar el rendimiento en detección aérea de tráfico entre modelo base y un modelo entrenado con VisDrone.

### Setup
- **Modelos**: `yolo11n` vs `yolo11n-visdrone`
- **Entrada**: vídeo captado por Pegasus (helicóptero de tráfico)
- **Parámetros**: ver “Global settings”

### Resultados (cualitativos)
- Se aprecia claramente la superioridad de `yolo11n-visdrone` en detección de coches.
- Ninguno de los dos modelos detectó el tractor que aparece en el vídeo.

### Conclusión
Para escenas de tráfico desde altura, `yolo11n-visdrone` ofrece mejores detecciones de vehículos que el modelo base.

---

## Comparación 4: `yolo11(visdrone)` vs `yolov5m(visdrone)` (5 vídeos)
### Objetivo
Comparar resultados en varios vídeos entre un modelo YOLOv11 y un modelo YOLOv5 entrenados con VisDrone.

### Setup
- **Modelos**: `yolo11(visdrone)` vs `yolov5m(visdrone)`
- **Entrada**: 5 vídeos en total:
  - Parkour
  - Pegasus
  - Vista de dron de alpinistas
  - Rotonda / carretera
  - Coche solitario en carretera
- **Parámetros**: ver “Global settings”

### Resultados (cualitativos)
- **Vídeo de parkour**: `yolo11(visdrone)` captó más información que `yolov5m(visdrone)`.
- **Planos tipo dron**: ambos detectaron muy rara vez a las personas.
- **Detección aérea de vehículos y peatones**: el ganador general fue `yolo11(visdrone)`.
- **Vídeo de la rotonda**: `yolo11(visdrone)` detectó más coches.
- **Vídeo del coche solitario**: `yolo11(visdrone)` captó mas tiempo el coche (durante todo el vídeo).
- **Vídeo de montaña (alpinistas)**: resultados parecidos, pero `yolov5m(visdrone)` fue más estable al detectar personas.
- **Vídeo de Pegasus**: `yolo11(visdrone)` tuvo más falsos positivos con las letras alrededor del vídeo, pero detectó mejor los coches en general.

### Conclusión
En el conjunto de vídeos, `yolo11(visdrone)` rindió mejor para detección aérea de vehículos (y en general), aunque `yolov5m(visdrone)` mostró más estabilidad en un caso concreto (montaña).
