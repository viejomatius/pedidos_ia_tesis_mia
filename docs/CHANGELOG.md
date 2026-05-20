# 📑 Control de Cambios (Changelog)

Todas las actualizaciones notables, decisiones arquitectónicas y mejoras en las métricas de este proyecto se documentan en este archivo, siguiendo el estándar [Keep a Changelog 1.1.0](https://keepachangelog.com/es-ES/1.1.0/).

> **Nota sobre nomenclatura de versiones:** las entradas `[V3.0]`–`[V14.0]` numeran iteraciones sucesivas del notebook durante la fase de desarrollo (marzo–abril 2026). Las entradas `[V3.3-notebook]`–`[V3.0-notebook]` corresponden al versionado académico final del notebook (mayo 2026) tal como aparece en el código: `V3`, `V3.1`, `V3.2`, `V3.3`. La secuencia de desarrollo interna y la nomenclatura académica son paralelas e independientes.

---

## [V3.3-notebook] - 2026-05-12 (Trazabilidad input-output)

### Contexto

Tras estabilizar la UI con V3.1 y agregar smart load del Ablation con V3.2, se identificó una oportunidad crítica de explicabilidad: el operador humano veía qué SKU había identificado el sistema pero **no veía qué fragmento del texto del cliente generó ese ítem**. La versión V3.3 implementa trazabilidad input-output completa.

### Añadido

- **Campo `texto_fuente: Optional[str]` en el schema `PedidoIA` (Celda 2):** preserva el fragmento exacto del pedido del cliente que generó cada ítem. Opcional para preservar retro-compatibilidad con resultados de cache previos.

- **REGLA 5 [TRAZABILIDAD INPUT-OUTPUT] en PROMPT_EXTRACTOR_V2 (Celda 5):** instrucción explícita al LLM con dos ejemplos few-shot para que emita el campo `texto_fuente` por cada ítem. Cita textual, no reformulada.

- **Función `_extraer_fragmento_fuente_fallback()` (Celda 5):** patrón defense-in-depth aplicado a UI. Si el LLM no emite el campo, una función de fallback heurístico localiza la cantidad numérica usando regex y devuelve el texto desde 5 caracteres antes del match hasta 60 caracteres después (`inicio = match.start() - 5`, `fin = match.end() + 60`). Si no encuentra match literal (ejemplo: "una docena" en lugar de "12"), devuelve el texto completo con prefijo `[aprox.]` como marker visual de aproximación.

- **Columna "📝 Texto leído" en la tabla Gradio (Celda 20A):** entre "Ítem #" y "Estado". Nunca queda vacía: 3 escenarios (LLM emite / fallback exitoso / aproximación `[aprox.]`).

- **Payload JSON downstream enriquecido:** cada ítem del JSON que viajará al ERP incluye ahora el campo `texto_fuente`.

### Cambiado

- **`version_sistema`** del payload: `"v3.1"` → `"v3.3"` para señalar el cambio del contrato downstream.

- **Header de la columna "Detalle":** simplificado para diferenciarlo de la nueva columna "Texto leído".

- **Sin impacto en métricas oficiales:** la trazabilidad es un cambio post-evaluación. Las métricas F1 Identificación (0.7633) y F1 Transaccional (0.3067) se preservan idénticas. El smart load V3.2 carga `metricas_oficiales_v3.json` sin re-ejecutar.

### Impacto en costos

- Incremento estimado: **~$0.0003 USD por pedido** (~4 %) por los ~50-80 tokens adicionales en la respuesta del LLM.
- Latencia: **+~0.1-0.2 s** apenas perceptible.

---

## [V3.2-notebook] - 2026-05-10 (Smart Load del Ablation)

### Contexto

La celda de Ablation V3 consumía API (~$0.40 USD) cada vez que se ejecutaba, incluso cuando los resultados ya estaban persistidos. Esto desincentivaba la iteración del notebook.

### Añadido

- **Tres modos de operación explícitos para `cargar_o_ejecutar_ablation_v3` (Celda 15):**
  - `"auto"` (default): carga si existe archivo válido y consistente, ejecuta si no.
  - `"force_recompute"`: fuerza re-ejecución descartando archivo existente.
  - `"load_only"`: solo carga, falla con `FileNotFoundError` si no existe.

- **Validación estructural:** verifica que el JSON tenga campos `metadata` y `runs`, que los modelos esperados estén presentes (`gpt-4o-mini`, `gpt-4o`), y que cada run tenga métricas duales V3 (`f1_id_macro` Y `f1_tx_macro`).

- **Validación de consistencia de configuración:** compara `UMBRAL_CONFIANZA_AUTOMATICA`, `PESO_FAISS`, `PESO_BM25`, `UMBRAL_SIMILITUD_HITL`, `N_CASOS` del archivo con la configuración runtime actual. Si difieren, re-ejecuta y notifica qué cambió.

- **Helper `_cargar_resultados_v3_si_existen()`:** centraliza la lógica de carga + validación con manejo seguro de errores.

### Cambiado

- **Patrón aplicado:** Cache Pattern con Cache Invalidation explícita. Reduce el costo de re-ejecución de ~$0.40 USD a **$0.00 USD** cuando los resultados ya son válidos.

---

## [V3.1-notebook] - 2026-05-08 (UI Gradio enriquecida)

### Contexto

Cuatro oportunidades de mejora UX identificadas en sesión de pruebas: (1) la tabla mostraba el SKU pero no el nombre del producto, (2) el campo "Detalle" estaba truncado, (3) no había forma de inspeccionar el JSON downstream, (4) no había acción final de confirmación.

### Añadido

- **Columna "Producto" en la tabla (Celda 20B):** lookup del campo `Nombre_Tecnico` en `df_catalogo`.

- **Detalle sin truncar (Celda 20B):** eliminado el corte `[:80]`. Columna "Detalle" con `wrap=True` en `gr.DataFrame`.

- **Accordion "📦 Payload JSON" colapsable (Celda 20B):** componente `gr.Accordion(open=False)` con `gr.Code(language="json")`.

- **Botón "📤 Confirmar Pedido y Generar JSON" (Celdas 20A/20B):** nuevo Paso 4. Genera `pedido_<timestamp>.json` en `/content/pedidos_confirmados/`. Componente `gr.File` para descarga directa.

- **Validación fail-safe en confirmación (Celda 20A):** `handler_confirmar_pedido_completo` rechaza generar el JSON si quedan ítems con `sku=REVISION_MANUAL`. Codifica el principio **no-default-deny**.

### Cambiado

- **Campo `version_sistema`** del payload: introducido con valor `"v3.1"` para versionado explícito del contrato downstream.

---

## [V3.0-notebook] - 2026-05-02 (V3 Tutor-Compliant: Respuesta a las 11 observaciones docentes)

### Contexto

El tutor académico emitió `REVISION-CAPSTONE-IA-PEDIDOS-001` (27 abril 2026) con 11 observaciones técnicas sobre el notebook V2. Análisis cruzado identificó 5 puntos críticos no resueltos (OBS-01, 02, 03, 04, 05) y 6 puntos resueltos en el refactor previo.

### Añadido

- **Filtro determinístico de confianza (resuelve OBS-01 — Celdas 2, 5):** `UMBRAL_CONFIANZA_AUTOMATICA` parametrizable y función `confianza_es_suficiente()`. El filtro opera como **Filtro #2** en `validar_inventario`, forzando `REVISION_MANUAL` cuando `confianza_ia < umbral`, independientemente del SKU emitido por el LLM. Patrón defense-in-depth aplicado a IA.

- **Métrica dual de evaluación (resuelve OBS-02 — Celda 8):** nueva función `calcular_metricas_pedido_dual()` que reporta:
  - **F1 Identificación** (solo SKU): compatible con V1/V2.
  - **F1 Transaccional** (SKU + cantidad exacta): refleja validez B2B real.
  - **MAE cantidad** (Mean Absolute Error en unidades).
  - **Categorización automática de errores** en 7 categorías.

- **Reciprocal Rank Fusion (resuelve OBS-03 — Celda 4):** clase `BuscadorHibridoRRF` implementando `score_RRF(d) = Σ_i peso_i / (k + rank_i(d))` con `peso_faiss=0.5`, `peso_bm25=0.5`, `rrf_k=60`. Alias `BuscadorHibridoPersonalizado = BuscadorHibridoRRF` para retro-compatibilidad. Referencia: Cormack et al., SIGIR'09.

- **PROMPT_REWRITER_V2 con heurísticas léxicas (resuelve OBS-04 — Celda 5):** 4 reglas + 4 ejemplos few-shot. Distingue cantidades de compra (eliminables) de atributos técnicos numéricos (preservables: calibres, tipos, dimensiones).

- **Memoria HITL con umbral de similitud (resuelve OBS-05 — Celda 4):** `ServicioMemoriaHITL` con `top_k=3` (antes: 1) y `umbral_similitud=0.85`. Instrucción reformulada de *"úsala sin cuestionar"* a *"usa como referencia y verifica que aplique"*.

- **Suite de tests pytest-style integrada (Celda 14):** 8 tests críticos:
  - `test_confianza_es_suficiente_OBS01`
  - `test_validar_inventario_filtra_confianza_baja_OBS01`
  - `test_metricas_dual_OBS02`
  - `test_rrf_fusion_balanceada_OBS03`
  - `test_prompt_rewriter_preserva_numeros_OBS04`
  - `test_memoria_hitl_umbral_OBS05`
  - `test_categorizar_error_correctamente`
  - `test_pipeline_e2e_smoke`

- **Generador de ejemplos para defensa (Celda 18):** 4 artefactos demo en `/content/` (texto, imagen JPG, PDF nativo, caso ambiguo).

- **Dashboard académico — figuras 1 y 2 (Celda 19A):**
  - `figura_01_comparativo_f1_v3.png`: F1 entre 6 configuraciones.
  - `figura_02_iteracion_v1_v2_v3.png`: evolución V1→V2→V3 + métrica transaccional.
  - [Figuras adicionales — PENDIENTE - verificar Celda 19B]

- **Backup a Google Drive (Celda 21):** respalda artefactos críticos a `Tesis_Backup/backup_<timestamp>/`.

- **Refuerzo prompt VLM bajo GDPR Art. 5(1)(c) (refuerza OBS-10 — Celda 10):** cita explícita al principio de minimización de datos.

- **Renombrado del archivo HITL (refuerza OBS-11 — Celda 2):** `dataset_finetuning_hitl.jsonl` → `historial_correcciones_hitl.jsonl`.

### Cambiado

- **`PROMPT_EXTRACTOR_V2` activo desde el inicio (Celda 5):** antes requería hot-swap; ahora es el prompt activo por defecto.

- **`AppSettings` extendida (Celda 2):** nuevos campos: `UMBRAL_CONFIANZA_AUTOMATICA` (default `"Alta"`), `UMBRAL_SIMILITUD_HITL` (default `0.85`), `PESO_FAISS`/`PESO_BM25` (default `0.5`/`0.5`), `RRF_K` (default `60`), `K_MEMORIA_HITL` (1 → 3), `RUTA_HISTORIAL_HITL` (renombrado).

- **Benchmark Generator V3 (Celda 7):** ground truth preserva `items_esperados: List[Tuple[str, int]]` en lugar de solo SKUs. Retro-compatibilidad via propiedad `caso.skus_esperados`.

- **Comparativo V1→V2→V3 (Celda 16):** deltas explícitos `Δ V1→V3`.

### Métricas oficiales V3 (run 2026-05-15, `metricas_oficiales_v3.json`)

#### GPT-4o + Prompt V2 + RRF + Filtros V3 (configuración recomendada)

| Métrica | V1 (gpt-4o) | V2 (gpt-4o) | **V3 (gpt-4o)** | Δ V1→V3 |
|---|---|---|---|---|
| F1 Identificación | 0.6433 | 0.8000 | **0.7633** | +0.1200 |
| F1 Transaccional | — | — | **0.3067** | (nuevo V3) |
| MAE cantidad | — | — | **9.3** unidades | (nuevo V3) |
| Latencia promedio | 3.18 s | 2.97 s | 4.17 s | +0.99 s |
| Costo por pedido | $0.00615 | $0.00703 | $0.00850 | +$0.00235 |
| Aciertos perfectos (id) | — | — | 31/50 (62 %) | — |
| Aciertos perfectos (tx) | — | — | 11/50 (22 %) | (nuevo V3) |

#### GPT-4o-mini + Prompt V2 + RRF + Filtros V3

| Métrica | V1 (mini) | V2 (mini) | **V3 (mini)** | Δ V1→V3 |
|---|---|---|---|---|
| F1 Identificación | 0.5700 | 0.6633 | **0.7527** | +0.1827 |
| F1 Transaccional | — | — | **0.2833** | (nuevo V3) |
| MAE cantidad | — | — | **9.92** unidades | (nuevo V3) |
| Latencia promedio | 4.26 s | 5.62 s | 7.42 s | +3.16 s |
| Costo por pedido | $0.000353 | $0.000416 | $0.000492 | +$0.000139 |

#### Distribución de errores — GPT-4o V3 (n=50)

| Categoría | Casos | % |
|---|---|---|
| derivacion_correcta | 25 | 50 % |
| acierto_perfecto | 11 | 22 % |
| derivacion_innecesaria | 10 | 20 % |
| alucinacion_sku | 3 | 6 % |
| falla_no_deriva | 1 | 2 % |

### Hallazgo central (anticipado por el tutor en OBS-02)

> **Brecha F1 Identificación vs F1 Transaccional: 0.4566 puntos** (gpt-4o V3). El sistema identifica el SKU correcto en el 76 % de los casos, pero SKU + cantidad simultáneamente solo en el 31 %. Esto cuantifica el costo real de errores en cantidad — invisible bajo la métrica original.

### Hallazgo Pareto (no anticipado)

> **GPT-4o es ~44 % más rápido que GPT-4o-mini** (4.17 s vs 7.42 s) siendo 16× más caro por token. En B2B donde la latencia importa más que el costo unitario, `gpt-4o` domina en TCO real.

---

## [V14.0] - 2026-04-29 (Refactor consolidado pre-V3)

### Añadido
- **Refactor arquitectónico completo del notebook:** consolidación en arquitectura modular con `OrquestadorPipeline` como State Manager en cada celda.
- **Setup local profesional:** estructura Clean Architecture con carpetas `app/{api,core,adapters,services,schemas}`, `requirements.txt` versionado con LangChain pinned 0.3.27, `.env.example` template, `.gitignore` de 104 líneas.
- **Repositorio Git inicializado:** rama `main` con primer commit siguiendo Conventional Commits.

### Cambiado
- **Migración de Colab a entorno local con VS Code:** código modularizado para futura extracción a `app/`.

---

## [V13.0] - 2026-04-15 (Benchmarking con modelo GPT-4o)

### Añadido
- **Benchmarking comparativo LLM vs SLM:** migración del motor de inferencia de `gpt-4o-mini` a `gpt-4o` para evaluar el impacto en la resolución de ambigüedades.

### Cambiado
- **Superación del techo cognitivo:** F1-Score pasó del ~53 % (gpt-4o-mini) al ~70.83 % (gpt-4o).
- **Validación de arquitectura:** la separación de responsabilidades (RAG + LLM + validación stock) se verificó correcta; el techo estaba en la capacidad de razonamiento del modelo pequeño.

---

## [V12.0] - 2026-04-08 (Chain of Thought y estabilización de métricas)

### Añadido
- **Zero-Shot Chain of Thought (CoT):** campo `analisis_previo` como primer elemento de la salida JSON, forzando al LLM a instanciar su razonamiento antes de emitir un SKU.
- **Prompts balanceados (Model Alignment):** mandato dual "automatizar la certeza y enrutar la ambigüedad" que estabilizó el comportamiento del modelo.

### Cambiado
- **Corrección matemática del Stress Test:** migración de `set()` a `collections.Counter()` para soportar conteo correcto de Verdaderos Positivos duplicados en canastas multi-ítem.
- **F1-Score estabilizado en ~83.87 %** con latencia de ~7.8 s/transacción.

---

## [V11.0] - 2026-04-07 (Migración de OCR Tradicional a VLM multimodal)

### Cambiado
- **Deprecated:** Tesseract OCR + OpenCV para extracción de imágenes y PDFs.
- **Nuevo:** inferencia multimodal directa con `gpt-4o` (Vision). Conversión de imagen a Base64 en memoria (`BytesIO`) sin almacenamiento en disco.

---

## [V10.0] - 2026-04-06 (Refactorización UI/UX y Progressive Disclosure)

### Añadido
- **Columna `Ítem #`** en la tabla de resultados: llave primaria visual para vincular cada fila con el dropdown HITL.
- **Wizard Layout:** reestructuración en `gr.Group` semánticos (Paso 1: Ingesta, Paso 2: Revisión, Paso 3: Auditoría HITL).

### Cambiado
- **Progressive Disclosure:** payload JSON y telemetría encapsulados en `gr.Accordion` colapsado.

---

## [V9.0] - 2026-04-05 (Jerarquía cognitiva y tolerancia a fallos)

### Añadido
- **Jerarquía de prompting:** lecciones HITL tenían prioridad sobre restricciones del catálogo RAG. *(Nota: esta prioridad fue reformulada en V3.0-notebook — OBS-05 — a "usa como referencia y verifica", no aplicación ciega.)*
- **`globals().get()` para memoria HITL:** tolerancia a fallos cuando el vectorstore no ha sido inicializado.

### Cambiado
- **Ventana de recuperación ampliada:** `k=10` → `k=15` para canastas B2B de 5-6 productos simultáneos.

---

## [V8.0] - 2026-04-05 (Orquestador DAG y prevención de errores de estado)

### Añadido
- **`OrquestadorPipeline`:** clase estática que verifica existencia de variables globales antes de permitir ejecución de módulos dependientes. Lanza `RuntimeError` semánticos ante secuencia rota.

---

## [V7.0] - 2026-04-05 (Memoria HITL dinámica y Poka-Yoke)

### Añadido
- **Memoria dinámica HITL (Estrategia 1):** motor vectorial FAISS secundario indexando `historial_correcciones_hitl.jsonl`. Lecciones recuperadas como few-shot dinámicos en el prompt principal.
- **Poka-Yoke Visual:** dropdown de corrección anclado a `df_catalogo['SKU'].unique()`. Cero errores tipográficos en el dataset de aprendizaje.
- **Recálculo transaccional en vivo:** la UI recalcula stock y razonamiento al aplicar una corrección humana.

---

## [V6.0] - 2026-04-05 (Query Rewriter — doble inferencia)

### Añadido
- **Query Rewriter (Pre-Retrieval):** etapa de pre-procesamiento con LLM que normaliza el input antes de la búsqueda.
- **Patrón de doble inferencia:** Input Crudo → Query Rewriter → Motor Híbrido → Extractor Final.

---

## [V5.0] - 2026-04-05 (Producción PoC)

### Añadido
- **`BuscadorHibridoPersonalizado`:** clase nativa FAISS + BM25 (50/50), sin dependencia de `langchain.retrievers`.
- **Evaluación basada en Counter:** métricas por canasta de pedido con frecuencia exacta.

### Eliminado
- **Text Chunking:** eliminado `RecursiveCharacterTextSplitter` — relación 1:1 (1 registro DataFrame = 1 Documento LangChain).

---

## [V4.0] - 2026-03-31

### Añadido
- **Bootstrapping dinámico del Ground Truth:** muestreo directo de `df_catalogo` en tiempo de ejecución.

### Cambiado
- **Few-Shot Prompting:** migración de Zero-Shot a Few-Shot con pares (Input → Razonamiento → JSON).

---

## [V3.0] - 2026-03-30

### Añadido
- **Filtro HITL inicial:** capa determinista que fuerza `REVISION_MANUAL` si confianza es Media/Baja.

### Cambiado
- **Búsqueda híbrida (Ensemble Retriever):** FAISS semántico + BM25 léxico 50/50.
- **Chain-of-Thought (CoT):** razonamiento explícito antes de asignar confianza.

### Eliminado
- **Text Splitting:** eliminado `RecursiveCharacterTextSplitter`.

---

*Mantenedor: Mateo Córdova · Juan Portero · MIA UDLA 2026.*
