# **Extensión de Hermes para Procesamiento de Imágenes**

### **1. Detección automática de tipo de archivo**
- El sistema detecta automáticamente si la URL apunta a una imagen
- Usa las extensiones existentes en `Config.OostoConfig.AllowedImageExtensions`

### **2. Nuevo procesador de imágenes**
- Se creó un nuevo componente (`ImageRecorder`) que trata las imágenes como "videos de un solo fotograma"
- Carga las imágenes directamente en memoria (sin guardar archivos)
- Funciona con URLs locales y remotas (MinIO, S3, HTTP, etc.)

### **3. Selección automática de procesador**
- El sistema elige automáticamente el procesador correcto:
  - **Imágenes** → `ImageRecorder` (Nuevo)
  - **Videos** → `OpencvRecorder` 

### **4. Procesamiento unificado**
- Las imágenes pasan por el mismo flujo de análisis que los videos
- Todos los módulos de análisis funcionan igual (VideoAnalytics, Oosto, VaxALPR)
- Mismo formato de respuesta y notificaciones

## **Orden de ejecución**

### **Para imágenes:**
1. **Recepción**: Cliente envía imagen a través del mismo endpoint (`POST /api/Task`)
2. **Detección**: Sistema detecta automáticamente que es una imagen por la extensión
3. **Selección**: Elige `ImageRecorder` en lugar del procesador de videos
4. **Conversión**: Trata la imagen como un "video" de un solo fotograma
5. **Análisis**: Pasa el fotograma por todos los módulos de análisis configurados
6. **Resultado**: Devuelve los resultados en el mismo formato que los videos

## **Resultado final**

El ***POST*** `/api/task` puede procesar tanto **videos como imágenes**  y devolve los mismos tipos de análisis (detección de objetos, reconocimiento facial, lectura de matrículas, etc.), manteniendo toda la funcionalidad existente sin cambios.