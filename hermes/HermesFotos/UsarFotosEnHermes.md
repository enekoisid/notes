## Extender el enum ResourceType
Añadir Image al enum `ResourceType` en `Utils.cs`

## Crear un nuevo ImageRecorder
Crear `ImageRecorder.cs` que herede de Recorder
> Este recorder tratará las imágenes como un "video" de un solo fotograma
> Simulará FPS y timestamp apropiados para imágenes

## Modificar Utils.GetResourceType
Detectar automáticamente si la URL es una imagen basándose en la extensión
Usar las extensiones ya definidas en ``OostoConfig.AllowedImageExtensions``

## Modificar AnalysisTask
Detectar el tipo de recurso y crear el recorder apropiado
- Para imágenes: usar ``ImageRecorder``
- Para videos: usar ``OpencvRecorder`` o ``GStreamerRecorder``

## El endpoint existente funcionará para ambos
El ***POST*** ``/api/Task`` aceptará tanto videos como imágenes
> El sistema detectará automáticamente el tipo y lo procesará correctamente

## Flujo de procesamiento para imágenes:
1. Cliente envía imagen: ***POST*** ``/api/Task`` con ``SourceUrl`` apuntando a una imagen
2. ``Utils.GetResourceType`` detecta que es una imagen
3. ``AnalysisTask`` crea un ``ImageRecorder`` en lugar de ``OpencvRecorder``
4. ``ImageRecorder`` carga la imagen como un único fotograma con **timestamp 0**
5. Los módulos de análisis procesan el fotograma normalmente
6. Se devuelve el resultado con la información de la imagen
