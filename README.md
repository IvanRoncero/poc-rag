# PoC RAG con MuleSoft AI Chain + Ollama + Llama3

## Descripción

Este proyecto es una **prueba de concepto (PoC)** de un sistema **RAG (Retrieval-Augmented Generation)** construido con:

- **Anypoint Studio**
- **MuleSoft AI Chain Connector**
- **Ollama**
- **modelo Llama3**
- un **documento TXT** como base de conocimiento

La aplicación permite:

1. **ingestar un documento** en un vector store
2. **realizar preguntas** sobre ese documento
3. **obtener respuestas generadas por el modelo** usando el contenido del documento como contexto

---

## Arquitectura

La PoC se divide en dos fases:

### 1. Ingesta del documento

Endpoint:

```http
POST /rag/init
