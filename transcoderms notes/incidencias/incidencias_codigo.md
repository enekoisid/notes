# Incidencias de Codigo

## Criticas

### 1. Credenciales hardcodeadas en el repositorio
- `Program.cs:37` - Password del certificado HTTPS `"ISIDpwd7454"`
- `Include/config.json` - Password de BD `"1root1"`, claves AWS en texto plano
- `Dockerfile:33` - Certificado `cert.pfx` copiado al contenedor desde el repo

### 2. SQL Injection via ExecuteSqlRaw
- `TranscoderContext.cs:107-113` - Interpolacion de string directa en SQL:
  ```csharp
  db.Database.ExecuteSqlRaw($"DELETE FROM {tableName};");
  ```
- `ContextLayer.cs:501,544` - Mismo patron con `ALTER TABLE`

### 3. Race condition en adquisicion de tareas
- `ContextLayer.cs:190-228` (`GetNextPendingTask`) - Read-then-update no atomico: dos threads pueden adquirir la misma tarea

### 4. processMap sin lock consistente
- `WorkloadManager.cs:82-95` (`CancelTask`) - Acceso sin lock mientras `ReportCompletedProcess` si lo usa

### 5. Fuga de HttpClient (socket exhaustion)
- `ProcessTask.cs:45-47`, `ContextLayer.cs:71-73`, `Notifier.cs:40-43` - Nuevo `HttpClient` por operacion, nunca se dispone

---

## Alta

### 6. Bug logico en limpieza de tareas antiguas
- `ContextLayer.cs:135` - `(x.finishDateTime - DateTime.Now).TotalDays > age_days` esta invertido: las tareas antiguas dan valor negativo, nunca se borran

### 7. Bloqueo de threads con .Result / .Wait()
- `TaskController.cs:85` - `.Result` en handler HTTP
- `ProcessTask.cs:87,114-120,150,161,172` - `.Wait()` en operaciones S3
- `StorageManager.cs:28,129` - `.Wait()` dentro de `lock` (riesgo de deadlock)

### 8. .First() sin proteccion de coleccion vacia
- `ContextLayer.cs:238`, `ProcessTranscodeTask.cs:215`, `FfmpegLogic.cs:906,920` - `InvalidOperationException` si no hay elementos

### 9. Null checks ausentes tras queries
- `ContextLayer.cs:284,288,291` - `FirstOrDefault()` puede devolver null y se usa directamente

### 10. Path traversal via S3 key
- `ProcessTask.cs:94,102-105` - `s3pathOut.key` del usuario usado en `Path.Combine` sin sanitizar

### 11. Hash collision en watermarks
- `ContextLayer.cs:603-628` - `GetHashCode()` como ID unico: colisiones causan watermarks incorrectos

### 12. Cast inseguro de JSON
- `TaskController.cs:105-113` - `(int)wat["watermarkID"]` sin validacion de tipo

---

## Media

### 13. IP hardcodeada
- `Program.cs:48` - `http://10.1.1.66:8083/`

### 14. catch vacio
- `SharedQueue.cs:126` - Excepciones tragadas silenciosamente

### 15. Queries ineficientes
- `ContextLayer.cs:135` - `GetTasks()` carga TODAS las tareas en memoria para filtrar localmente

### 16. Thread.Sleep en ruta critica
- `ContextLayer.cs:486,529` - `Thread.Sleep(100)` innecesario

### 17. Sin HEALTHCHECK en Dockerfile

### 18. Stream de MediaInfo no dispuesto
- `Utilities.cs:101-187`

### 19. ProgressReader sin exception handling
- `ProgressReader.cs:19-42` - Loop infinito sin try/catch

### 20. Deadlock potencial en shutdown
- `WorkloadManager.cs:199-202` - `process.Join()` fuera de lock mientras otros threads modifican `processMap`

### 21. Logica compleja de SupraProfile sin validar
- `TaskController.cs:165-252`
