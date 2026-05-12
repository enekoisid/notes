# Modelos OCR en Ollama

## Modelos OCR dedicados

### GLM-OCR (Zhipu AI / Z.ai) — Recomendado

- **Parámetros:** 0.9B (CogViT 400M encoder + GLM-0.5B decoder)
- **Licencia:** MIT (pipeline completo incluye PP-DocLayoutV3 bajo Apache 2.0)
- **Comando:** `ollama run glm-ocr`
- **Formatos de entrada:** PNG, JPG, PDF (hasta 50MB / 100 páginas). TIFF/BMP/WEBP no documentados oficialmente pero probablemente funcionen vía PIL
- **Idiomas soportados:** chino, inglés, francés, español, ruso, alemán, japonés, coreano (8 idiomas confirmados; afirma +100 pero los no latinos como árabe/hindi son débiles, score 69.3 en multilingüe)
- **Salida:** Markdown, JSON, LaTeX
- **Ventana de contexto:** 128K tokens (en Ollama ajustar `num_ctx` a mínimo 16384, el default 4096 es insuficiente para imágenes)

#### Rendimiento y velocidad

| Métrica | Valor |
|---|---|
| Velocidad PDF | 1.86 páginas/segundo |
| Velocidad imágenes | 0.67 imágenes/segundo |
| Token generation (GPU) | ~260 tok/s |
| Token generation (CPU, M1 Pro) | ~60 tok/s |

Comparativa de velocidad:

| Modelo | Imágenes (img/s) | PDFs (págs/s) |
|---|---|---|
| **GLM-OCR** | **0.67** | **1.86** |
| PaddleOCR-VL-1.5 | 0.39 | 1.22 |
| DeepSeek-OCR-2 | 0.32 | — |
| MinerU 2.5 | 0.18 | 0.48 |

El mecanismo Multi-Token Prediction (5.2 tokens por paso de decodificación) aporta ~50% de mejora sobre decodificación autoregresiva estándar.

#### Consumo de recursos

| Precisión | VRAM aprox. | Tamaño modelo |
|---|---|---|
| BF16/FP16 | 2-4 GB | 2.2 GB (Ollama) |
| Q8_0 (GGUF) | ~1-1.5 GB | 950 MB |
| Q4 (quantized) | ~0.8-1 GB | — |

- **RAM total en inferencia:** ~2.5 GB (Ollama en Mac)
- **Mínimo recomendado:** 8 GB RAM, GPU recomendada
- **CPU es muy lento:** vision encoding tarda ~3 minutos en M1 Pro vs segundos en GPU

Hardware de referencia:

| Hardware | VRAM usado | Throughput |
|---|---|---|
| RTX 3060 | 4-6 GB | ~1.5 págs/s |
| RTX 4090 | 4-6 GB | ~2.5 págs/s |
| AWS g4dn.xlarge | 16 GB | ~1.8 págs/s |
| 4x A100 (80GB) | distribuido | ~7.0 págs/s |

#### Benchmarks de precisión

**OmniDocBench V1.5 — #1 overall (score 94.62):**
- Text Edit Distance: 0.040
- Formula CDM: 93.90
- Table TEDS: 93.96
- Table TEDS-S: 96.39
- Reading Order Edit: 0.044

Comparativa con otros modelos:

| Modelo | OmniDocBench v1.5 | Parámetros |
|---|---|---|
| **GLM-OCR** | **94.62** | 0.9B |
| PaddleOCR-VL-1.5 | 94.50 | 0.9B |
| DeepSeek-OCR-2 | 91.09 | 3B |
| Gemini-3 Pro | 90.33 | enorme |
| MinerU 2.5 | 90.67 | — |
| Qwen3-VL-235B | 89.15 | 235B |

Otros benchmarks:

| Benchmark | GLM-OCR | Mejor competidor |
|---|---|---|
| OCRBench (texto) | **94.0** | dots.ocr (92.1) |
| UniMERNet (fórmulas) | **96.5** | Varios (96.4) |
| TEDS_TEST (tablas) | **86.0** | MinerU 2.5 (85.4) |
| PubTabNet (tablas) | 85.2 | MinerU 2.5 (**88.4**) |

#### Rendimiento por tipo de documento

| Tipo de documento | Score | Notas |
|---|---|---|
| Recibos/facturas (KIE) | 94.5 | Supera a GPT-5.2 (83.5) |
| Tablas reales | 91.5 | Muy fuerte, aunque pierde vs MinerU en PubTabNet |
| Sellos/stamps | 90.5 | — |
| Texto manuscrito | 87.0 | Bueno pero inferior a Gemini-3 Pro (94.5) |
| Código fuente | 84.7 | — |
| Fórmulas | 96.5 | Excelente |
| Multi-columna | 76.7 | Manejado vía PP-DocLayoutV3 |
| Multilingüe | 69.3 | Débil en scripts no latinos |
| Escaneos antiguos/degradados | 37.6 | Punto débil principal |

#### Tareas soportadas

1. **Reconocimiento de texto** — extracción de texto plano
2. **Reconocimiento de tablas** — recuperación de estructura con alineación fila/columna (HTML)
3. **Reconocimiento de fórmulas** — salida LaTeX
4. **Extracción de información** — le pasas un esquema JSON y lo rellena con datos del documento (facturas, IDs, recibos)
5. **Parsing de documentos** — conversión estructural completa con análisis de layout a Markdown

#### Limitaciones

1. **Pipeline de dos etapas:** errores de PP-DocLayoutV3 (layout) se propagan al OCR sin corrección posible
2. **Texto manuscrito:** significativamente peor que modelos frontier (86.1 vs 94.5 Gemini-3 Pro)
3. **Escaneos viejos/degradados:** rendimiento muy pobre (37.6 en olmOCR)
4. **Scripts no latinos:** débil en árabe, hindi y otros (69.3 multilingüe)
5. **Sin razonamiento:** no puede responder preguntas sobre el contenido ni hacer análisis cross-page
6. **CPU impracticable:** vision encoding ~3 min en CPU vs segundos en GPU
7. **Variación estocástica:** pequeñas diferencias no deterministas en formato entre ejecuciones

#### Despliegue

- **Cloud API:** $0.03 por millón de tokens (Z.ai MaaS)
- **Self-hosted:** vLLM (recomendado para producción), SGLang, Ollama, Transformers
- **GGUF:** disponible via ggml-org/GLM-OCR-GGUF y llama.cpp
- **Fine-tuning:** soportado via LLaMA-Factory

---

### DeepSeek-OCR (DeepSeek)

- **Parámetros:** 3B (MoE, solo 570M activos). Basado en DeepSeek-VL2
- **Licencia:** MIT (v1), Apache 2.0 (v2)
- **Comando:** `ollama run deepseek-ocr`
- **Tamaño Ollama:** 6.7 GB
- **Disponible desde:** Ollama 0.13
- **Formatos de entrada:** PNG, JPG, PDF (multi-página, renderiza cada página como imagen)
- **Dimensión máxima imagen:** 2000px (redimensiona proporcionalmente si excede)
- **Enfoque:** compresión óptica de contexto — comprime imágenes de páginas en tokens de visión compactos

#### Modos de resolución

| Modo | Resolución | Tokens de visión |
|---|---|---|
| Tiny | 512x512 | 64 |
| Small | 640x640 | 100 |
| Base | 1024x1024 | 256 |
| Large | 1280x1280 | 400 |
| Gundam | 1024x640 crop | múltiples ventanas |

#### Modos de prompt

- `Free OCR.` — extracción rápida de texto
- `<|grounding|>Convert the document to markdown.` — OCR con coordenadas de bounding box, ideal para layouts multi-columna
- `Extract the text in the image.` — extracción directa

**Nota:** sensible al prompt — "un punto o salto de línea que falte puede causar output incorrecto" (docs de Ollama)

#### Consumo de recursos

| Escenario | VRAM |
|---|---|
| FP16 completo | ~16 GB mínimo, 24 GB recomendado |
| Q4 quantized | ~2 GB |
| Uso reportado real | ~10-12 GB con Flash Attention en tarjeta 32GB |

- **RAM:** 32 GB mínimo, 64 GB recomendado para producción
- **Comparativa con GLM-OCR:** GLM-OCR necesita solo 2-4 GB VRAM y 8 GB RAM

#### Rendimiento y velocidad

| Métrica | Valor |
|---|---|
| Throughput en A100-40G | ~2,500 tokens/segundo |
| Producción PDF (A100) | 200k+ páginas/día |
| Latencia por página (comunidad) | ~10 segundos/página con Flash Attention |
| Tesis de 100 páginas (A100) | ~2 minutos |
| API inference (Simplismart) | 800 tokens/segundo |

#### Benchmarks de precisión

**OmniDocBench v1.5:** 91.09% (DeepSeek-OCR-2) — vs GLM-OCR 94.62%

**olmOCR-bench:**

| Categoría | Score |
|---|---|
| Overall | 75.7 |
| Arxiv Math | 77.2 |
| Tablas | 80.2 |
| Headers/Footers | 96.1 |
| Texto largo pequeño | 79.4 |
| Multi-columna | 66.4 |
| Escaneos viejos math | 73.6 |
| Escaneos viejos | 33.3 |

**Precisión por compresión:**
- <10x compresión: 97% OCR precision
- 15x compresión: ~86-87%
- 20x compresión: ~60% (se degrada mucho)

**Precisión reportada general:**
- Texto impreso limpio: 99%+
- Notas manuscritas: 92%+
- Fórmulas: 95%
- Documentos mixtos (formularios, recibos, escaneos): 96-97% token-level

#### Rendimiento por tipo de documento

| Tipo | Rendimiento | Notas |
|---|---|---|
| Documentos impresos limpios | Excelente (99%+) | — |
| Recibos/facturas | Bueno | Parsing layout-aware, extracción estructurada |
| Tablas | Moderado (80.2) | Problemas con celdas fusionadas, multi-headers, tablas cross-page |
| Multi-columna | Moderado (66.4) | Layouts creativos requieren intervención manual |
| Fórmulas matemáticas | Bueno (~95%) | — |
| Texto manuscrito | Moderado (92%+) | Puede alucinar contenido en vez de admitir ilegibilidad |
| Escaneos viejos/degradados | Pobre (33.3) | — |
| Documentos inclinados/rotados | Pobre | Problemas con márgenes inconsistentes y rotación |

#### Limitaciones

1. **Escaneos antiguos:** score 33.3% — rendimiento severamente degradado
2. **Multi-columna débil:** solo 66.4% — peor que GLM-OCR (76.7%)
3. **Compresión agresiva:** a 20x la precisión cae a ~60%
4. **Alucinaciones:** inventa texto en vez de admitir ilegibilidad (GitHub issue #312)
5. **Bounding boxes inconsistentes:** posiciones a veces inventadas
6. **Código Python en imágenes:** problemas de reconocimiento (issue #320)
7. **Texto en parte inferior de imagen:** a veces no detectado (issue #304)
8. **Tablas:** filas desordenadas, celdas fusionadas incorrectamente (issue #305)
9. **GPU utilization stuck at 25%** para algunos usuarios (issue #328)
10. **Errores Triton/CUDA** en vLLM 0.11.2 (issue #299)
11. **Sin soporte nativo macOS/Apple Silicon** para vLLM (issue #319)
12. **247 issues abiertos** en GitHub — señal de inmadurez relativa
13. **Degradación en producción:** accuracy baja 15-25% vs benchmarks en layouts complejos

#### ¿Por qué hay menos documentación que GLM-OCR?

1. **Enfoque diferente:** la contribución de DeepSeek-OCR es la innovación en compresión óptica, no el OCR tradicional. Su paper se centra en el mecanismo de compresión
2. **GLM-OCR es más nuevo y purpose-built:** diseñado exclusivamente para document understanding (marzo 2026)
3. **Simplicidad de GLM-OCR:** 0.9B params es mucho más fácil de desplegar → más guías de la comunidad
4. **Complejidad de DeepSeek-OCR:** múltiples modos de resolución, ratios de compresión y tuning de GPU hacen difícil documentar de forma concisa
5. **Transparencia de datos de entrenamiento:** preguntas sobre posible uso de Anna's Archive sin disclosure adecuado

#### Arquitectura técnica

**DeepSeek-OCR v1:**
- Stage 1 (DeepEncoder): SAM vision transformer windowed + CLIP-Large encoder + compresor convolucional 16x
- Stage 2 (Decoder): DeepSeek-3B-MoE-A570M (Mixture of Experts, solo 570M activos)
- Token `<|grounding|>` para output con layout preservado

**DeepSeek-OCR v2:**
- DeepEncoder V2: reutiliza un decoder Qwen2 como vision encoder
- Visual Causal Flow: captura vista global de página primero, luego procesa secuencialmente
- Repetition rate reducido de 6.25% a 4.17% (imágenes), 3.69% a 2.88% (PDFs)

---

## Comparativa directa GLM-OCR vs DeepSeek-OCR

| Aspecto | GLM-OCR | DeepSeek-OCR-2 |
|---|---|---|
| Parámetros | 0.9B | 3B (570M activos) |
| OmniDocBench v1.5 | **94.62%** | 91.09% |
| VRAM mínimo | 2-4 GB | 16 GB (FP16) |
| Velocidad PDF | **1.86 págs/s** | ~10s/página |
| Velocidad imágenes | **0.67 img/s** | 0.32 img/s |
| Tablas (olmOCR) | 77.6 | **80.2** |
| Multi-columna | **76.7** | 66.4 |
| Escaneos viejos | **37.6** | 33.3 |
| Token efficiency | Estándar | **Mejor** (100 tokens/página) |
| Batch throughput (A100) | — | 200k+ págs/día |
| Despliegue | Fácil (Ollama directo) | Complejo (tuning GPU) |
| Madurez/estabilidad | Alta | 247 issues abiertos |
| Licencia | MIT + Apache 2.0 | MIT / Apache 2.0 |

---

## Modelos multimodales generalistas (con OCR fuerte, licencia libre)

| Modelo | Parámetros | Licencia | Comando | Notas |
|---|---|---|---|---|
| Qwen2-VL | 2B / 7B | Apache 2.0 | `ollama run qwen2-vl` | Muy bueno en OCR multilingüe, texto manuscrito y documentos complejos |
| Granite 3.2 Vision (IBM) | 2B | Apache 2.0 | `ollama run granite3.2-vision` | Diseñado para comprensión de documentos: tablas, gráficos, diagramas. Ideal para edge |
| Moondream | 2B | Apache 2.0 | `ollama run moondream` | Ultra-ligero, OCR mejorado para documentos y tablas |
| LLaVA 1.6 | 7B/13B/34B | Apache 2.0 | `ollama run llava` | OCR correcto pero menos preciso que los dedicados |

## Descartados para venta libre (restricciones de licencia)

| Modelo | Problema de licencia |
|---|---|
| Llama 3.2 Vision / Llama 4 | Requiere atribución "Built with Llama", naming obligatorio y Acceptable Use Policy de Meta |
| MiniCPM-V | Uso comercial gratuito solo con <5.000 dispositivos o <1M DAU; por encima necesita autorización |
| Qwen2.5-VL 7B | Qwen Research License, no permite uso comercial |
| Gemma | Licencia propietaria de Google con restricciones |

## Recomendación

Para aplicación de venta libre con escaneo de PDFs:
- **Motor principal:** GLM-OCR — rápido, ligero, preciso, MIT. Gana en prácticamente todas las métricas relevantes y es mucho más fácil de desplegar
- **Complemento** (comprensión profunda / tareas complejas): Qwen2-VL 7B — Apache 2.0
- **DeepSeek-OCR** queda como alternativa viable solo si se necesita eficiencia extrema en tokens (batch masivo en A100) o si los modos de prompt con bounding boxes son imprescindibles. Para el caso general, GLM-OCR es superior en precisión, velocidad, consumo y facilidad de despliegue
