
| Nombre en XML       | Función C++ llamada            |
| ------------------- | ------------------------------ |
| `TRANSCRIPCIONES`   | `TaskVideoma_WhisperX`         |
| `SYMBOL ANALYSIS`   | `TaskVideoma_SimbolosText`     |
| `TEXT ANALYSIS`     | `TaskVideoma_SimbolosText`     |
| `SUBTITULOS`        | `TaskVideoma_Subtitulos`       |
| `SPEAKERID`         | `TaskVideoma_SpeakerId`        |
| `OCR`               | `TaskVideoma_OCR`              |
| `TRANSLATE`         | `TaskVideoma_Translate`        |
| `SUMARIZACION`      | `TaskVideoma_Sumarize`         |
| `MEDIA DESCRIPTION` | `TaskVideoma_MediaDescription` |
| `WORD SPOTTING SD`  | `TaskVideoma_WordSpotting`     |
| `ALPR`              | `TaskVideoma_ALPR`             |
| `QUALITY CONTROL`   | `TaskVideoma_QC`               |
| `VIDEO ANALYTICS`   | `TaskVideoma_VideoAnalytics`   |
| `FACE RECOGNITION`  | `TaskVideoma_FaceRecognition`  |




---




| Proyecto             | Framework               | Veredicto           | Bloqueante principal                                                                 |
| -------------------- | ----------------------- | ------------------- | ------------------------------------------------------------------------------------ |
| `azure-translator`   | C# .NET Framework 4.6.1 | SÍ (tras migración) | Migrar a .NET 6+, quitar `Colorful.Console` y `System.Drawing` (solo decoración)    |
| `biovox_xt`          | C++ nativo              | NO                  | `windows.h`, `rpc.h`, `rpcrt4.lib`, librería propietaria BioVox                     |
| `nxindex`            | C# .NET Framework 4.5.2 | NO                  | .NET Framework + DLL propietaria `Nexidia.Workbench.NET.dll` (sin versión Linux)    |
| `nxsearch`           | C# .NET Framework 4.5.2 | NO                  | Igual que `nxindex`                                                                  |
| `nxtextnormalizer`   | C# .NET Framework 4.5.2 | NO                  | Igual que `nxindex`                                                                  |
| `ocrnet`             | C# .NET 6.0             | SÍ (casi directo)   | Cambiar `OpenCvSharp4.Windows` por `OpenCvSharp4.runtime.linux`                      |
| `openalpr`           | C/C++ con CMake         | SÍ                  | Core multiplataforma, excluir wrappers Windows                                       |
| `systran-translator` | C# .NET Framework 4.8   | SÍ (tras migración) | Migrar `.csproj` a .NET 6+. Código 100% portable, sin tocar ni una línea             |
| `vaxwinanalyzer`     | C++ nativo              | NO                  | `windows.h`, APIs Win32, librería propietaria Vaxtor (`libVaxWrap.lib`)              |
| `videoanalytics6`    | C# .NET 6.0             | SÍ (con esfuerzo)   | Cambiar `OpenCvSharp4.runtime.win` por linux + reempaquetar DLLs GStreamer/FFmpeg    |
| `voskconsole`        | C# .NET Framework 4.8   | SÍ (tras migración) | Migrar a .NET 6+. Código y dependencias (`Vosk`, `WebRtcVad`) ya son multiplataforma |

