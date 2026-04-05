# Control de Cambios (Changelog)

Todas las actualizaciones notables, decisiones arquitectónicas y mejoras en las métricas de este proyecto se documentarán en este archivo.

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
