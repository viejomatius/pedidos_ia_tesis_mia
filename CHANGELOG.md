# Control de Cambios (Changelog)

Todas las actualizaciones notables, decisiones arquitectónicas y mejoras en las métricas de este proyecto se documentarán en este archivo.

## [V13.0] - 2026-04-15 (Benchmarking Cognitivo y Escalabilidad de Modelos Fronterizos)
### Añadido
- **Benchmarking de Modelos de Lenguaje (LLM vs SLM):** Se ejecutó un intercambio en caliente (*Hot-Swap*) del motor de inferencia en el Módulo 3, migrando de un Small Language Model (`gpt-4o-mini`) a un Frontier Model (`gpt-4o`) para evaluar el impacto en la resolución de ambigüedades.

### Cambiado
- **Superación del Techo Cognitivo:** Las métricas de la prueba de estrés (Módulo 4 con *Chaos Monkey*) experimentaron un salto cuantitativo. El F1-Score pasó del ~53% (sesgo de disponibilidad de contexto del modelo pequeño) al **70.83%**.
- **Comportamiento Asintótico:** El sistema demostró que la arquitectura RAG y el *Prompting* de Cadena de Pensamiento (CoT) eran correctos, pero requerían la profundidad de razonamiento paramétrico de un modelo mayor para acatar estrictamente las restricciones negativas (no adivinar frente a la ambigüedad).
- **Validación de Arquitectura:** Se comprobó el principio de "Separación de Responsabilidades" (Separation of Concerns). El código en Python se encarga exclusivamente del enrutamiento transaccional (Validación de Stock), confiando la decisión de confianza semántica íntegramente al LLM.

## [V12.0] - 2026-04-08 (Alineamiento de Modelo, Chain of Thought y Estabilización de Métricas)
### Añadido
- **Zero-Shot Chain of Thought (CoT):** Se reestructuró la arquitectura cognitiva del Extractor (Módulo 3). Se introdujo la llave `analisis_previo` como primer elemento de la salida JSON, forzando al LLM a instanciar su ruta de razonamiento lógico antes de emitir un SKU. Esto mitigó el sesgo de adivinación ("Síndrome del Buen Vendedor").
- **Prompts Equilibrados (Model Alignment):** Se calibró la función de recompensa semántica del prompt. Se reemplazó la penalización estricta (que causaba "Over-anchoring" y un colapso de F1 al 28%) por un mandato dual: "Automatizar la certeza y enrutar la ambigüedad", estabilizando el comportamiento del modelo.

### Cambiado
- **Refactorización Matemática del Stress Test (Chaos Monkey):** Se corrigió la lógica de evaluación del Módulo 4. La calificación mutli-etiqueta migró de Teoría de Conjuntos (`set()`) a Frecuencia Exacta (`collections.Counter()`) para soportar conteo de Verdaderos Positivos duplicados.
- **Inyección de Ruido B2B Realista:** El algoritmo de estrés ahora elimina sistemáticamente variables numéricas (calibres, pulgadas, metros) para forzar una ambigüedad insalvable (Irreducible Noise), evitando penalizar a la IA por aciertos legítimos.
- **Estabilización de Línea Base:** Con las correcciones matemáticas y cognitivas, el F1-Score se consolidó en la "Zona Ricitos de Oro" (~83.87%), con una latencia de inferencia de ~7.8 segundos por transacción (Reducción del Lead Time operativo superior al 95%).

## [V11.0] - 2026-04-07 (Migración de OCR Tradicional a Modelos Multimodales VLM)
### Cambiado
- **Extracción de Datos de Órdenes de Compra:** Se deprecó el uso de Tesseract OCR y OpenCV en el Módulo 5. Debido a la pérdida de estructura espacial (Information Loss) inherente al OCR en documentos tabulares, se implementó inferencia multimodal directa utilizando la API de `gpt-4o`.
- **Pre-procesamiento de Imágenes:** Se añadió la función de conversión de tensores NumPy a formato Base64 nativo en memoria intermedia (`BytesIO`), agilizando el envío de payloads visuales al endpoint del LLM sin requerir almacenamiento en disco.

## [V10.0] - 2026-04-06 (Refactorización de Interfaz UX/UI y Progressive Disclosure)
### Añadido
- **Mapeo Visual Directo:** Se añadió la columna `Ítem #` a la tabla renderizada en HTML. Esta columna actúa como una llave primaria visual que vincula biunívocamente cada fila de la tabla con el menú desplegable del panel de auditoría (HITL), erradicando la fricción cognitiva durante la corrección manual.
- **Flujo Guiado (Wizard Layout):** Reestructuración de la interfaz en `gr.Group` semánticos (Paso 1: Ingesta, Paso 2: Revisión, Paso 3: Auditoría) para alinear la herramienta con los flujos de trabajo tradicionales de un operador de ventas (Non-technical user).

### Cambiado
- **Progressive Disclosure:** El payload crudo (JSON) y los metadatos de telemetría (Costo/Tokens) se encapsularon dentro de un componente `gr.Accordion` (colapsado por defecto). Esto limpia la interfaz visual para el usuario de negocio, manteniendo la información accesible bajo demanda para el equipo de TI.
- **Estilización de Estados:** Refinamiento del CSS inyectado en la tabla de resultados para mejorar la legibilidad del semáforo de colores (Verde/Amarillo/Rojo) asociado a las reglas de negocio del ERP.

## [V9.0] - 2026-04-05 (Jerarquía Cognitiva, Preservación de Atributos y Tolerancia a Fallos)
### Añadido
- **Jerarquía de Prompting (Regla Suprema 0):** Se implementó una jerarquía absoluta en las instrucciones del LLM (Módulo 3). Las "Lecciones Aprendidas" inyectadas por la memoria dinámica (HITL) ahora anulan explícitamente cualquier restricción del catálogo RAG recuperado, garantizando que el modelo acate el reentrenamiento *In-Context* sin conflictos de lógica.
- **Búsqueda Segura en Memoria:** Se implementó `globals().get()` en el Módulo 3 para la instanciación de la memoria HITL. Esto evita dependencias cíclicas (`NameError`) permitiendo que el Módulo de Evaluación (Módulo 4) pueda ejecutarse "en frío" si el motor vectorial de memoria aún no ha sido inicializado, garantizando tolerancia a fallos.

### Cambiado
- **Mitigación de Inanición de Contexto (Context Starvation):** Se amplió la ventana de recuperación del motor híbrido de `k=10` a `k=15`, permitiendo procesar canastas B2B de hasta 5-6 productos simultáneos sin desplazar documentos relevantes del contexto del LLM.
- **Prevención de Truncamiento de Atributos (Information Loss):** Se reescribió el prompt del *Query Rewriter*. El agente pre-procesador ahora tiene la orden estricta de preservar atributos técnicos (calibre, tipo, grosor, material), evitando que la pérdida de variables categóricas genere falsos negativos en el buscador FAISS/BM25.
- **Calibración del Trade-Off (Precision-Recall):** Las nuevas restricciones cognitivas estabilizaron el F1-Score en el cuartil del 80%. Esta estabilización es producto de priorizar la "Alta Precisión" (postura conservadora del LLM que prefiere derivar a HITL antes que alucinar un SKU), alineando el modelo con la política de aversión al riesgo de la logística B2B.

## [V8.0] - 2026-04-05 (Orquestación DAG y Prevención de Errores de Estado)
### Añadido
- **Orquestador de Dependencias (State Manager):** Se implementó la clase estática `OrquestadorPipeline` en el Módulo 1. Este componente actúa como un validador de Grafos Acíclicos Dirigidos (DAGs) en tiempo de ejecución. Verifica la existencia de variables globales (modelos, dataframes, motores FAISS) antes de permitir la ejecución de módulos dependientes. 

### Cambiado
- **Refactorización de Módulos Periféricos:** Fusión arquitectónica de los Módulos 5 (Computer Vision) y 6 (Base de Datos HITL) en un solo bloque de ejecución para garantizar que el inicializador de memoria dinámica en caliente (`cargar_memoria_hitl`) disponga de las dependencias de pre-procesamiento visual antes de instanciar el frontend.
- **Defensa contra Capa 8:** Se mitigaron los riesgos inherentes a los entornos Jupyter/Colab (ejecución no secuencial de celdas) mediante excepciones tempranas (`RuntimeError` semánticos), asegurando tolerancia a fallos en la demostración de la PoC.

## [V7.0] - 2026-04-05 (UX/UI Poka-Yoke & Estrategia de Aprendizaje RAG)
### Añadido
- **Memoria Dinámica HITL (Estrategia 1 - In-Context Learning):** Implementación de un motor vectorial paralelo (FAISS) que indexa el archivo `dataset_finetuning_hitl.jsonl`. El sistema ahora recupera correcciones históricas hechas por humanos y las inyecta como ejemplos *Few-Shot* dinámicos en el prompt principal, permitiendo que el modelo "aprenda" en tiempo real sin necesidad de reentrenar pesos.
- **Poka-Yoke Visual (Combo Boxes):** Interfaz de auditoría humana rediseñada. El usuario ahora corrige errores seleccionando SKUs directamente desde un Dropdown enlazado a la base de datos oficial, garantizando 100% de integridad de datos y evitando errores tipográficos en el dataset de reentrenamiento.
- **Recálculo Transaccional en Vivo:** La UI ahora recalcula el stock y el razonamiento del sistema de forma automática al momento de aplicar una corrección humana.
- **Orquestador de Dependencias (State Manager):** Se implementó la clase `OrquestadorPipeline` en el Módulo 1 para garantizar la ejecución secuencial estricta del notebook. Utiliza el principio de Grafos Acíclicos Dirigidos (DAGs) para validar la existencia de variables globales en memoria (`globals()`), previniendo errores de capa 8 (ejecución desordenada por parte del usuario) y garantizando la integridad de los datos antes de la inferencia IA o el despliegue de la interfaz web.

### Cambiado
- **Mitigación de Inanición de Contexto (Context Starvation):** Se incrementó el hiperparámetro de recuperación `k=10` en los motores BM25 y FAISS para garantizar que los pedidos con múltiples ítems (canastas B2B) no excluyan documentos relevantes de la ventana de contexto del LLM.

### Nota de Arquitectura (Roadmap a Largo Plazo)
- **Aviso:** La actual Estrategia 1 (Memoria In-Context) es ideal para la fase de Prueba de Concepto (PoC) por su inmediatez y bajo costo. Sin embargo, a largo plazo, la arquitectura debe migrar a la **Estrategia 2 (Fine-Tuning / Ajuste Fino)**. Cuando el archivo `.jsonl` supere los 1,000 registros validados, se utilizará para modificar los pesos neuronales subyacentes de un modelo Open Source, eliminando la necesidad de inyectar contexto masivo en el prompt, reduciendo latencia y garantizando soberanía tecnológica.

## [V6.0] - 2026-04-05 (Optimización de Precisión)
### Añadido
- **Agente de Query Rewriting (Pre-Retrieval):** Implementación de una etapa de pre-procesamiento mediante un LLM especializado que purifica, corrige y normaliza el input del usuario (OCR/Texto crudo) antes de la búsqueda.
- **Patrón de Doble Inferencia:** Reestructuración del pipeline para separar la normalización de la consulta de la extracción final de entidades.

### Cambiado
- **Arquitectura del Pipeline de Inferencia:** El flujo ahora sigue la secuencia: *Input Crudo -> Query Rewriter -> Motor Híbrido -> Extractor Final*.

### Resultados
- **Ruptura de Umbral de Desempeño:** Superación del estancamiento del 80% en el F1-Score, mitigando el ruido léxico y los errores de transcripción del OCR.
- **Optimización de Recall:** Incremento significativo en la recuperación de documentos relevantes a cambio de un aumento marginal en la latencia.


## [V5.0] - 2026-04-05 (Producción PoC)
### Añadido
- **BuscadorHibridoPersonalizado:** Clase nativa que desacopla la orquestación de FAISS y BM25 (50/50), eliminando la dependencia de `langchain.retrievers` y optimizando la unificación de resultados sin duplicados.
- **Bootstrapping en Tiempo Real:** Función `generar_dataset_dinamico()` para alineación automática del Ground Truth con el catálogo en memoria.
- **Evaluación Basada en Teoría de Conjuntos:** Migración de métricas por ítem a métricas por "canasta de pedido" para reflejar la precisión transaccional real.

### Cambiado
- **Refactorización de Datos:** Depuración del generador sintético para restringir la 'Descripción de Jerga' solo a entidades (sustantivos/atributos), eliminando ruido conversacional.
- **Blindaje Cognitivo:** Evolución del prompt de Zero-Shot a Few-Shot con patrones de Chain-of-Thought (CoT) para mejorar la detección de productos fuera de dominio (Out-of-Domain).

### Eliminado
- **Text Chunking:** Se eliminó definitivamente la partición de documentos para garantizar la cohesión de los datos tabulares del ERP.

### Resultados
- **Optimización de Métricas:** Incremento del F1-Score ponderado global de 33.3% a +85%.

## [V4.0] - 2026-03-31
### Añadido
- **Bootstrapping Dinámico:** Implementación de generación dinámica del *Ground Truth* (Módulo 4) realizando un muestreo directo de la tabla de dimensiones en memoria para evitar falsos negativos (*Data Drift*).

### Cambiado
- **Few-Shot Prompting:** Transición de *Zero-Shot* a *Few-Shot Prompting* en el LLM (Módulo 3), inyectando pares de ejemplos (Input -> Razonamiento -> JSON) para mejorar el enrutamiento de entidades fuera de dominio.
- **Métricas de Evaluación:** Refinamiento de la evaluación multi-etiqueta ajustando la Teoría de Conjuntos. Ahora, la deducción correcta de incertidumbre (`REVISION_MANUAL`) computa positivamente como un "Verdadero Positivo" en el Recall Global.

---

## [V3.0] - 2026-03-30
### Añadido
- **Filtro de Seguridad HITL:** Nueva capa determinista que fuerza la etiqueta `REVISION_MANUAL` si la IA reporta confianza Media/Baja, previniendo alucinaciones hacia el ERP.

### Cambiado
- **Búsqueda Híbrida (Ensemble Retriever):** Migración a un motor híbrido que combina FAISS (búsqueda semántica, 50%) y BM25 (búsqueda léxica, 50%) para solucionar caídas en el F1-Score al buscar SKUs exactos.
- **Chain-of-Thought (CoT):** Reestructuración del prompt base exigiendo al modelo generar un razonamiento explícito *antes* de asignar la confianza, mejorando los Verdaderos Negativos.

### Eliminado
- **Text Splitting:** Se eliminó el `RecursiveCharacterTextSplitter` para evitar romper el contexto tabular. Se estableció una relación estricta 1:1 (1 registro de DataFrame = 1 Documento LangChain).
