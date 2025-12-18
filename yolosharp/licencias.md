### Análisis de licencias y riesgos

#### **Resumen ejecutivo**
- **Riesgo principal (alto)**: **YOLO11 (Ultralytics) → AGPL-3.0**. Si el producto **se vende** o se ofrece como **servicio** a clientes, **AGPL obliga a abrir el código fuente** del software que lo usa, salvo que se compre **licencia comercial/enterprise**.
- **Riesgo económico (medio)**: **SixLabors ImageSharp → licencia "split"**. En ciertos escenarios (p. ej. empresa > 1M USD ingresos anuales y uso directo), puede requerir **licencia comercial de pago**.
- **YoloSharp (la librería C#)**: El problema está en su uso de `ImageSharp`, aun siendo transitiva, se usa en el codigo de forma direct6a, por lo que podría haber problemas de licencia.

#### **Mapa de componentes y licencias**
- **Modelos**:
  - **YOLO11 (Ultralytics weights / ONNX)** → **AGPL-3.0** (**copyleft fuerte**, incluye uso por red).
- **Código/librerías**:
  - **YoloSharp.Gpu** → **MIT** (permisiva).
  - **SixLabors.ImageSharp** → **Split (Apache 2.0 o Commercial)** según criterios.
  - **Microsoft.ML.OnnxRuntime.Gpu/Managed** → **MIT** (permisiva).

#### **1) Ultralytics YOLO11 (modelos/pesos ONNX)**
- **Licencia**: **AGPL-3.0** (Ultralytics).
- **Qué implica para negocio**:
  - Si **vendemos/distribuimos** el producto o damos acceso por **red (SaaS/API)**, AGPL puede obligar a **publicar el código fuente** del servicio/aplicación que lo integra.
  - Alternativa típica: **licencia comercial/enterprise** de Ultralytics para uso propietario.
- **Riesgo**: **alto** (impacto directo en modelo de negocio).
  - Si el producto es **propietario** y se **vende** o se consume como **servicio** por clientes, **AGPL es incompatible** salvo licencia comercial.
  - El riesgo no es "royalty" automático; es **obligación de disclosure** (publicar código) o **comprar licencia**.
- **Mitigación**:
  - **Comprar licencia Ultralytics Enterprise**
  - **Cambiar a modelos con licencia permisiva** (Apache/MIT)

#### **2) SixLabors ImageSharp**
- **Licencia**: **Apache 2.0 / Six Labors Commercial Use License** (split).
- **Qué implica para negocio**:
  - En algunos casos (según criterios de SixLabors: tipo de consumo y umbral de ingresos), **puede requerir licencia comercial** (pago).
  - En nuestro código, **se usa explícitamente** (`using SixLabors.ImageSharp...`), lo que aumenta el riesgo de que se considere **uso directo** (aunque llegue transitivamente por NuGet).
  - Además, al importarse en código, es difícil defender "solo transitiva" ante auditoría.
- **Riesgo**: **medio** (económico/compliance; depende de ingresos y criterio).
  - Riesgo de **coste** y/o **incumplimiento** si no se cumple el criterio "free".
- **Mitigación**:
  - **Comprar licencia SixLabors** si no se cumplen criterios "free"
  - **Eliminar ImageSharp del producto** (migrar a OpenCvSharp/System.Drawing u otra alternativa con licencia permisiva).

#### **3) YoloSharp (nuget `YoloSharp.Gpu`)**
- **Licencia**: normalmente **MIT** (permisiva). **No es AGPL** (el componente AGPL es el **modelo/pesos** de Ultralytics).
- **Qué implica para negocio**:
  - MIT: no suele implicar pagos; solo **mantener avisos** de licencia.
  - Normalmente **sin royalties**; obligaciones de **atribución** y preservación de avisos.
- **Riesgo**: **bajo** por sí mismo.
- **Mitigación**: incluir notices; revisar dependencia transitiva (ImageSharp).

#### **Otros componentes (MIT / Apache 2.0)**
- **Microsoft.ML.OnnxRuntime.Gpu/Managed** → **MIT** (permisiva).
- Normalmente **sin royalties**; obligaciones de **atribución** y preservación de avisos. Apache además incluye **grant de patentes** y requisitos de NOTICE.

#### **Mitigación técnica propuesta (sin tocar el problema YOLO11)**
- **Eliminar ImageSharp** del binario (reemplazar conversiones de imagen por OpenCvSharp/System.Drawing, o pipeline ONNX directo).
- Mantener **OnnxRuntime + OpenCvSharp** (MIT/Apache).
- El problema de **YOLO11/AGPL** queda **igual** hasta que se tome decisión (licencia enterprise o cambio de modelo).

---

### Apéndice: “Qué significa” cada licencia (en 1 minuto)

- **MIT (permisiva)**:
  - **Puedes** usar/modificar/vender.
  - **Obligación**: incluir el texto de licencia/copyright.
  - **No** exige abrir tu código. **No royalties**.
- **Apache 2.0 (permisiva con patentes)**:
  - Similar a MIT, pero con **grant de patentes** y requisitos de **NOTICE** si aplica.
  - **No** exige abrir tu código. **No royalties**.
- **AGPL-3.0 (copyleft fuerte + red)**:
  - Si distribuyes o das acceso por **red** a un programa que incorpora AGPL, debes ofrecer el **código fuente** correspondiente bajo AGPL.
  - Muy problemática para software **propietario**.
  - “Solución” habitual: **licencia comercial** del titular o **usar alternativa** no-AGPL.
- **Licencia comercial (SixLabors / Ultralytics Enterprise)**:
  - Contrato de pago para permitir uso propietario y evitar obligaciones (p. ej., disclosure AGPL o restricciones split).
