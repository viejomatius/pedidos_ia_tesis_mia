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
