# Soluciones a proponer (por cada "foco de riesgo")

## 1) **YOLO11 (Ultralytics) — AGPL-3.0 (riesgo alto)**

### **Solución A (la estándar en producto comercial)**
- **Comprar licencia Enterprise/Commercial de Ultralytics** para usar YOLO11 en software propietario (evita obligación de abrir código por AGPL).
- **Ventaja**: Cero cambios técnicos, riesgo legal muy bajo.
- **Desventaja**: Coste de licencia.

### **Solución B (plan B técnico)**
- **Cambiar a modelos con licencia permisiva (Apache/MIT)** y mantener ONNX Runtime.
- **Modelos alternativos**:
  - **RT-DETR** (Apache 2.0)
  - **YOLOX** (Apache 2.0)
  - Otros modelos de detección con licencia permisiva
- **Ventaja**: Sin coste de licencia, sin restricciones AGPL.
- **Desventaja**: Requiere trabajo técnico de migración y posible ajuste de precisión/rendimiento.

**Recomendación**: Licencia Ultralytics; alternativa técnica: RT-DETR/YOLOX; alternativa organizativa: uso interno.

---

## 2) **ImageSharp (SixLabors) — Split Apache 2.0 / Comercial (riesgo medio)**

### **Solución A (rápida, "pagar y seguir")**
- **Comprar licencia comercial SixLabors** si la empresa no cumple criterios "free".
- **Ventaja**: Cero cambios de código, riesgo legal muy bajo.
- **Desventaja**: Coste de licencia (~$1,000 USD/año por desarrollador).

### **Solución B (recomendada si queréis evitar coste y ambigüedad)**
- **Eliminar ImageSharp del producto**:
  - Sustituir conversiones/carga/resize por **OpenCvSharp** (Apache 2.0) o **System.Drawing.Common** (MIT) o **SkiaSharp** (MIT).
  - En el repo, ImageSharp se usa en `OostoApi.cs` y en `Score.cs`, así que habría que sustituir el uso de ImageSharp en ambas clases.
- **Ventaja**: Sin coste de licencia, control total de dependencias.
- **Desventaja**: Requiere trabajo técnico de migración.

### **Solución C (higiene/legal)**
- Aunque se mantenga, **documentar** que se usa y bajo qué criterio, y añadir **NOTICE/atribuciones** si aplica.

**Recomendación**: O compramos SixLabors (coste), o lo eliminamos con OpenCV/Skia (coste 0, trabajo técnico).

---

## 3) **YoloSharp (librería C#) — MIT (riesgo bajo)**

### **Solución A**
- Mantener YoloSharp y cumplir MIT (**añadir notices** en documentación/distribución).
- **Ventaja**: Sin cambios técnicos.
- **Desventaja**: Mantiene dependencia transitiva de ImageSharp.

### **Solución B**
- Si se quita ImageSharp, **migrar a ONNX Runtime directo** (MIT) y dejar de depender de YoloSharp.
- Esto **no toca** el tema Ultralytics/YOLO11, solo elimina SixLabors.
- **Ventaja**: Control total de dependencias, elimina ImageSharp.
- **Desventaja**: Requiere reescribir código que usa YoloSharp.

**Recomendación**: Cumplimiento MIT (simple). Alternativa: retirar YoloSharp para controlar dependencias.

---

## 4) **Microsoft.ML.OnnxRuntime — MIT (riesgo bajo)**

### **Solución**
- Mantener, añadir notices si procede.
- **No hay royalties**, solo obligación de atribución.

---

## Plan de opciones para decisión

### **Opción 1 (preferida por negocio: mínima fricción)**
- Mantener YOLO11 + **comprar Ultralytics Enterprise**.
- Revisar ImageSharp y **comprar SixLabors** si aplica.
- **Ventaja**: Cero cambios técnicos.
- **Desventaja**: Coste de licencias.

### **Opción 2 (mixta: reduce costes/licencias)**
- Mantener YOLO11 + **comprar Ultralytics Enterprise**.
- **Eliminar ImageSharp** (migración a OpenCvSharp/Skia) para no pagar SixLabors.
- **Ventaja**: Reduce coste de licencias (solo Ultralytics).
- **Desventaja**: Requiere trabajo técnico para eliminar ImageSharp.

### **Opción 3 (plan B técnico)**
- Sustituir YOLO11 por modelo Apache/MIT (RT-DETR/YOLOX).
- Mantener ONNX Runtime/OpenCV.
- Sin licencias comerciales.
- **Ventaja**: Sin coste de licencias, sin restricciones AGPL.
- **Desventaja**: Requiere trabajo técnico significativo (migración de modelo).

---

## Solución técnica detallada: Migrar de YoloSharp a ONNX Runtime + OpenCV directo

**Cambios técnicos**:
- Eliminar dependencia: `YoloSharp.Gpu`
- Implementar inferencia directa con:
  - `Microsoft.ML.OnnxRuntime` (ya instalado)
  - `OpenCvSharp4` (ya instalado)
- Reescribir clase `Scorer` para usar ONNX Runtime directamente
- Reemplazar uso de ImageSharp en:
  - `Logic/Analysis/VideoAnalytics/Scorers/Score.cs`
  - `Logic/Analysis/Oosto/OostoApi.cs`

**Resultado**: Elimina dependencia de ImageSharp y YoloSharp, manteniendo solo ONNX Runtime (MIT) y OpenCvSharp (Apache 2.0).
