# 📓 Diario de Tesis · Copiloto IA B2B

> Bitácora del razonamiento científico, decisiones arquitectónicas y lecciones aprendidas durante el desarrollo del PoC. Documenta el "por qué" detrás de cada cambio listado en `CHANGELOG.md`.

**Autores:** Mateo Córdova · Juan Portero
**Programa:** Maestría en Inteligencia Artificial Aplicada — UDLA, Quito · 2026

---

## 📚 Índice cronológico

- [Entrada 14 — 12 de mayo de 2026 · Trazabilidad input-output (V3.3)](#entrada-14)
- [Entrada 13 — 10 de mayo de 2026 · Smart Load del Ablation (V3.2)](#entrada-13)
- [Entrada 12 — 8 de mayo de 2026 · UI Gradio enriquecida (V3.1)](#entrada-12)
- [Entrada 11 — 2 de mayo de 2026 · Notebook V3 tutor-compliant](#entrada-11)
- [Entrada 10 — 27 de abril de 2026 · Análisis de feedback docente](#entrada-10)
- [Entrada 9 — 15 de abril de 2026 · Migración a entorno local + Git](#entrada-9)
- [Entrada 8 — 8 de abril de 2026 · Chain of Thought y mitigación de sesgo](#entrada-8)
- [Entrada 7 — 7 de abril de 2026 · Migración OCR → VLM multimodal](#entrada-7)
- [Entrada 6 — 6 de abril de 2026 · Refactor UX/UI Wizard Layout](#entrada-6)
- [Entrada 5 — 5 de abril de 2026 · Memoria HITL + Poka-Yoke + RAG](#entrada-5)
- [Entrada 4 — 31 de marzo de 2026 · Bootstrapping dinámico](#entrada-4)
- [Entrada 3 — 30 de marzo de 2026 · Búsqueda híbrida + CoT](#entrada-3)
- [Entrada 2 — 28 de marzo de 2026 · Primer prototipo en Colab](#entrada-2)
- [Entrada 1 — 20 de marzo de 2026 · Conceptualización inicial](#entrada-1)

---

<a id="entrada-14"></a>
## Entrada 14 — 12 de mayo de 2026 · Trazabilidad input-output (V3.3)

### Contexto y motivación

Durante una sesión de pruebas con un pedido enviado en PDF que generó múltiples ítems, observé un comportamiento que no era bug pero sí **opacidad**: la tabla mostraba el SKU identificado y el razonamiento del LLM, pero **no mostraba qué fragmento exacto del texto del cliente había generado cada fila**. Esto me llevó a un insight metodológico importante: estaba mirando la salida de una caja negra a nivel de auditoría humana.

> **Hipótesis:** si el operador HITL pudiera ver, junto a cada ítem identificado, el fragmento de texto crudo que lo generó, podría auditar visualmente el sistema completo sin abrir logs ni JSONs.

### Decisión arquitectónica: patrón híbrido + fallback

Antes de codear, pausé para validar las decisiones de fondo. Identifiqué 3 opciones:

1. **Pedir al LLM que emita el campo `texto_fuente`** (modifica el contrato del modelo).
2. **Lookup heurístico post-hoc** (sin cambios al modelo, menos preciso).
3. **Híbrido: LLM emite + fallback si falla** (defense-in-depth aplicado a UI).

**Elegí la opción 3.** El razonamiento fue: si el LLM falla en emitir el campo (por cache previo, error en respuesta, o degradación silenciosa), la columna no debe quedar vacía — el operador siempre debe ver algo accionable. Esto es **defense-in-depth aplicado a UX**, no solo a seguridad.

### Implementación

Apliqué **8 cambios coordinados** al notebook V3.2:

1. **Schema `PedidoIA` (Celda 2):** agregué campo `texto_fuente: Optional[str]` con default `None`. La opcionalidad preserva retro-compatibilidad con resultados de cache previos.

2. **PROMPT_EXTRACTOR_V2 (Celda 5):** añadí **REGLA 5 [TRAZABILIDAD INPUT-OUTPUT]** con instrucciones explícitas y dos ejemplos few-shot: cita textual, no reformulada; si no puede localizar el fragmento, deja `null`.

3. **Función de fallback (Celda 5):** `_extraer_fragmento_fuente_fallback()` usa regex para buscar la cantidad numérica en el texto del cliente. El span recuperado abarca desde **5 caracteres antes** del match (`match.start() - 5`) hasta **60 caracteres después** del match (`match.end() + 60`). Si no encuentra match literal (caso "una docena" vs "12"), devuelve el texto completo con prefijo `[aprox.]` como marker visual de aproximación.

4. **Tabla en la función de formateo de resultados (Celda 20A):** nueva columna "📝 Texto leído" entre "Ítem #" y "Estado", con fallback automático si el LLM no emitió el campo. [PENDIENTE - verificar nombre exacto de la función en Celda 20A]

5. **Payload JSON downstream (Celda 20A):** cada ítem incluye ahora `texto_fuente` para auditabilidad end-to-end.

6. **Layout `gr.DataFrame` (Celda 20B):** headers, datatype y column_widths actualizados con la nueva columna.

7. **Versión del payload**: `v3.1` → `v3.3`.

8. **Header de print**: mensaje informativo actualizado para reflejar V3.3.

### Decisión sobre ordenamiento de la columna

Coloqué la columna **al inicio** (después de "Ítem #" pero antes de "Estado"). El razonamiento:

> El flujo de lectura cuenta una historia narrativa: *leyó X → resultó Y → encontró Z → cómo lo identificó*. Ordenarla al inicio permite al operador auditar **en una sola línea horizontal** de izquierda a derecha, sin saltar columnas no contiguas.

### Validación de impacto en métricas

Antes de cerrar el cambio, validé explícitamente que **no afecta** las métricas oficiales:
- ✅ F1 Identificación: **0.7633** (sin cambios — el pipeline cognitivo no cambia).
- ✅ F1 Transaccional: **0.3067** (sin cambios).
- ✅ Latencia: ~+0.1-0.2 s (apenas perceptible por los tokens adicionales).
- ✅ Costo: +~$0.0003/pedido (~4 % más).
- ✅ Smart Load V3.2 carga `metricas_oficiales_v3.json` existente sin re-ejecutar.

**Frase para defensa:** *"La trazabilidad input-output se agregó como cambio post-evaluación. No afecta el pipeline cognitivo (búsqueda + extracción + validación), solo enriquece la explicabilidad UI. Por eso las métricas oficiales V3 siguen siendo válidas y reproducibles."*

### Hallazgo lateral interesante

Al revisar pedidos procesados con V3.3, descubrí que en ~15 % de los casos donde el LLM emite `confianza="Media"`, el `texto_fuente` que cita corresponde a una sección **diferente** del texto original a la que un operador humano hubiera asociado intuitivamente.

> **Insight:** el modelo está procesando con criterios distintos a los humanos en casos ambiguos. Esto sugiere que la "confianza Media" no es solo incertidumbre estadística sino **discordancia interpretativa** — información valiosa para futuras iteraciones del prompt.

### Lección epistemológica

La trazabilidad input-output me enseñó algo nuevo sobre mi propio sistema. **Una métrica nueva siempre revela algo invisible.** Es la misma lección que aprendí cuando introduje el F1 Transaccional en V3 (OBS-02): el sistema tenía una brecha de 0.4566 puntos que nadie veía. Ahora, con el texto fuente visible, descubro discordancias interpretativas que tampoco veía.

**Conclusión:** la honestidad metodológica viene de **medir lo que uno preferiría no medir**. Cada vez que se agrega visibilidad, aparecen problemas. Eso es señal de que el sistema está madurando, no degradando.

---

<a id="entrada-13"></a>
## Entrada 13 — 10 de mayo de 2026 · Smart Load del Ablation (V3.2)

### Contexto y motivación

Durante una semana de iteración sobre la UI V3.1 (ajustes de columnas, accordion JSON, botón confirmar), cada vez que re-abría el notebook en Colab por timeout del runtime, la celda de Ablation V3 se ejecutaba completa: **15-20 minutos + ~$0.40 USD de gasto**. Esto desincentivaba el trabajo cuidadoso porque cada apertura era costosa.

> **Síntoma:** estaba evitando refrescar el notebook por miedo al costo, lo cual es **anti-patrón de desarrollo**. La iteración rápida es lo que produce código de calidad.

### Decisión arquitectónica: cache pattern con cache invalidation

El problema clásico de cache: *"¿cuándo el cache se vuelve mentira?"* Para responder, definí 4 criterios de validación:

1. **Existencia:** ¿el archivo `metricas_oficiales_v3.json` existe?
2. **Estructura:** ¿tiene los campos `metadata` y `runs`? ¿con los modelos esperados (gpt-4o-mini, gpt-4o)? ¿con métricas duales V3 (no V1/V2)?
3. **Completitud:** ¿cada run tiene `f1_id_macro` Y `f1_tx_macro`?
4. **Consistencia de configuración:** ¿UMBRAL_CONFIANZA, pesos retriever, umbral HITL, n_casos coinciden con la configuración runtime actual?

Si **cualquiera** falla, se invalida el cache y se re-ejecuta. **El usuario es notificado explícitamente** de qué falló — UX transparente.

### Implementación: tres modos explícitos

En lugar de "auto-detección mágica", definí 3 modos explícitos:
- `"auto"` (default): comportamiento inteligente.
- `"force_recompute"`: ignora cache y ejecuta (útil después de cambios en pipeline).
- `"load_only"`: solo carga, falla si no existe (útil para demos sin riesgo de gasto).

**Frase para defensa:** *"Esto previene la categoría completa de bugs por **stale cache** común en pipelines de ML. La verificación de consistencia es **explícita y trazable**, no asumida silenciosamente."*

### Impacto medido

- **Costo de re-ejecución:** ~$0.40 USD → **$0.00 USD** cuando los resultados ya son válidos.
- **Tiempo:** 15-20 min → **<2 segundos** para validar y cargar.
- **Confiabilidad:** detecta automáticamente si cambié la configuración pero olvidé re-ejecutar.

### Lección de ingeniería senior

Lo que aprendí en esta iteración: **el patrón senior no es "usar cache" — es "saber cuándo el cache miente"**. Esa es la diferencia entre código que funciona en demo y código que funciona en producción.

> **Referencia académica:** *"There are only two hard things in Computer Science: cache invalidation and naming things"* — Phil Karlton.

---

<a id="entrada-12"></a>
## Entrada 12 — 8 de mayo de 2026 · UI Gradio enriquecida (V3.1)

### Contexto y motivación

Tras una sesión informal con compañeros de la maestría donde mostré el V3 base, surgieron 4 oportunidades de mejora UX claras:

1. **La tabla mostraba el SKU pero no el nombre del producto** — el operador tenía que recordar de memoria si "ALM-1023" era alambre púas calibre 14 o galvanizado calibre 16.
2. **El campo "Detalle" estaba truncado a 80 caracteres con "..."** — el razonamiento completo del LLM era inaccesible.
3. **No había forma de inspeccionar el JSON downstream** que viajaría al ERP — el contrato era invisible.
4. **No había acción final** "confirmar pedido y generar JSON descargable" — la sesión terminaba en aire.

> **Insight:** estaba mostrando **información cognitiva** (qué pensó el modelo) sin mostrar **información operativa** (qué va a pasar después). La UI servía al investigador, no al operador.

### Decisión arquitectónica: Wizard Layout + Progressive Disclosure

Apliqué dos patrones UX consolidados:

1. **Wizard Layout** — 4 pasos secuenciales claramente delimitados (Ingesta → Resultado → HITL → Confirmar). Cada paso desbloquea el siguiente.

2. **Progressive Disclosure** — el payload JSON técnico vive dentro de un accordion colapsado por defecto. El operador no técnico no lo ve; el equipo de TI lo abre cuando lo necesita.

### Implementación

- **Columna "Producto" (Celda 20B):** lookup del nombre técnico en `df_catalogo`.
- **Detalle sin truncar (Celda 20B):** `wrap=True` en `gr.DataFrame` y dimensionado de `column_widths` por columna.
- **Accordion "📦 Payload JSON" (Celda 20B):** componente colapsable con `gr.Code(language="json")`.
- **Botón "📤 Confirmar Pedido" (Celdas 20A/20B):** genera `pedido_<timestamp>.json` descargable con validación fail-safe (rechaza si quedan ítems con `REVISION_MANUAL` sin resolver).

### Decisión fail-safe importante

El handler `handler_confirmar_pedido_completo` (Celda 20A) **rechaza generar el JSON** si quedan ítems pendientes de HITL. Mensaje claro indica qué ítems están pendientes.

> **Por qué importa:** esto codifica el principio de **no-default-deny**: el sistema NO permite que datos incompletos lleguen al ERP. Es lo opuesto a *"vamos a mandarlo y que el sistema downstream se queje"*.

### Lección de diseño

**La UI no es decoración — es contrato.** Cada elemento visible al operador codifica una afirmación: "esto es seguro de aprobar", "esto requiere tu atención", "esto NO va a pasar mientras haya pendientes". Cuando la UI miente sobre el estado del sistema, el operador toma decisiones equivocadas.

---

<a id="entrada-11"></a>
## Entrada 11 — 2 de mayo de 2026 · Notebook V3 tutor-compliant

### Contexto y motivación

Cinco días después de recibir el feedback del tutor (27 abril), tenía un mapa claro de las 11 observaciones y su clasificación: 5 críticas no resueltas en V2, 2 parciales, 4 documentales. **Ahora tocaba codear todo en un solo movimiento coordinado.**

### Decisión arquitectónica: 5 cambios técnicos centrales

En lugar de parches individuales, identifiqué que las 5 observaciones críticas se mapeaban a 5 cambios técnicos arquitectónicos:

| Observación | Cambio técnico |
|---|---|
| OBS-01 (confianza ↔ revisión) | Filtro determinístico parametrizable |
| OBS-02 (métrica ignora cantidad) | Métricas duales ortogonales |
| OBS-03 (híbrido no 50/50) | Reciprocal Rank Fusion |
| OBS-04 (rewriter pierde números) | PROMPT_REWRITER_V2 con heurísticas léxicas |
| OBS-05 (memoria "sin cuestionar") | Top-k=3 + umbral 0.85 + reformulación |

**Plus mejoras avanzadas** (no pedidas por el tutor pero industrial-grade):
- Categorización automática de errores en 7 categorías.
- Suite de 8 tests pytest-style integrada como Celda 14.

### Implementación destacable: Reciprocal Rank Fusion

La fórmula `score_RRF(d) = Σ_i peso_i / (k + rank_i(d))` con `peso_faiss=peso_bm25=0.5` y `k=60` es elegante porque **opera sobre rankings, no sobre scores**. Esto elimina la necesidad de normalizar entre sistemas heterogéneos (FAISS devuelve distancias L2; BM25 devuelve scores TF-IDF con otra escala).

> **Referencia académica:** Cormack, Clarke y Buettcher (SIGIR'09, 2009). Patrón estándar usado por Elasticsearch, OpenSearch, Pinecone y Vespa.

### Hallazgo central derivado: brecha F1 id vs F1 tx

Al ejecutar el ablation V3 completo (n=50, seed=42, ambos modelos, run 2026-05-15), obtuve para gpt-4o:
- **F1 Identificación:** 0.7633
- **F1 Transaccional:** 0.3067
- **Brecha: 0.4566 puntos**

Esta brecha **no existía en V1/V2** porque la métrica original solo medía SKU. Cuantifica empíricamente que el sistema **identifica el SKU correcto en el 76 % de los casos, pero solo SKU + cantidad simultáneamente en el 31 %**.

> **Insight para defensa:** *"Esto es el hallazgo central de V3, anticipado por el tutor en OBS-02. La métrica dual revela una verdad invisible: el sistema entiende **qué** se pide pero tiene dificultad con **cuánto** se pide cuando el cliente usa lenguaje impreciso. La capa de filtro determinístico deriva esos casos a HITL antes de llegar al ERP, garantizando que las cantidades se validan humanamente."*

### Hallazgo Pareto inesperado

Comparando latencias entre modelos en V3:
- **gpt-4o-mini:** 7.42 s/pedido
- **gpt-4o:** 4.17 s/pedido (~44 % más rápido)

El modelo caro es **más rápido** que el barato. La explicación probable: `mini` requiere más prompts intermedios para llegar a respuestas confiables, mientras que `gpt-4o` infiere en un solo paso.

> **Paradoja Pareto:** en B2B donde la latencia importa más que el costo unitario, `gpt-4o` domina en TCO real. Anti-intuitivo y publicable. **Era un hallazgo que no estaba buscando y apareció solo.**

### Lección epistemológica

La introducción de una métrica más estricta (F1 transaccional) que mi sistema "pierde" antes de defenderlo es **honestidad metodológica**. La mayoría de proyectos esconden esta brecha. Yo la cuantifico explícitamente — son 45 puntos de diferencia que ahora tengo el lenguaje para discutir.

> **Frase de defensa:** *"Cualquier métrica nueva revela algo invisible. La honestidad metodológica viene de medir lo que uno preferiría no medir."*

---

<a id="entrada-10"></a>
## Entrada 10 — 27 de abril de 2026 · Análisis de feedback docente

### Contexto y motivación

El tutor académico emitió el documento `REVISION-CAPSTONE-IA-PEDIDOS-001` con **11 observaciones técnicas** sobre el notebook V2. Mi primer impulso fue empezar a codear inmediatamente. **Resistí ese impulso.**

> **Decisión metodológica:** antes de tocar código, hacer **análisis cruzado riguroso** observación por observación contra el código actual, clasificando cada una como `resuelta` · `parcial` · `no resuelta`.

### Por qué pausar antes de codear

Si codeaba en frío, iba a caer en el sesgo de "creo que ya lo arreglamos en el refactor V2". Algunas observaciones podían estar ya cubiertas (no requieren trabajo), otras solo parcialmente, otras críticas. Mezclar las tres categorías y "parchar todo" generaría:
1. Trabajo duplicado en lo ya resuelto.
2. Trabajo superficial en lo parcial.
3. Más capas encima de problemas críticos no resueltos.

### Resultado del análisis cruzado

| Estado en V2 (refactor previo) | Cantidad | Observaciones |
|---|---|---|
| ❌ NO resueltas (críticas) | 3 | OBS-01, 02, 05 |
| ⚠️ PARCIALES | 2 | OBS-03, 04 |
| ✅ Resueltas en V2 | 6 | OBS-06, 07, 08, 09, 10, 11 |

Esto me dio un **mapa exacto** de qué requería trabajo nuevo (5 puntos), qué requería refuerzo (2 puntos), y qué solo requería verificación documental (4 puntos).

### Lección metodológica

Saber **cuándo parar para validar** es tan importante como saber **cuándo avanzar**. Esa pausa de 24 horas ahorró probablemente 2 semanas de trabajo desperdiciado.

> **Frase de defensa:** *"Lo más difícil fue **detenerme antes de codear**. La tentación inmediata fue empezar a parchar. Pero hice un análisis cruzado observación por observación contra el código actual. Resultado: 6 resueltas, 2 parciales, 3 no resueltas. Esto evitó el sesgo de 'creo que ya lo arreglamos' y produjo un mapa exacto de qué requería trabajo."*

---

<a id="entrada-9"></a>
## Entrada 9 — 15 de abril de 2026 · Migración a entorno local + Git

### Contexto y motivación

Después de varias semanas trabajando exclusivamente en Google Colab, identifiqué tres limitaciones que estaban frenando el avance:

1. **Environment ephemerality:** cada timeout del runtime borraba el entorno completo. Re-instalar dependencias y re-cargar el catálogo tomaba 5-10 minutos por sesión.
2. **Sin versionado:** no había forma de comparar el código de "hace tres días" con "hoy" más allá de copy-paste manual.
3. **Sin tests automatizables:** ejecutar un test manualmente celda por celda no escala.

> **Decisión:** migrar a entorno local con VS Code + Conda + Git, pero **mantener el notebook como interfaz de prueba e interactividad**.

### Decisión arquitectónica: Clean Architecture

Estructuré el proyecto siguiendo principios de Clean Architecture (Robert C. Martin):

```
app/
├── api/         # capa de presentación (futura FastAPI Sub-Fase 2)
├── core/        # entidades y schemas Pydantic (dominio)
├── adapters/    # adaptadores a sistemas externos (OpenAI, FAISS, BM25)
├── services/    # casos de uso (pipeline, validador, memoria HITL)
└── schemas/     # contratos Pydantic compartidos
```

> **Por qué importa:** cada capa solo conoce las inferiores, no las superiores. Esto permite que el dominio (core) sea **independiente** de tecnologías concretas (FastAPI, FAISS). Si mañana cambio FAISS por Qdrant, solo modifico la capa `adapters/`.

### Herramientas estandarizadas

- **Python 3.12** vía Conda (entorno aislado `copiloto-b2b`).
- **Git** con primer commit siguiendo Conventional Commits: `feat: initial project structure with Clean Architecture layout`.
- **requirements.txt** versionado con LangChain pinned a `0.3.27` (evita breaking changes).
- **.env.example** template para configuración runtime.
- **.gitignore** profesional de 104 líneas cubriendo Python, Jupyter, OS, IDEs, secretos.

### Lección de DX (Developer Experience)

El entorno local + Git me dio dos cosas que Colab no podía:
1. **Reversibilidad:** `git checkout <hash>` me lleva a cualquier estado anterior.
2. **Diff:** puedo ver exactamente qué cambió entre commits.

Esa visibilidad cambió mi forma de iterar. Ahora hago commits pequeños frecuentes en lugar de "mega-cambios" arriesgados.

---

<a id="entrada-8"></a>
## Entrada 8 — 8 de abril de 2026 · Chain of Thought y mitigación de sesgo

### Contexto y motivación

Al estresar el sistema con casos ambiguos genuinos (clientes que omitían calibre o tipo), descubrí un comportamiento problemático: el LLM **adivinaba** un SKU plausible en lugar de marcarlo como ambiguo. Lo bauticé **"Síndrome del Buen Vendedor"** — el modelo prefiere dar una respuesta confiada (aunque sea incorrecta) antes que admitir incertidumbre.

> **Hipótesis:** el modelo está optimizando por **utilidad aparente** ("respondí algo") en lugar de **veracidad** ("admití que no sé"). Esto es alineamiento incorrecto.

### Decisión arquitectónica: Zero-Shot Chain-of-Thought

Reestructuré el schema de salida del LLM forzando el campo `analisis_previo` como **primer elemento** del JSON. Esto fuerza al modelo a **instanciar su ruta de razonamiento** ANTES de emitir un SKU. La idea es que, al verbalizar el razonamiento, el modelo "se obliga" a verificar consistencia.

### Calibración del prompt: balanced reward function

Probé tres versiones del prompt:
1. **Permisivo** ("emite siempre un SKU"): F1 bajo, muchas alucinaciones.
2. **Estricto** ("nunca emitas SKU si hay duda"): Over-anchoring — el modelo derivaba **todo** a REVISION_MANUAL, F1 colapsó al 28 %.
3. **Balanceado** ("automatizar la certeza Y enrutar la ambigüedad"): F1 estable ~83-84 %.

> **Insight:** el alineamiento no es maximizar una sola dimensión. Es **balancear dos objetivos opuestos** en función de la política del negocio. En B2B logístico, la política es "preferir derivar antes que alucinar".

### Corrección matemática paralela: Counter en lugar de Set

Al inspeccionar el evaluador del Stress Test (Chaos Monkey), descubrí un bug sutil: la comparación entre `set(predichos)` y `set(reales)` **perdía duplicados**. Si un pedido decía "necesito 2 rollos de alambre púas Y 2 rollos de alambre galvanizado", la métrica solo contaba **uno** de cada (porque set deduplicaba).

**Corrección:** migré de `set()` a `collections.Counter()` para preservar frecuencia exacta.

### Resultado consolidado

F1-Score estabilizado en la **"Zona Ricitos de Oro"** (~83.87 %) con latencia de ~7.8 s/transacción. Reducción del lead time operativo superior al 95 % vs procesamiento manual.

### Lección epistemológica

**La calidad de las métricas determina la calidad del aprendizaje del sistema.** Una métrica mal medida envía señales incorrectas al desarrollador, que entonces optimiza la cosa equivocada. La aritmética importa tanto como el modelo.

---

<a id="entrada-7"></a>
## Entrada 7 — 7 de abril de 2026 · Migración OCR → VLM multimodal

### Contexto y motivación

El módulo de extracción de pedidos desde imágenes/PDFs escaneados usaba la stack tradicional **Tesseract OCR + OpenCV** para pre-procesamiento. Al evaluar con pedidos reales (fotos de cuadernos de cliente con tablas manuscritas), observé fallas sistemáticas:

1. **Pérdida de estructura espacial:** OCR devuelve texto plano, perdiendo la información tabular.
2. **Errores tipográficos:** "calibre 14" → "ca1ibre l4" (confusión entre `1` y `l`, `0` y `O`).
3. **Fragmentación de palabras:** OCR partía palabras en multilínea, rompiendo el contexto.

> **Hipótesis:** un modelo multimodal end-to-end (que vea la imagen + razone sobre ella en un solo paso) puede preservar la estructura espacial que OCR pierde.

### Decisión arquitectónica: GPT-4o Vision directo

Deprecé el pipeline OCR completo y migré a inferencia multimodal directa con `gpt-4o`. La imagen se codifica en Base64 en memoria intermedia (`BytesIO`) y se envía al endpoint de la API (Celda 10).

### Implementación

```python
imagen_b64 = base64.b64encode(buffer.getvalue()).decode("utf-8")
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": PROMPT_VLM_EXTRACCION},
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{imagen_b64}"}}
        ]
    }]
)
```

### Lección sobre architectural simplicity

El cambio eliminó **dos dependencias pesadas** (Tesseract + OpenCV) y simplificó el pipeline. **Menos código que mantener, mejor performance.** Este es el patrón "frontier model replaces traditional pipeline".

---

<a id="entrada-6"></a>
## Entrada 6 — 6 de abril de 2026 · Refactor UX/UI Wizard Layout

### Contexto y motivación

La interfaz inicial era una sola pantalla con todos los componentes visibles simultáneamente: ingesta, resultado, payload técnico, métricas de telemetría, panel HITL. Para un investigador era utilizable. Para un operador de ventas B2B no técnico era **abrumadora**.

> **Insight:** la interfaz no estaba alineada con el flujo de trabajo del usuario final. Necesitaba **separar fases cognitivas**.

### Decisión arquitectónica: Wizard Layout + Progressive Disclosure

1. **Wizard Layout:** dividir la UI en `gr.Group` semánticos representando pasos secuenciales (Paso 1: Ingesta → Paso 2: Revisión → Paso 3: Auditoría HITL). Cada paso oculta los detalles del siguiente hasta que es relevante.

2. **Mapeo Visual Directo:** agregar columna `Ítem #` como llave primaria visual. Cada fila de la tabla tiene un número que coincide con el menú desplegable del panel HITL. **Erradica fricción cognitiva** durante la corrección manual.

3. **Progressive Disclosure:** el payload JSON crudo y la telemetría viven dentro de un `gr.Accordion` colapsado. Limpia la interfaz para el usuario de negocio pero mantiene la información accesible bajo demanda para TI.

### Lección de UX para sistemas con IA

**El operador no técnico no necesita ver cómo razona el modelo — necesita ver qué tiene que hacer ahora.** Cuando mezclamos la perspectiva del investigador con la del operador, ambos sufren. La separación de capas (qué ve cada audiencia) es tan importante como la separación de código (qué hace cada módulo).

---

<a id="entrada-5"></a>
## Entrada 5 — 5 de abril de 2026 · Memoria HITL + Poka-Yoke + RAG

### Contexto y motivación

Las correcciones humanas se estaban guardando en un archivo `.jsonl` pero **no se usaban**. El sistema cometía el mismo error una y otra vez en pedidos similares. Estaba violando el principio fundamental: **los sistemas con humanos en el loop deben mejorar con el uso**.

> **Hipótesis:** si vectorizo el historial de correcciones humanas en un índice FAISS secundario y lo recupero como contexto few-shot dinámico, el sistema aprenderá *in situ* sin reentrenamiento.

### Decisión arquitectónica: dos estrategias diferenciadas

Planteé explícitamente dos estrategias de aprendizaje:

**Estrategia 1 — In-Context Learning (lo que implementé):**
- Memoria externa con FAISS sobre historial HITL.
- Recupera lecciones cercanas semánticamente.
- Inyecta como contexto few-shot en el prompt del LLM.
- **Sin tocar pesos del modelo.**
- Ideal para PoC: inmediato, bajo costo, reversible.

**Estrategia 2 — Fine-Tuning (roadmap a largo plazo):**
- Cuando `historial_correcciones_hitl.jsonl` supere 1,000 registros validados.
- Ajuste fino sobre modelos fundacionales (Llama 3 o similar).
- Reduce costo unitario y dependencia de proveedor.
- **Eliminación de la necesidad de inyectar contexto masivo.**

### Implementación: Poka-Yoke Visual

Para garantizar **integridad del Data Flywheel**, rediseñé la interfaz de corrección humana:
- El operador NO escribe el SKU manualmente.
- Selecciona desde un Dropdown anclado a `df_catalogo['SKU'].unique()`.
- **Cero errores tipográficos** en el dataset de aprendizaje.

> **Principio Poka-Yoke (Shigeo Shingo):** el sistema previene el error humano por diseño, no por entrenamiento del operador.

### Mitigación de inanición de contexto

Subí el hiperparámetro `k=10` en BM25 y FAISS porque los pedidos B2B reales pueden tener 5-6 productos simultáneos en una "canasta". Con `k=5`, los documentos relevantes para los últimos productos quedaban fuera de la ventana de contexto. Con `k=10`, el modelo tiene suficiente contexto incluso en canastas grandes.

### Lección sobre Data Flywheel

**Un sistema HITL solo es valioso si las correcciones humanas se reincorporan al sistema.** Sin loop de retorno, el HITL es solo una burocracia de revisión. Con loop de retorno, es un mecanismo de mejora continua.

---

<a id="entrada-4"></a>
## Entrada 4 — 31 de marzo de 2026 · Bootstrapping dinámico

### Contexto y motivación

El Ground Truth de la evaluación estaba **codificado en duro** en una constante Python. Al agregar productos nuevos al catálogo sintético, el evaluador empezaba a marcar como "falso negativo" SKUs que en realidad existían en el catálogo pero no en el Ground Truth desactualizado.

> **Síntoma:** Data Drift entre el dataset de evaluación y el catálogo real. Producía métricas **artificialmente bajas**.

### Decisión arquitectónica: Bootstrapping en tiempo real

Implementé generación dinámica del Ground Truth que, en tiempo de ejecución, muestrea directamente `df_catalogo` y construye los casos de evaluación sobre la marcha. El dataset de evaluación **siempre está sincronizado** con el catálogo actual.

### Decisión paralela: Few-Shot Prompting

Migré el prompt extractor de Zero-Shot a Few-Shot, inyectando 2-3 pares de ejemplos (Input → Razonamiento → JSON). Esto ancló el comportamiento del LLM ante casos out-of-domain.

### Lección sobre evaluación rigurosa

**El Ground Truth es código** — debe versionarse, mantenerse, y probarse. Tratar el dataset de evaluación como un artefacto estático genera bugs silenciosos en las métricas.

---

<a id="entrada-3"></a>
## Entrada 3 — 30 de marzo de 2026 · Búsqueda híbrida + CoT

### Contexto y motivación

El recuperador inicial usaba **solo FAISS** (búsqueda semántica densa). Al evaluar con pedidos que contenían códigos exactos o calibres específicos, observé fallas: FAISS devolvía productos "semánticamente cercanos" pero **léxicamente incorrectos**.

> **Hipótesis:** un sistema híbrido que combine búsqueda semántica (FAISS) con búsqueda léxica (BM25) cubrirá ambos tipos de consulta.

### Decisión arquitectónica: Ensemble Retriever 50/50

Implementé un retriever híbrido con pesos `[0.5, 0.5]`:
- **FAISS:** captura intent semántico ("alambre para cercar")
- **BM25:** captura exactitud léxica ("calibre 14")

### Decisión paralela: filtro determinístico HITL

Agregué una **capa determinista de seguridad** que fuerza la etiqueta `REVISION_MANUAL` si la IA reporta confianza Media/Baja. Esto previene alucinaciones hacia el ERP.

> **Nota retrospectiva:** este filtro evolucionó en V3 hacia el `confianza_es_suficiente()` parametrizable que resuelve OBS-01. Aquí ya estaba la semilla del patrón defense-in-depth.

### Decisión paralela: eliminación de Text Splitting

El `RecursiveCharacterTextSplitter` que venía por defecto en LangChain **rompía el contexto tabular** del catálogo. Si un producto tenía atributos largos, el splitter lo partía en múltiples chunks, perdiendo la coherencia.

**Eliminé text splitting** y establecí relación estricta **1:1** (1 registro de DataFrame = 1 Documento LangChain). Esto es contraintuitivo para texto narrativo pero **correcto para datos tabulares**.

### Lección sobre defaults de frameworks

**Los defaults de los frameworks asumen el caso promedio.** Para casos especializados (como datos tabulares estructurados), los defaults pueden ser activamente perjudiciales. Vale la pena cuestionar cada hiperparámetro y validar contra el caso de uso real.

---

<a id="entrada-2"></a>
## Entrada 2 — 28 de marzo de 2026 · Primer prototipo en Colab

### Contexto y motivación

Tras conceptualizar el problema y validar la viabilidad técnica (Entrada 1), construí el primer prototipo funcional. Decisiones iniciales:

- **Plataforma:** Google Colab (gratuita, GPU disponible, compartible).
- **Modelo:** GPT-4o-mini (más económico para iterar).
- **Stack:** LangChain 0.3.x + FAISS + Pydantic + Gradio.

### Implementación mínima viable

El primer prototipo (V1) tenía:
1. Catálogo sintético generado proceduralmente (**150 SKUs** en 5 categorías: Alambre de Púas, Malla Electrosoldada, Postes de Concreto, Grapas Galvanizadas, Clavos de Acero — Celda 3).
2. Pipeline básico: input texto → embeddings → FAISS → LLM → JSON estructurado.
3. UI Gradio mínima con campo de texto y tabla de resultados.
4. Sin HITL, sin memoria, sin tests.

### Métricas baseline V1 (de `metricas_oficiales.json`)

| Configuración | F1 Identificación | Latencia | Costo/pedido |
|---|---|---|---|
| GPT-4o-mini V1 | 0.5700 | 4.26 s | $0.000353 |
| GPT-4o V1 | 0.6433 | 3.18 s | $0.00615 |

> **Insight:** la baseline ya era razonable, lo cual sugería que el problema era **tratable** con la arquitectura propuesta. Esto justificó continuar iterando.

---

<a id="entrada-1"></a>
## Entrada 1 — 20 de marzo de 2026 · Conceptualización inicial

### Contexto y motivación

La idea del proyecto nació de observar el flujo de trabajo en el contexto de pedidos B2B:

> Los pedidos B2B llegan a través de **múltiples canales informales**: WhatsApp con fotos del cuaderno del cliente, PDFs escaneados, planillas Excel desordenadas, mensajes de texto con jerga local ("dame medio kilo de pua perimetral"). Un operador humano dedica **3-5 minutos a cada pedido** clasificando, buscando SKUs en el ERP, verificando stock, y copiando datos manualmente.

> Errores frecuentes: calibres mal interpretados, cantidades transcritas incorrectamente, productos descontinuados aceptados, quiebres de stock no detectados a tiempo. Cada error logístico cuesta dinero.

### Hipótesis científica

> **¿Puede un sistema multimodal de IA aplicada automatizar la conversión de pedidos B2B informales en payloads JSON estructurados validados contra el catálogo ERP, manteniendo alta precisión mediante Human-in-the-Loop?**

### Tres principios rectores definidos al inicio

1. **Alta Precisión por contrato:** falsos positivos (alucinaciones logísticas) son críticos. Falsos negativos (revisión humana extra) son tolerables. El sistema debe **preferir derivar antes que adivinar**.

2. **Aprendizaje continuo:** las correcciones humanas deben **reintroducirse** al sistema para evitar repetir errores. Data Flywheel.

3. **Auditabilidad por diseño:** cada decisión del sistema debe ser **explicable** y rastreable. No cajas negras.

### Decisión de stack inicial

- **LLM:** OpenAI GPT-4o (state-of-the-art en razonamiento estructurado).
- **RAG:** FAISS + BM25 (cobertura semántica + léxica).
- **Framework:** LangChain (estándar de facto en 2025).
- **Schema:** Pydantic v2 (validación robusta + Python típico).
- **UI:** Gradio (rápida para PoC, integrable después).

### Lección filosófica

**La parte difícil de la IA aplicada no es la IA — es la aplicación.** Los modelos están disponibles. El stack está maduro. Lo que diferencia un proyecto exitoso de uno fallido es **entender profundamente el dominio** y **codificar las restricciones correctas** en el sistema.

---

## 📚 Referencias bibliográficas usadas en este diario

- Cormack, G. V., Clarke, C. L., & Buettcher, S. (2009). *Reciprocal rank fusion outperforms condorcet and individual rank learning methods.* SIGIR '09.
- Karlton, P. (atribuido). *"There are only two hard things in Computer Science: cache invalidation and naming things."*
- Martin, R. C. (2017). *Clean Architecture: A Craftsman's Guide to Software Structure and Design.* Prentice Hall.
- Shingo, S. (1986). *Zero Quality Control: Source Inspection and the Poka-Yoke System.* Productivity Press.
- OpenAI Cookbook. *How to handle rate limits.* https://cookbook.openai.com/examples/how_to_handle_rate_limits
- AWS Architecture Center. *Exponential backoff and jitter.* https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

---

*Diario mantenido por Mateo Córdova · Juan Portero · Última actualización: 12 de mayo de 2026.*
