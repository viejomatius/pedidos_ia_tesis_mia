# TesisIAAplicada
Repositorio donde se van a guardar los codigos que se realizan referentes al sistema de pedidos de IA que se utilizara como tesis de MIA

# 🏭 Copiloto IA B2B - Ideal Alambrec (Intelligent Order Processing)

Este repositorio contiene la Prueba de Concepto (PoC) para el procesamiento inteligente de órdenes de compra informales B2B. Desarrollado como proyecto de Tesis para la Maestría en Inteligencia Artificial Aplicada.

## 🎯 Arquitectura de la Solución
El sistema aborda la "brecha semántica" entre la jerga del cliente y el catálogo ERP, utilizando una arquitectura MLOps multimodal que combina:

1. **Computer Vision (OpenCV + Tesseract):** Preprocesamiento morfológico (binarización adaptativa) y OCR para documentos manuscritos o fotografiados.
2. **Retrieval-Augmented Generation (RAG) Híbrido:** Combinación de BM25 (búsqueda léxica/Sparse) y FAISS (búsqueda semántica/Dense) para recuperar el contexto exacto del catálogo.
3. **Generación Estructurada (LLMs):** Uso de GPT-4o-mini con técnicas de *Few-Shot* y *Chain-of-Thought prompting* para extraer entidades en un payload JSON estricto.
4. **Lógica Transaccional y HITL:** Validación determinista de inventario (quiebres de stock) y enrutamiento automatizado hacia validadores humanos (*Human-in-the-Loop*) en escenarios de ambigüedad.
5. **Data Flywheel & Aprendizaje Dinámico:** El sistema captura el *feedback* humano en un formato estructurado (JSONL). Para esta PoC, se utiliza la **Estrategia 1 (In-Context Learning)**, donde una base de datos vectorial secundaria inyecta correcciones históricas en el prompt para evitar que la IA cometa el mismo error dos veces. 
   > ⚠️ **Roadmap a futuro:** A largo plazo y como evolución natural de este proyecto, se implementará la **Estrategia 2 (Fine-Tuning)**, utilizando este repositorio JSONL para realizar un ajuste fino de pesos neuronales sobre modelos fundacionales eficientes (Ej. Llama 3).

## ⚙️ Estructura del Pipeline
El flujo de trabajo se divide en los siguientes módulos dentro del notebook principal, protegidos por un orquestador de estado estricto (DAG):
* **Módulo 1:** Configuración, observabilidad (Logs), gestión de secretos y Orquestador de Dependencias.
* **Módulo 2:** Ingesta del ERP simulado y construcción del Espacio Latente Híbrido.
* **Módulo 3:** Motor de Inferencia Core (Query Rewriting + Few-Shot Extractor).
* **Módulo 4:** Bootstrapping dinámico y Evaluación Multi-Etiqueta.
* **Módulo 5 y 6:** Visión Artificial (OpenCV) y Data Flywheel con recarga de memoria en caliente (In-Context Learning).
* **Módulo 7:** Frontend interactivo multimodal desplegado con Gradio y validación Poka-Yoke.
* **Módulo 8:** Generación automática de reportes visuales y gráficos (Alta resolución para métricas académicas).

## 🚀 Requisitos de Ejecución
Para reproducir este entorno de pruebas:
* **Entorno:** Google Colab (Se recomienda hardware T4 GPU para el motor de visión).
* **Dependencias de Sistema:** Debian/Ubuntu requiere instalar los binarios de Tesseract (`sudo apt-get install tesseract-ocr tesseract-ocr-spa`).
* **Credenciales:** Se requiere configurar `OPENAI_API_KEY` en el gestor de secretos (Vault) del entorno.

## 📊 Métricas Objetivo (KPIs)
* **Métricas Técnicas (Evaluación Multi-Etiqueta):** * Optimización del F1-Score ponderado mediante Búsqueda Híbrida y Expansión de Consultas.
* **Precision-Recall Trade-Off:** El sistema está calibrado intencionalmente hacia la *Alta Precisión* (High Precision). Se prioriza el enrutamiento seguro hacia revisión humana (*Falsos Negativos* tolerables) por encima del riesgo de alucinación logística (*Falsos Positivos* críticos).
* **Métricas de Negocio:** Reducción drástica del *Lead Time Operativo* (medido en segundos por transacción) y monitorización del Costo de Inferencia API (USD/req).
* **Integridad de Datos:** Garantía de *Zero-Typo* en el Data Flywheel mediante interfaces de corrección Poka-Yoke ancladas a la tabla de dimensiones del ERP.
---
*Desarrollado bajo los estándares de Calidad de Software ISO/IEC 25010 (Adecuación Funcional y Fiabilidad).*
