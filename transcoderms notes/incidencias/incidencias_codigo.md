# Incidencias de Codigo

## Criticas

### 1. Race condition en adquisicion de tareas

- `ContextLayer.cs:190-228` (`GetNextPendingTask`) - Read-then-update sin lock: dos threads pueden adquirir la misma tarea

### 2. processMap sin lock consistente

- `WorkloadManager.cs:82-95` (`CancelTask`) - Acceso sin lock mientras `ReportCompletedProcess` si lo usa

---

## Alta

### 4. Bug logico en limpieza de tareas antiguas

- `ContextLayer.cs:135` - `(x.finishDateTime - DateTime.Now).TotalDays > age_days` esta invertido: las tareas antiguas dan valor negativo, nunca se borran

### 5. .First() sin proteccion de coleccion vacia

- `ContextLayer.cs:238`, `ProcessTranscodeTask.cs:215`, `FfmpegLogic.cs:906,920` - `InvalidOperationException` si no hay elementos

### 6. Null checks ausentes tras queries

- `ContextLayer.cs:284,288,291` - `FirstOrDefault()` puede devolver null y se usa directamente

### 7. Cast inseguro de JSON

- `TaskController.cs:105-113` - `(int)wat["watermarkID"]` sin validacion de tipo

---

## Media

### 8. IP hardcodeada

- `Program.cs:48` - `http://10.1.1.66:8083/`

### 9. catch vacio

- `SharedQueue.cs:126` - Excepciones tragadas silenciosamente

### 10. Queries ineficientes

- `ContextLayer.cs:135` - `GetTasks()` carga TODAS las tareas en memoria para filtrar localmente

### 11. Thread.Sleep

- `ContextLayer.cs:486,529` - `Thread.Sleep(100)` innecesario

