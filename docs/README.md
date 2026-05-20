# 🏭 Copiloto IA B2B — Procesamiento Inteligente de Pedidos · **V3**

[![Versión](https://img.shields.io/badge/Versión-V3.3-7c3aed)](CHANGELOG.md)
[![Estado](https://img.shields.io/badge/Estado-Tutor--Compliant-success)](#-cobertura-de-observaciones-docentes)
[![Python](https://img.shields.io/badge/Python-3.12-blue)]()
[![Modelo](https://img.shields.io/badge/LLM-GPT--4o-orange)]()
[![F1 Identificación](https://img.shields.io/badge/F1_Id-0.7633-success)]()
[![F1 Transaccional](https://img.shields.io/badge/F1_Tx-0.3067-yellow)]()

> **Sistema de IA aplicada que automatiza la conversión de órdenes de compra B2B informales (texto libre, fotos manuscritas, PDFs y planillas Excel) en payloads JSON estructurados validados contra el catálogo ERP, mediante una arquitectura RAG híbrida (Reciprocal Rank Fusion) con Human-in-the-Loop, filtro determinístico de confianza y trazabilidad input-output completa.**

**Trabajo de Titulación — Maestría en Inteligencia Artificial Aplicada**
Universidad de las Américas (UDLA), Quito · Ecuador · 2026

**Autores:** Mateo Córdova · Juan Portero

---

## 🎯 Estado del proyecto

| Sub-Fase | Descripción | Estado |
|---|---|---|
| 1.1 | PoC en Google Colab — Notebook V1 (baseline) | ✅ Completado |
| 1.2 | Notebook V2 — Calibración del Prompt (anti over-conservatism) | ✅ Completado |
| 1.3 | Notebook V3 — Incorporación de las 11 observaciones docentes | ✅ Completado |
| 1.4 | Notebook V3.1 — UI Gradio con Producto + Detalle + Payload + Confirmar | ✅ Completado |
| 1.5 | Notebook V3.2 — Smart Load del Ablation (cache pattern + invalidation) | ✅ Completado |
| 1.6 | **Notebook V3.3 — Trazabilidad input-output (columna "Texto leído")** | ✅ **Completado** |
| 2.0 | Migración a API REST con FastAPI | 🚧 En diseño |
| 3.0 | Producción containerizada (Docker + DB + Auth) | 📋 Roadmap |

---

## 📊 Métricas oficiales V3 (Ablation Study reproducible)

Resultados sobre **50 casos sintéticos** con `seed=42`, Chaos Monkey al 50 %, configuración V3 completa (RRF + Filtro Confianza + Métrica Dual). Run ejecutado el **2026-05-15** (`metricas_oficiales_v3.json`).

### Configuración recomendada: GPT-4o + Prompt V2 + RRF + Filtros V3

| Métrica | Valor | Comentario |
|---|---|---|
| **F1 Identificación** (solo SKU) | **0.7633** | Cifra comparable con V1/V2 |
| **F1 Transaccional** (SKU + cantidad) | **0.3067** | 🆕 V3: refleja validez transaccional completa |
| **MAE cantidad** | **9.3** unidades | Error medio absoluto en cantidad |
| Aciertos perfectos (id) | 31/50 (62 %) | |
| Aciertos perfectos (tx) | 11/50 (22 %) | |
| Latencia promedio | 4.17 s | |
| Costo por pedido | $0.00850 USD | |

### Configuración alternativa: GPT-4o-mini + Prompt V2 + RRF + Filtros V3

| Métrica | Valor |
|---|---|
| **F1 Identificación** | **0.7527** |
| **F1 Transaccional** | **0.2833** |
| **MAE cantidad** | 9.92 unidades |
| Aciertos perfectos (id) | 29/50 (58 %) |
| Aciertos perfectos (tx) | 9/50 (18 %) |
| Latencia promedio | 7.42 s |
| Costo por pedido | $0.000492 USD |

### Comparativo histórico V1 → V2 → V3 (modelo gpt-4o)

| Versión | F1 Identificación | F1 Transaccional | Latencia | Costo/pedido |
|---|---|---|---|---|
| V1 (baseline) | 0.6433 | _no medido_ | 3.18 s | $0.00615 |
| V2 (calibrado) | 0.8000 | _no medido_ | 2.97 s | $0.00703 |
| **V3 (tutor-compliant)** | **0.7633** | **0.3067** 🆕 | 4.17 s | $0.00850 |

> 🎯 **Hallazgo central de V3** (anticipado por el tutor en OBS-02): la brecha de **0.4566** entre F1 Identificación y F1 Transaccional cuantifica el **costo real de errores en cantidad** sobre identificación correcta — un hallazgo invisible bajo la métrica original. El sistema identifica el SKU correcto en el 76 % de los casos, pero SKU + cantidad simultáneamente solo en el 31 %.

> 🎯 **Hallazgo Pareto V3** (no anticipado): GPT-4o es **~44 % más rápido** que GPT-4o-mini (4.17 s vs 7.42 s) siendo 16× más caro por token. En contextos B2B donde la latencia importa más que el costo unitario por pedido, GPT-4o domina en TCO real.

> Resultados reproducibles vía `metricas_oficiales.json` (V1), `metricas_oficiales_v2.json` (V2) y `metricas_oficiales_v3.json` (V3 — fuente única de verdad oficial, run 2026-05-15).

### Distribución de errores — GPT-4o V3 (n=50)

| Categoría | Casos | % |
|---|---|---|
| derivacion_correcta | 25 | 50 % |
| acierto_perfecto | 11 | 22 % |
| derivacion_innecesaria | 10 | 20 % |
| alucinacion_sku | 3 | 6 % |
| falla_no_deriva | 1 | 2 % |

---

## ✏️ Cobertura de observaciones docentes

V3 resuelve las **11 observaciones** del tutor académico documentadas en `REVISION-CAPSTONE-IA-PEDIDOS-001` (27 abril 2026):

| # | Observación | Solución V3 | Verificación |
|---|---|---|---|
| **01** | Política confianza ↔ revisión | `UMBRAL_CONFIANZA_AUTOMATICA` + filtro determinístico en `validar_inventario` | Test pytest |
| **02** | Métrica ignora cantidad | `calcular_metricas_pedido_dual()` con F1_id + F1_tx + MAE | Test pytest |
| **03** | Híbrido no es 50/50 | `BuscadorHibridoRRF` (Reciprocal Rank Fusion, Cormack et al., 2009) | Test pytest |
| **04** | Rewriter pierde números técnicos | `PROMPT_REWRITER_V2` con heurísticas léxicas + 4 ejemplos | Test pytest |
| **05** | Memoria HITL "sin cuestionar" | Umbral similitud 0.85 + top_k=3 + reformulación instrucción | Test pytest |
| 06 | Feedback HITL pierde texto VLM | Resuelto en refactor V2 — verificado en V3 | Inspección |
| 07 | HTML sin escape | Resuelto vía `gr.DataFrame` — verificado en V3 | Inspección |
| 08 | Métricas inconsistentes | Una sola fuente: `metricas_oficiales_v*.json` | Trazabilidad |
| 09 | README ↔ código desalineados | Documentación realineada (este archivo) | Trazabilidad |
| 10 | Lenguaje "bypass PII" | Reformulado bajo principio GDPR Art. 5(1)(c) | Inspección |
| 11 | Memoria vs fine-tuning | Distinción explícita + renombre archivo HITL | Inspección |

**Cobertura total: 11/11 (100 %).**

---

## 🏗️ Arquitectura de la Solución V3

### Pipeline de procesamiento (Celdas 1–13)

```
[Texto / Imagen / PDF / Excel]
       ↓
  Router Multi-formato (extensión + magic bytes)          (Celda 13)
       ↓
  Query Rewriter V2 (preserva números técnicos)           (Celda 5)
       ↓
  Buscador Híbrido RRF (FAISS 50 % + BM25 50 %)          (Celda 4)
       ↓
  LLM + CoT + Memoria HITL (umbral 0.85, top_k=3)        (Celda 5)
       ↓
  Validación V3 (5 filtros encadenados)                   (Celda 5)
       ↓
  JSON estructurado → ERP / HITL Wizard
```

### 1. Ingesta multimodal con minimización de datos (GDPR Art. 5(1)(c)) — Celdas 10–13

* **Vision-Language Model (GPT-4o Vision):** procesa imágenes y PDFs escaneados.
* **Parsing nativo (pypdf):** PDFs digitales sin pasar por VLM (costo $0).
* **Adapter Excel:** detección de columnas `cantidad`, `producto`, `unidad`, `atributos` via palabras clave; fallback a linealización si sin estructura.
* **Router multi-formato:** dispatcher central con detección por magic bytes + extensión.
* **Principio de minimización:** el prompt VLM omite información personal identificable (firmas, sellos, nombres, direcciones, RUC) — alineado con GDPR Art. 5(1)(c).

### 2. Búsqueda híbrida con Reciprocal Rank Fusion 🆕 V3 — Celda 4

```
score_RRF(d) = Σ_i  peso_i / (k + rank_i(d))
```

* **FAISS (semántico):** captura similitud conceptual (jerga del cliente).
* **BM25 (léxico):** captura coincidencias exactas (códigos, calibres, números técnicos).
* **Fusión balanceada 50/50** (`peso_faiss=0.5, peso_bm25=0.5, rrf_k=60`) — parametrizable.
* **Referencia:** Cormack et al., SIGIR'09 (2009). Estándar usado por Elasticsearch, OpenSearch, Pinecone, Vespa.

### 3. Pipeline RAG con Query Rewriter calibrado 🆕 V3 — Celda 5

* **PROMPT_REWRITER_V2:** 4 reglas léxicas + 4 ejemplos few-shot. Elimina verbos de compra y cantidades de compra; conserva calibres, tipos, dimensiones.
* **PROMPT_EXTRACTOR_V2:** Chain-of-Thought + 5 reglas jerárquicas + ejemplos de ambigüedad genuina.
* **REGLA 5 V3.3:** cada item incluye `texto_fuente` con cita exacta del fragmento del cliente.
* **Calibración de confianza:** `confianza_ia` ∈ {Alta, Media, Baja} con criterio explícito.

### 4. Filtro determinístico de confianza 🆕 V3 (resuelve OBS-01) — Celda 5

```python
# Dentro de validar_inventario — Filtro #2
if item.sku != "REVISION_MANUAL" and not confianza_es_suficiente(item.confianza_ia, umbral):
    item.sku = "REVISION_MANUAL"  # forzado por código, no por LLM
```

* **Defense-in-depth:** filtro determinístico opera sobre el output del LLM, independientemente de su razonamiento.
* **Política parametrizable:** `UMBRAL_CONFIANZA_AUTOMATICA` ∈ {Alta, Media, Baja} — configurable en `AppSettings`.

### 5. Memoria HITL con umbral de similitud 🆕 V3 (resuelve OBS-05) — Celda 4

* **Top-k = 3:** recupera múltiples lecciones para que el LLM tenga contexto.
* **Umbral de similitud = 0.85:** conversión L2→similitud `1/(1+dist)`. Filtra lecciones débilmente relacionadas.
* **Instrucción reformulada:** *"usa como referencia y verifica que aplique"* (antes: *"úsala sin cuestionar"*).
* **Persistencia:** `historial_correcciones_hitl.jsonl` (renombrado de `dataset_finetuning_hitl.jsonl`, OBS-11).

### 6. Métricas duales y categorización de errores 🆕 V3 (resuelve OBS-02) — Celda 8

* **F1 Identificación:** evalúa solo aciertos de SKU (compatible con V1/V2).
* **F1 Transaccional:** evalúa SKU + cantidad simultáneamente.
* **MAE cantidad:** error medio absoluto en unidades por SKU correctamente identificado.
* **Categorización automática:** 7 categorías (`acierto_perfecto`, `acierto_solo_sku`, `derivacion_correcta`, `derivacion_innecesaria`, `alucinacion_sku`, `falla_no_deriva`, `no_detecta_items`).

---

## 🎨 Mejoras avanzadas V3.1, V3.2 y V3.3

### V3.1 — UI Gradio enriquecida (Celdas 20A–20C)

* **Columna "Producto":** lookup del `Nombre_Tecnico` en `df_catalogo` junto al SKU.
* **Detalle sin truncar:** razonamiento completo del LLM con `wrap=True` en `gr.DataFrame`.
* **Accordion "📦 Payload JSON":** componente colapsable con `gr.Code(language="json")`.
* **Botón "📤 Confirmar Pedido y Generar JSON":** genera `pedido_<timestamp>.json` en `/content/pedidos_confirmados/`. Rechaza si quedan ítems con `REVISION_MANUAL` sin resolver (fail-safe).

### V3.2 — Smart Load del Ablation (Cache pattern) — Celda 15

* **Tres modos:** `"auto"` (default), `"force_recompute"`, `"load_only"`.
* **Validación estructural:** campos `metadata`/`runs`, modelos esperados, métricas duales (`f1_id_macro`, `f1_tx_macro`).
* **Validación de consistencia:** `UMBRAL_CONFIANZA`, pesos retriever, umbral HITL, n_casos.
* **Reduce costo de re-ejecución de ~$0.40 a $0.00 USD** cuando los resultados ya son válidos.

### V3.3 — Trazabilidad input-output 🆕 (Celdas 2, 5)

* **Campo `texto_fuente: Optional[str]` en `PedidoIA`:** fragmento exacto del pedido que generó cada ítem.
* **REGLA 5 en PROMPT_EXTRACTOR_V2:** instrucción al LLM con dos ejemplos few-shot.
* **Función `_extraer_fragmento_fuente_fallback()`:** localiza la cantidad numérica en el texto usando regex; extrae desde 5 caracteres antes hasta 60 caracteres después del match. Si no hay match literal, devuelve el texto completo con prefijo `[aprox.]`.
* **Columna "📝 Texto leído"** en la tabla Gradio: nunca queda vacía (3 escenarios: LLM emite / fallback exitoso / aproximación).

---

## ⚙️ Estructura del Pipeline V3.3

```
┌─────────────────────────────────────────────────────────────┐
│            ENTRADA MULTIMODAL                               │
│  Texto │ Imagen (VLM+GDPR) │ PDF (nativo+VLM) │ Excel      │
└──────────────────┬──────────────────────────────────────────┘
                   ▼
        Router Multi-formato (extensión + magic bytes)   [Celda 13]
                   ▼
        Query Rewriter V2 (preserva números técnicos)    [Celda 5]
                   ▼
        Buscador Híbrido RRF (FAISS 50 % + BM25 50 %)   [Celda 4]
                   ▼
        LLM + CoT + Memoria HITL (umbral 0.85, top_k=3) [Celda 5]
                   ▼
                   + texto_fuente (V3.3 — trazabilidad)
                   ▼
   ┌──────────────────────────────────────────────────┐
   │  Validación V3 (5 filtros en orden):             │
   │   1. Schema Pydantic                             │
   │   2. 🛡️ Filtro determinístico de confianza      │
   │   3. Ambigüedad marcada por LLM (REVISION_MANUAL)│
   │   4. Anti-alucinación (existencia en catálogo)   │
   │   5. Validación de stock                         │
   └─────────┬──────────────────────────────┬─────────┘
             ▼                              ▼
       Pedidos claros → ERP        Ambiguos → HITL Wizard  [Celdas 20A-C]
                                        ▼
                               Memoria HITL (umbral 0.85)
                                        ▼
                   Confirmar pedido → JSON downstream
                       (con texto_fuente trazable)
```

---

## 🚀 Quick Start

### Requisitos

- Python 3.12.x
- Conda (recomendado) o venv
- API Key de OpenAI (Tier 1+ recomendado para evitar rate limits)
- Saldo OpenAI ≥ $1 USD para reproducir el ablation completo (~$0.40 USD por run)

### Setup local

```bash
# 1. Clonar el repositorio
git clone https://github.com/viejomatius/pedidos_ia_tesis_mia.git
cd pedidos_ia_tesis_mia

# 2. Crear y activar el ambiente Conda
conda create -n copiloto-b2b python=3.12 -y
conda activate copiloto-b2b

# 3. Instalar dependencias
pip install -r requirements.txt

# 4. Configurar variables de entorno
cp .env.example .env
# Editar .env y agregar tu OPENAI_API_KEY

# 5. (Opcional) Ejecutar el notebook V3.3 en Colab
#    Subir main_procesamiento_pedidos_ia_V3_FULL_v33.ipynb a Google Colab
#    y ejecutar celdas en orden secuencial
```

### Reproducir las métricas oficiales V3

```bash
# Desde el notebook V3.3 en Colab:
# 1. Ejecutar Celdas 1-14 (pipeline base + suite de 8 tests)
# 2. Ejecutar Celda 15 (smart load del ablation — carga desde disco si ya existe, $0)
# 3. Ejecutar Celda 16 (comparativo V1→V2→V3 + figura publicable)
# 4. Descargar metricas_oficiales_v3.json del panel /content/
```

> 💾 **Smart Load V3.2 (Celda 15):** detecta si ya existe un archivo válido y consistente con la configuración actual. Reduce el costo de re-ejecución de ~$0.40 a $0.00 USD.

---

## 🛠️ Stack tecnológico V3

| Capa | Tecnología | Versión |
|---|---|---|
| LLM principal | OpenAI GPT-4o | last available |
| LLM ligero (ablation) | OpenAI GPT-4o-mini | last available |
| VLM | OpenAI GPT-4o Vision | last available |
| Embeddings | text-embedding-3-small | last available |
| Vector store | FAISS (CPU) | ≥1.8 |
| Léxico | rank-bm25 | ≥0.2.2 |
| Validación | Pydantic v2 | ≥2.5,<3.0 |
| Orquestación | LangChain | 0.3.27 (pinned) |
| UI | Gradio | ≥4.40 |
| Tests | pytest-style integrados (8 tests) | nativo Python |
| Visualización | Matplotlib | [PENDIENTE - verificar versión] |
| Backup | Google Drive API | nativo Colab |

> Versiones completas con pin exacto en `requirements.txt`. Los paquetes de sistema (`tesseract-ocr`, `poppler-utils`) se instalan vía `apt-get` en Celda 1.

---

## 📁 Archivos producidos por el notebook V3.3

```
/content/                                  (entorno Colab)
├── catalogo_erp_ideal.csv                 (150 SKUs sintéticos — Celda 3)
├── historial_correcciones_hitl.jsonl      (memoria HITL — Celda 4, escritura incremental)
├── metricas_oficiales.json                (V1 baseline — subir manualmente)
├── metricas_oficiales_v2.json             (V2 calibrado — subir manualmente)
├── metricas_oficiales_v3.json             (V3 tutor-compliant — OFICIAL, Celda 15)
├── demo_pedido_imagen.jpg                 (demo defensa — Celda 18)
├── demo_pedido_nativo.pdf                 (demo defensa — Celda 18)
├── pedidos_confirmados/
│   └── pedido_<timestamp>.json            (pedidos confirmados con texto_fuente — Celda 20A)
└── figuras_tesis/
    ├── figura_01_comparativo_f1_v3.png    (F1 6 configuraciones — Celda 19A)
    ├── figura_02_iteracion_v1_v2_v3.png   (Evolución + F1 transaccional — Celda 19A)
    └── [figuras 03-05 — PENDIENTE - verificar Celda 19B]
```

> **Nota:** `metricas_oficiales.json` (V1) y `metricas_oficiales_v2.json` (V2) no se generan en este notebook — deben subirse manualmente al panel `/content/` de Colab antes de ejecutar Celda 9 (carga histórica) y Celda 16 (comparativo).

---

## 📊 Métricas objetivo (KPIs)

* **Métricas técnicas:**
  * **F1 Identificación:** acierto de SKU (compatible V1/V2).
  * **F1 Transaccional:** acierto de SKU + cantidad simultáneamente (V3).
  * **MAE cantidad:** error medio absoluto en unidades.
  * **Distribución de errores:** categorización automática en 7 categorías.
* **Precision-Recall Trade-Off:** sistema calibrado hacia **Alta Precisión** mediante filtro determinístico de confianza. Falsos negativos (revisión humana extra) son tolerables; falsos positivos (alucinación logística) son críticos.
* **Métricas de negocio:**
  * **Lead Time Operativo:** ~4.2 s/pedido (gpt-4o) vs procesamiento manual de minutos.
  * **TCO estimado:** [PENDIENTE - verificar cálculo con datos actuales]
* **Integridad de datos:** `gr.DataFrame` con escape automático + dropdown HITL anclado al catálogo (Poka-Yoke).
* **Trazabilidad** 🆕 V3.3: cada ítem del payload incluye el fragmento del texto del cliente que lo generó (`texto_fuente`).

---

## 🛡️ Resiliencia y observabilidad V3

| Capa | Mecanismo | Documentado en |
|---|---|---|
| **Errores schema** | Pydantic v2 + fallback a REVISION_MANUAL | `validar_inventario` (Celda 5) |
| **Alucinaciones** | Anti-alucinación (verificación existencia SKU en catálogo) | `validar_inventario` (Celda 5) |
| **Confianza débil** | 🆕 V3: filtro determinístico parametrizable | `confianza_es_suficiente()` (Celda 2) |
| **Persistencia** | Backup a Google Drive | Celda 21 |
| **Recuperación parcial** | JSON oficial guardado tras cada modelo evaluado | `ejecutar_ablation_v3` (Celda 15) |
| **Smart Load** | 🆕 V3.2: cache pattern con cache invalidation | `cargar_o_ejecutar_ablation_v3` (Celda 15) |
| **Trazabilidad UI** | 🆕 V3.3: columna "Texto leído" con fallback heurístico | `_extraer_fragmento_fuente_fallback` (Celda 5) |
| **Rate limits** | [PENDIENTE - verificar si exponential back-off está implementado en notebook] | — |

---

## 🤝 Estrategia de aprendizaje (distinción OBS-11)

El sistema implementa **aprendizaje dinámico por memoria externa**, NO fine-tuning de pesos:

> **Lo que SÍ hace V3:** las correcciones humanas se guardan en `historial_correcciones_hitl.jsonl`, se vectorizan en un índice FAISS secundario, y se recuperan como **contexto few-shot** durante futuras inferencias (In-Context Learning con umbral de similitud 0.85).

> **Lo que NO hace V3:** modificar pesos del modelo GPT.

> ⚠️ **Roadmap a futuro (Estrategia 2):** cuando `historial_correcciones_hitl.jsonl` supere los 1,000 registros validados y curados, se implementará **fine-tuning supervisado** sobre modelos fundacionales eficientes para reducir costo unitario y dependencia de proveedor.

---

## 📚 Documentación adicional

* [`CHANGELOG.md`](CHANGELOG.md) — historial técnico completo.
* [`diario_tesis.md`](diario_tesis.md) — bitácora de razonamiento científico y experimentos.
* `figuras_tesis/` — figuras publicables en alta resolución (300 DPI).

---

## 👥 Autores

**Mateo Córdova** · **Juan Portero**
Maestría en Inteligencia Artificial Aplicada — UDLA, Quito · 2026

---

## 📄 Licencia

Proyecto académico. Todos los derechos reservados a los autores y a la Universidad de las Américas (UDLA).

---

*Desarrollado bajo los principios de Privacy by Design del GDPR Art. 5(1)(c) (minimización de datos).*
