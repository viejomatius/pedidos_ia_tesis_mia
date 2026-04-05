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

## ⚙️ Estructura del Pipeline
El flujo de trabajo se divide en los siguientes módulos dentro del notebook principal (`procesamiento_pedidos_core.ipynb`):
* **Módulo 1:** Configuración, observabilidad (Logs) y gestión de secretos.
* **Módulo 2:** Ingesta del ERP simulado y construcción del Espacio Latente Híbrido.
* **Módulo 3:** Motor de Inferencia Core y reglas de negocio.
* **Módulo 4:** Bootstrapping dinámico y Evaluación Multi-Etiqueta (Precision, Recall, F1-Score).
* **Módulos 5-7:** Visión Artificial, Data Flywheel (JSONL) y Frontend interactivo desplegado con Gradio.

## 🚀 Requisitos de Ejecución
Para reproducir este entorno de pruebas:
* **Entorno:** Google Colab (Se recomienda hardware T4 GPU para el motor de visión).
* **Dependencias de Sistema:** Debian/Ubuntu requiere instalar los binarios de Tesseract (`sudo apt-get install tesseract-ocr tesseract-ocr-spa`).
* **Credenciales:** Se requiere configurar `OPENAI_API_KEY` en el gestor de secretos (Vault) del entorno.

## 📊 Métricas Objetivo (KPIs)
* **Métricas Técnicas:** Optimización del F1-Score ponderado para clasificación Multi-Etiqueta, mitigando el *Data Drift*.
* **Métricas de Negocio:** Reducción drástica del *Lead Time Operativo* (medido en segundos por transacción) y monitorización del Costo de Inferencia API (USD/req).

---
*Desarrollado bajo los estándares de Calidad de Software ISO/IEC 25010 (Adecuación Funcional y Fiabilidad).*
