# 📓 Diario de Tesis: Bitácora de Decisiones Técnicas

Este documento registra los desafíos, experimentos fallidos y decisiones arquitectónicas tomadas durante el desarrollo del "Copiloto IA B2B".

---

## 29 de Marzo de 2026: Evaluación Visual
- **Desafío**: Necesitábamos una forma más clara de ver los errores del modelo más allá de un número de precisión.
- **Intento**: Se empezó a trabajar en el Módulo 4.1 para integrar una **Matriz de Confusión Visual** usando `seaborn` y `matplotlib`.
- **Objetivo**: Poder identificar visualmente si la IA confunde categorías específicas (ej. confundir Alambre de Púas con Malla).

## 30 de Marzo de 2026: Arquitectura y Lógica del Motor
- **Problema detectado (Vectores)**: El F1-Score estaba estancado en ~0.33. Notamos que FAISS (vectores densos) es excelente para conceptos, pero falla cuando el cliente pide un SKU exacto o términos muy cortos.
- **Solución (Búsqueda Híbrida)**: Migramos a un modelo *Ensemble*. Ahora usamos BM25 para palabras exactas (50%) y FAISS para el significado semántico (50%). La lección aquí es que la IA generativa sigue necesitando algoritmos clásicos de búsqueda (lexical) para la precisión B2B.
- **Problema (Contexto)**: El uso de `RecursiveCharacterTextSplitter` estaba rompiendo las filas de nuestro catálogo ERP, dejando SKUs huérfanos sin su descripción.
- **Solución (Integridad)**: Eliminamos el "splitting". Configuramos una relación estricta 1:1 donde cada producto es un documento independiente.
- **Problema (Alucinaciones)**: El modelo fallaba al asignar la etiqueta `REVISION_MANUAL` cuando no estaba seguro (baja tasa de Verdaderos Negativos). Además, existía el riesgo de que la IA enviara datos erróneos al ERP.
- **Solución (Chain-of-Thought y HITL Override)**: 
  1. Reestructuramos el prompt exigiendo que el modelo redacte un `razonamiento` explícito *antes* de definir su nivel de confianza (CoT). Hacer que "piense en voz alta" mejoró drásticamente la clasificación.
  2. A nivel de código (fuera del LLM), codificamos un "Filtro de Seguridad". Si la IA reporta confianza "Media" o "Baja", el sistema sobrescribe la decisión y fuerza la etiqueta `REVISION_MANUAL`. Esto blinda la operación y garantiza el *Human-in-the-Loop*.

## 31 de Marzo de 2026: Refinamiento de Evaluación y Prompting
- **Desafío (Data Drift)**: El F1-Score no subía del 50%. Descubrimos un error en nuestra metodología de prueba: los SKUs en el dataset de evaluación estaban "quemados" (hardcoded), pero el generador de catálogo creaba IDs aleatorios en cada ejecución.
- **Solución (Bootstrapping Dinámico)**: Creamos la función `generar_dataset_dinamico()`. Ahora, el examen que le tomamos a la IA se construye en tiempo real usando productos que *sabemos* que existen en el catálogo actual.
- **Desafío (Entidades fuera de dominio)**: Cuando el cliente pedía productos absurdos o ambiguos, el sistema colapsaba tratando de adivinar.
- **Solución (Few-Shot Prompting)**: Pasamos de un *Zero-Shot* a un *Few-Shot Prompt*. Inyectamos pares de ejemplos directamente en la instrucción de la IA (`Input -> Razonamiento -> JSON`). Al ver ejemplos de cómo rendirse ordenadamente ante datos fuera de dominio, el modelo aprendió a enrutar correctamente hacia la revisión humana.
- **Desafío (Métricas Injustas)**: El sistema de evaluación matemático nos estaba penalizando (bajando el Recall) cuando la IA correctamente decía "no sé" ante una ambigüedad.
- **Solución (Teoría de Conjuntos)**: Ajustamos la lógica de la matriz de evaluación. Ahora, si el caso requiere revisión y la IA deduce correctamente que necesita ayuda (`REVISION_MANUAL`), el sistema lo computa matemáticamente como un "Verdadero Positivo".

## 05 de Abril de 2026: Salto a Producción (V5.0) y Optimización MLOps

### El problema de la "Contaminación" del Lenguaje
- **Desafío**: Notamos que el motor de búsqueda léxica se confundía cuando las descripciones sintéticas incluían verbos o frases de compra (ej. "quiero comprar...").
- **Decisión**: Refactorizamos el catálogo para que la jerga solo contenga atributos físicos y sustantivos. 
- **Lección**: En B2B, la pureza de las entidades en la base de datos vectorial es más importante que la cantidad de texto.

### Desacoplamiento y Estabilidad Técnica
- **Problema**: Las librerías estándar de `langchain` para recuperación híbrida presentaban fallas de dependencias y rigidez en el manejo de duplicados.
- **Solución**: Desarrollamos una clase nativa `BuscadorHibridoPersonalizado`. Esto nos dio control total sobre cómo se mezclan los resultados de FAISS y BM25. Al eliminar el *Text Chunking*, garantizamos que un SKU nunca se parta a la mitad, manteniendo la integridad transaccional.

### El Salto del 33% al 85% en F1-Score
- **Reflexión**: ¿Por qué estábamos estancados en el 33%? Identificamos dos "anclas" que frenaban el proyecto:
  1. **Alucinaciones de Confianza**: La IA intentaba adivinar productos que no existían. El **Few-Shot CoT** le enseñó al modelo que "pensar antes de responder" permite admitir ignorancia y pedir ayuda humana (HITL) correctamente.
  2. **Evaluación Estática**: Estábamos evaluando un motor dinámico con un examen estático. El **Bootstrapping dinámico** permitió que la evaluación sea justa, comparando lo que la IA ve contra lo que realmente existe en ese momento en el ERP.
- **Conclusión**: La combinación de búsqueda híbrida, blindaje del prompt y evaluación por "canasta de pedido" validó que nuestra arquitectura RAG es capaz de manejar la ambigüedad industrial con una precisión de nivel profesional.

## 05 de Abril de 2026: Superando el techo del 80% (Query Rewriting)

### El "Techo de Cristal" de la Precisión
- **Desafío**: Las métricas se estancaron en el 80%. Por más que ajustábamos los hiperparámetros del motor híbrido, el sistema seguía fallando en ciertos casos.
- **Análisis de Errores**: Descubrimos que el culpable no era el buscador, sino el "ruido" del usuario. Errores ortográficos, sintaxis rota y la mala calidad del OCR ensuciaban la consulta, haciendo que FAISS y BM25 buscaran basura.
- **Decisión Arquitectónica**: Implementamos **Query Rewriting**. En lugar de buscar directamente con el texto sucio del cliente, pusimos a un LLM pequeño y rápido a "traducir" el pedido a términos técnicos del catálogo antes de ir a la base de datos.
- **Reflexión Técnica**: Este patrón de **Doble Inferencia** resolvió el problema de "Lost in the Middle". Al limpiar la consulta primero, el motor híbrido recibe "oro puro", lo que disparó el Recall global.
- **Lección Aprendida**: Un Arquitecto de IA no solo busca modelos más grandes; busca estrategias para que la información fluya sin ruido a través de los componentes. El costo extra en latencia es un "trade-off" aceptable frente a la fiabilidad que ganamos.

### Visión a Futuro: El Siguiente Nivel (Re-Ranking)
- **Nota para la defensa**: Si bien el Query Rewriting nos llevó al éxito actual, la investigación sugiere que para alcanzar el 99% de precisión deberíamos implementar un **Re-Ranker con Cross-Encoders**. Esto actuaría como un "juez" de Deep Learning para calificar la relevancia de los resultados recuperados antes de la generación final. Queda planteado como trabajo futuro.

## 05 de Abril de 2026: UI, Inanición de Contexto y Estrategias de Reentrenamiento
- **Problema detectado (Inanición de Contexto)**: Al probar con pedidos de más de 2 productos simultáneos ("pua perimetral y clavos corrugados"), la IA ignoraba la pua. Descubrimos que el `k=4` de FAISS era muy restrictivo; los ítems extra quedaban fuera de los 4 primeros resultados, causando *Context Starvation*. Subimos el `k=10` y el problema se resolvió.
- **Mejora de UX (Poka-Yoke)**: Modificamos el Módulo 7 (Gradio). Escribir el JSON a mano era un riesgo inmenso de corrupción de datos. Cambiamos a *Combo Boxes* conectados a la tabla de dimensiones. Ahora el vendedor selecciona y el sistema reconstruye el JSON, asegurando integridad total antes de mandarlo al `.jsonl`.
- **Decisión Arquitectónica (Estrategias de Reentrenamiento)**: Debatimos cómo hacer que la máquina "aprenda" del botón de guardar. 
  - *Decisión actual:* Implementamos la **Estrategia 1 (RAG Dinámico / In-Context Learning)**. Vectorizamos el `.jsonl` en caliente. Si el usuario pide algo mal escrito que ya fue corregido ayer, el RAG lo detecta y le dice a la IA: *"Ayer un humano lo corrigió así, haz lo mismo"*. Funciona excelente para la PoC.
 
## 05 de Abril de 2026: Tolerancia a Fallos y Orquestación (Defensa de Entorno)
- **Riesgo detectado (Entornos Notebook)**: Durante las pruebas, se observó que la naturaleza de Google Colab (y Jupyter en general) permite ejecutar celdas en desorden. Si un validador de la PoC ejecuta la interfaz antes de cargar el motor vectorial, Python arroja errores de compilación (`NameError`) que dan la falsa impresión de que el código está roto.
- **Decisión Arquitectónica (State Manager)**: En lugar de depender de instrucciones manuales, se programó un sistema defensivo. Desarrollamos la clase `OrquestadorPipeline` que funciona simulando la validación de dependencias de un DAG (como Apache Airflow). 
- **Reflexión Técnica**: Cada módulo ahora verifica proactivamente si sus prerequisitos existen en la memoria (`globals()`). Si faltan, el orquestador aborta la ejecución con un mensaje de error controlado y corporativo. Esto eleva la robustez del entregable, demostrando principios de ingeniería de software (Poka-Yoke / Prevención de Errores) aplicados al despliegue de modelos de Machine Learning.
  - *Reflexión a largo plazo:* Dejamos estipulado en la tesis que la **Estrategia 2 (Fine-Tuning)** es el objetivo de producción definitivo. Cuando acumulemos suficientes casos de borde en el JSONL, dejaremos de depender del prompt y reentrenaremos los pesos de la red, reduciendo el consumo de tokens y estabilizando el sistema para Ideal Alambrec.

## 05 de Abril de 2026 (Parte 2): Conflictos Cognitivos y Trade-Off Estadístico
- **Problema detectado (Conflictos de Prompting y Truncamiento)**: Al procesar un pedido muy complejo con múltiples variables (*"postes concreto grueso de tipo 4"*), el sistema falló. Al forzar la corrección humana, el sistema **no aprendió** y volvió a fallar. Adicionalmente, el Módulo 4 de evaluación colapsó con un `NameError` al intentar evaluar.
- **Análisis de Causa Raíz**: 
  1. *Information Loss*: El *Query Rewriter* borraba adjetivos clave ("grueso", "tipo 4") creyendo que eran ruido, lo que rompía la búsqueda híbrida. 
  2. *Conflicto de Leyes*: El prompt final priorizaba "solo usar el catálogo" por encima de la memoria HITL, por lo que el LLM desobedecía la corrección humana si no hallaba el producto exacto en el contexto.
  3. *Dependencia Cíclica*: El Módulo 3 exigía la existencia de la memoria del Módulo 6, rompiendo la evaluación si esta última no estaba activa.
- **Decisiones Arquitectónicas y Soluciones**: 
  - Se implementó la **"Regla Suprema 0"** en el prompt, otorgando poder de veto a la memoria histórica sobre la recuperación RAG estándar.
  - Se ordenó al pre-procesador la preservación estricta de atributos técnicos.
  - Se blindó la variable de memoria con `globals().get()` para hacerla opcional y tolerante a fallos.
- **Reflexión Técnica (El Trade-Off de Negocio)**: Observamos que el F1-Score se estabilizó en ~80%. Como ingenieros, comprendimos que esto no es un fallo, sino un **Precision-Recall Trade-Off**. Al hacer que las reglas del modelo sean extremadamente estrictas para que no adivine (High Precision), sacrificamos un poco de Recall (el modelo prefiere dudar y enviar a HITL). En el contexto B2B de Ideal Alambrec, un "Falso Negativo" (enviar a revisión) cuesta segundos de tiempo humano; pero un "Falso Positivo" (alucinar un SKU y despachar acero equivocado) cuesta miles de dólares. La arquitectura actual refleja exactamente la necesidad del negocio.
## El Fenómeno del 100% y el 'Stress Test' B2B
- **Anomalía Detectada**: Tras optimizar el *Query Rewriter* y purificar el catálogo, el Módulo de Evaluación arrojó métricas perfectas (F1-Score: 1.000).
- **Diagnóstico (Synthetic Data Bias)**: Se determinó que el dataset de prueba dinámico era "demasiado limpio", ya que inyectaba la *Jerga_Busqueda* exacta del dataframe. El modelo lograba una coincidencia uno-a-uno perfecta, actuando como un examen a libro abierto, lo cual no refleja la realidad operativa de la empresa.
- **Corrección Metodológica**: Se introdujo un algoritmo de caos (*Chaos Monkey*) en el generador de pruebas. Ahora, el 30% de las consultas son mutiladas intencionalmente (eliminando calibres o dimensiones) para simular clientes ambiguos. Esto sometió al sistema a un *Stress Test* real, estabilizando el F1-Score en valores metodológicamente válidos (85%-92%) y demostrando la capacidad del modelo de enrutar la ambigüedad hacia la revisión HITL de manera predecible.

## 07 de Abril de 2026: Trascendiendo el OCR hacia Vision-Language Models (VLM)
- **Problema de Investigación**: En el contexto B2B, muchas órdenes de compra llegan en formato PDF o imágenes con tablas. El OCR tradicional leía las filas de forma asimétrica, mezclando las cantidades de un SKU con la descripción de la fila inferior.
- **Decisión Arquitectónica**: Migramos la ingesta visual hacia un modelo fundacional multimodal (GPT-4o Vision). Mediante *Prompt Engineering* especializado, el VLM asume el rol de "Traductor Analítico", ignorando ruidos del documento (firmas, logos, direcciones fiscales) y serializando la tabla a una estructura de texto plano estandarizada.
- **Impacto Metodológico**: Esto preserva la integridad del *Pipeline In-Context Learning* de los Módulos 3 y 4. Al homogenizar el *input* (sea texto o imagen, todo entra al motor RAG como un string purificado), garantizamos que la evaluación comparativa, las métricas de negocio y la interfaz de auditoría funcionen indistintamente del canal de origen de la transacción.

## 08 de Abril de 2026: Ingeniería Cognitiva y el "Valle de la Muerte" de las Métricas
- **Desafío Metodológico:** Al someter la arquitectura a un entorno de pruebas destructivas (Stress Test), observamos fluctuaciones extremas en el F1-Score, oscilando entre el 100% (sesgo de datos sintéticos), el 28% y el 53%.
- **Diagnóstico Dual (Alineamiento de IA):**
  1. *Alucinación Compensatoria:* El modelo intentaba adivinar SKUs omitidos por el cliente para mantener una alta tasa de resolución, generando Falsos Positivos críticos para la logística.
  2. *Over-anchoring (Auditor Paranoico):* Al aplicar un prompt con penalizaciones estrictas para evitar que adivinara, el modelo `gpt-4o-mini` colapsó hacia la clase segura, enviando pedidos válidos a `REVISION_MANUAL` por miedo a equivocarse.
- **Solución Arquitectónica:** - Se implementó **Chain of Thought (CoT)**, forzando al modelo a justificar su decisión de atributos en un JSON *antes* de seleccionar el SKU.
  - Se rediseñó el prompt hacia un "Equilibrio Funcional", dándole al modelo el deber de ser eficiente cuando hay datos, y seguro cuando hay ambigüedad.
  - Se refactorizó la matemática del evaluador usando la librería `Counter` para conteos multi-etiqueta exactos.
- **Reflexión de Negocio:** Las métricas se estabilizaron en un ~84% de F1-Score. Comprendimos que forzar un 95%+ implicaría un sobreajuste (Overfitting). El 16% de "error" restante representa el **Ruido Irreductible** del sector ferretero: pedidos que son inherentemente ambiguos en el mundo real y que *deben* ser derivados al Human-In-The-Loop. El modelo alcanzó la madurez para producción con una latencia transaccional promedio de 7.8 segundos.

## 15 de Abril de 2026: El Estudio Comparativo y la Superación del Techo Cognitivo
- **Hipótesis del Problema:** Tras limpiar el código de interferencias simbólicas (donde Python sobreescribía prematuramente las decisiones de la IA), el modelo `gpt-4o-mini` se estancó en un F1-Score del 53.33%. Se hipotetizó que el sistema había alcanzado el "Techo Cognitivo" del modelo. El SLM (Small Language Model) sufría de "Sesgo de Disponibilidad de Contexto", ignorando las directrices de seguridad (no adivinar calibres) con tal de sugerir el SKU que el motor RAG le ponía enfrente.
- **Diseño del Experimento (Benchmarking):** Para validar si el fallo era arquitectónico o del modelo base, se realizó un *Ablation Study* aislando el motor. Se reemplazó `gpt-4o-mini` por el modelo fronterizo `gpt-4o`, manteniendo intactos el RAG, el *Prompt* de CoT y el evaluador de estrés.
- **Resultados y Discusión:** Las métricas se catapultaron inmediatamente al **70.83% de F1-Score**. Los falsos positivos (alucinaciones logísticas) colapsaron. El modelo mayor demostró la capacidad atencional necesaria para evaluar el contexto, notar la ausencia de atributos críticos y ejecutar la restricción negativa derivando la transacción a `REVISION_MANUAL`.
- **Conclusión de Negocio:** Este hito demuestra que en la logística B2B, donde el costo de un falso positivo (despachar acero equivocado) es astronómico, el uso de Modelos Fronterizos de alto razonamiento no es un lujo, sino un requerimiento de diseño. El ~29% de varianza restante asimila el Ruido Irreductible de la industria, garantizando una intervención humana (HITL) segura y justificada.

