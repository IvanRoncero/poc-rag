# Guía paso a paso: Implementación de RAG en Azure

**Audiencia**: managers técnicos y stakeholders de decisión.
**Objetivo**: explicar qué se va a construir, qué recursos Azure se necesitan, qué requisitos previos hay que cumplir, y la secuencia de pasos para llegar de cero a producción.

---

## 1. Resumen ejecutivo

Vamos a construir un sistema **RAG (Retrieval-Augmented Generation)** sobre Azure que permite a la organización consultar documentos propios usando lenguaje natural. El sistema combina dos capacidades:

1. **Búsqueda inteligente** sobre nuestros documentos (Azure AI Search).
2. **Generación de respuestas en lenguaje natural** usando un LLM (Azure OpenAI).

**Beneficios para el negocio**:
- Consultas en lenguaje natural sobre documentación interna
- Respuestas trazables a documentos fuente (auditables)
- Datos siempre dentro del tenant de Azure (cumplimiento GDPR, datos en EU)
- Integración nativa con Entra ID para control de acceso

**Tiempos**: 7-10 semanas desde aprobación hasta producción.
**Coste año 1**: ~$15.000-25.000 USD según volumen (detallado en sección 8).

---

## 2. Arquitectura objetivo

```
┌─────────────┐
│  Usuario    │
└──────┬──────┘
       │ pregunta
       ▼
┌──────────────────────────────────┐
│  API (Azure Container Apps)      │
│  - FastAPI + LangChain           │
│  - Autenticación con Entra ID    │
└──────┬───────────────────────────┘
       │
       ├──► [1] Embeddings ──► Azure OpenAI (text-embedding-3-small)
       │
       ├──► [2] Retrieval ───► Azure AI Search (vector + semantic)
       │
       ├──► [3] Generación ──► Azure OpenAI (GPT-5 o equivalente)
       │
       ▼
┌──────────────────────────────────┐
│  Respuesta + citas a documentos  │
└──────────────────────────────────┘

Observabilidad: Application Insights + Log Analytics
Almacenamiento documentos fuente: Azure Blob Storage
Secretos y claves: Azure Key Vault
```

### Recursos Azure necesarios

| Servicio | Función | Obligatorio |
|---|---|---|
| Azure AI Search | Base vectorial + búsqueda híbrida | Sí |
| Azure OpenAI Service | Embeddings + LLM generación | Sí |
| Azure Blob Storage | Almacén de documentos fuente | Sí |
| Azure Container Apps (o App Service) | Hosting de la API | Sí |
| Azure Key Vault | Secretos y API keys | Sí |
| Application Insights | Observabilidad y logs | Sí |
| Azure Container Registry | Imágenes Docker de la API | Recomendado |
| Azure Front Door / API Management | Gateway, rate limiting, WAF | Recomendado en producción |
| Private Endpoints | Red privada entre servicios | Recomendado en producción |

---

## 3. Requisitos previos

### 3.1. Requisitos organizativos

| Requisito | Quién lo aprueba | Tiempo estimado |
|---|---|---|
| Suscripción Azure con presupuesto asignado | Finanzas / CTO | 1-2 semanas |
| Acceso a Azure OpenAI Service | Solicitud formal a Microsoft (formulario) | 1-3 días (suele ser inmediato) |
| Cuotas (TPM) suficientes para el modelo elegido | Microsoft (solicitud en portal) | 1-7 días |
| Aprobación de tratamiento de datos / DPIA si hay datos personales | Legal / DPO | 2-4 semanas |
| Política de retención de logs e historial de queries | Legal / Compliance | 1-2 semanas |

> **Nota crítica sobre Azure OpenAI**: Microsoft ya no exige aprobación previa extensa para acceder a Azure OpenAI (eso cambió a finales de 2024). Hoy es activación rápida, pero **las cuotas de TPM (tokens per minute) sí se piden por separado** y son el verdadero cuello de botella para producción.

### 3.2. Requisitos técnicos

| Requisito | Detalle |
|---|---|
| Roles Azure | Contributor sobre el Resource Group, User Access Administrator para asignar RBAC |
| Entra ID | Tenant configurado, capacidad de crear App Registrations |
| Red | Decisión: ¿servicios públicos con firewall o private endpoints en VNet propia? |
| Región | Recomendado **West Europe** o **Sweden Central** (modelos OpenAI disponibles + residencia EU) |
| IaC | Bicep o Terraform (recomendado, no provisioning manual) |
| CI/CD | GitHub Actions o Azure DevOps |
| Equipo | 1 ingeniero ML/backend + 0.5 DevOps + acceso puntual a arquitecto cloud |

### 3.3. Decisiones que el manager debe firmar antes de empezar

1. **¿Qué documentos van a entrar al sistema?** Define el corpus inicial (volumen, formatos, sensibilidad).
2. **¿Datos personales?** Si sí → DPIA obligatorio, anonimización a estudiar.
3. **¿Quién puede consultar el RAG?** Define el modelo de permisos (toda la empresa, departamentos, roles).
4. **¿Qué modelo LLM?** Trade-off coste/calidad (sección 8).
5. **¿Región de despliegue?** Implica disponibilidad de modelos y cumplimiento.
6. **¿Modelo de red?** Pública con autenticación vs Private Endpoints (impacta complejidad y coste).

---

## 4. Fase 1 — Setup y prototipo (Semana 1-2)

**Objetivo**: tener un prototipo funcionando con un corpus pequeño en Azure free tier.
**Coste**: ~$0 (free tier + créditos iniciales de Azure OpenAI).

### Paso 1.1 — Crear el Resource Group

```bash
az group create \
  --name rg-rag-dev \
  --location westeurope
```

### Paso 1.2 — Provisionar Azure AI Search (tier Free)

```bash
az search service create \
  --name srch-rag-dev-001 \
  --resource-group rg-rag-dev \
  --sku Free \
  --location westeurope
```

**Limitaciones del free tier que el equipo debe conocer**:
- 1 instancia free por suscripción
- 50 MB de almacenamiento total
- 3 índices máximo
- Vector search disponible, pero **sin semantic ranking** (eso requiere Basic+)
- Sin SLA, recursos compartidos

### Paso 1.3 — Provisionar Azure OpenAI

```bash
az cognitiveservices account create \
  --name oai-rag-dev-001 \
  --resource-group rg-rag-dev \
  --location westeurope \
  --kind OpenAI \
  --sku S0
```

Después, **desplegar los modelos** desde el portal o por CLI:
- `text-embedding-3-small` (embeddings, 1536 dims)
- `gpt-5` o `gpt-4o` (modelo de generación)

> **Importante**: cada despliegue de modelo consume cuota TPM. Para el prototipo basta con 30K TPM por modelo, pero para producción se necesitan 100K-500K TPM según volumen.

### Paso 1.4 — Provisionar Storage y Key Vault

```bash
# Blob Storage para documentos
az storage account create \
  --name stragdev001 \
  --resource-group rg-rag-dev \
  --location westeurope \
  --sku Standard_LRS

# Key Vault para secretos
az keyvault create \
  --name kv-rag-dev-001 \
  --resource-group rg-rag-dev \
  --location westeurope
```

### Paso 1.5 — Construir pipeline de ingesta

Componentes en Python:

```python
from langchain_community.vectorstores.azuresearch import AzureSearch
from langchain_openai import AzureOpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import AzureBlobStorageContainerLoader

# 1. Cargar documentos desde Blob
loader = AzureBlobStorageContainerLoader(
    conn_str=BLOB_CONN_STR,
    container="documentos",
)
docs = loader.load()

# 2. Chunking
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
chunks = splitter.split_documents(docs)

# 3. Embeddings
embeddings = AzureOpenAIEmbeddings(
    azure_deployment="text-embedding-3-small",
    openai_api_version="2024-02-01",
)

# 4. Indexar en Azure AI Search
vector_store = AzureSearch(
    azure_search_endpoint=SEARCH_ENDPOINT,
    azure_search_key=SEARCH_KEY,
    index_name="documentos-v1",
    embedding_function=embeddings.embed_query,
)
vector_store.add_documents(chunks)
```

### Paso 1.6 — Construir pipeline de consulta

```python
from langchain_openai import AzureChatOpenAI
from langchain.chains import RetrievalQA

llm = AzureChatOpenAI(
    azure_deployment="gpt-5",
    openai_api_version="2024-02-01",
    temperature=0.1,
)

retriever = vector_store.as_retriever(search_kwargs={"k": 5})

qa = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=retriever,
    return_source_documents=True,
)

resultado = qa.invoke({"query": "¿Cuál es la política de vacaciones?"})
```

### Entregables fase 1

- [ ] Recursos Azure provisionados (idealmente vía Bicep ya desde el inicio)
- [ ] 50-100 documentos representativos indexados
- [ ] Pipeline ingesta + consulta funcionando
- [ ] Set de 30-50 preguntas con respuestas esperadas (golden dataset)
- [ ] Métricas base: recall@5, precision@5, latencia, coste por query

### Hito de salida de fase 1

Demo funcional con corpus pequeño. Métricas iniciales para decidir si merece la pena escalar.

---

## 5. Fase 2 — POC en Basic tier (Semana 3-4)

**Objetivo**: validar el sistema con corpus realista, calidad de producción y stakeholders probándolo.
**Coste**: ~$200-400/mes durante esta fase.

### Por qué subir a Basic tier

- El free tier (50MB) se llena rápido con cualquier corpus serio
- **Semantic ranking** solo está disponible desde Basic en adelante, y es donde está gran parte de la calidad
- Se necesita estabilidad para que stakeholders prueben sin que el servicio caiga

### Paso 2.1 — Upgrade a Basic tier

Crear un nuevo recurso Azure AI Search en tier Basic (no se puede hacer upgrade del Free directamente; hay que crear nuevo y reindexar):

```bash
az search service create \
  --name srch-rag-stg-001 \
  --resource-group rg-rag-dev \
  --sku Basic \
  --location westeurope \
  --replica-count 1 \
  --partition-count 1
```

**Capacidad Basic**:
- 2 GB de storage por Search Unit
- Hasta 3 réplicas y 3 particiones
- Vector search + semantic ranking habilitados
- Coste: ~$73/mes con 1 Search Unit

### Paso 2.2 — Activar semantic ranking

Semantic ranking es un reranker neural que mejora notablemente la precisión del retrieval. Hay que habilitarlo en el recurso y configurarlo en el índice:

```python
from azure.search.documents.indexes.models import (
    SemanticConfiguration, SemanticPrioritizedFields, SemanticField,
)

semantic_config = SemanticConfiguration(
    name="semantic-config",
    prioritized_fields=SemanticPrioritizedFields(
        title_field=SemanticField(field_name="title"),
        content_fields=[SemanticField(field_name="content")],
    ),
)
```

En la query:

```python
results = vector_store.semantic_hybrid_search(
    query="política de teletrabajo",
    k=5,
)
```

### Paso 2.3 — Activar hybrid search

Combina vector search con keyword (BM25). Mejora bastante cuando hay términos técnicos, nombres propios o jerga específica de la empresa.

### Paso 2.4 — Activar compresión vectorial

Reduce el espacio en disco entre 2-4x con pérdida de calidad mínima:

```python
from azure.search.documents.indexes.models import (
    ScalarQuantizationCompression,
)

# En la definición del índice
compressions=[
    ScalarQuantizationCompression(
        compression_name="scalar-compression",
        rescoring_options={"enable_rescoring": True},
    )
],
```

### Paso 2.5 — Desplegar API en Container Apps

```bash
# Crear entorno de Container Apps
az containerapp env create \
  --name cae-rag-stg \
  --resource-group rg-rag-dev \
  --location westeurope

# Desplegar la API
az containerapp create \
  --name ca-rag-api-stg \
  --resource-group rg-rag-dev \
  --environment cae-rag-stg \
  --image <ACR>.azurecr.io/rag-api:v1 \
  --target-port 8000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3
```

### Paso 2.6 — Autenticación con Entra ID

1. Crear App Registration en Entra ID.
2. Configurar Container Apps con autenticación integrada Entra ID.
3. Validar tokens JWT en la API.

### Paso 2.7 — Observabilidad básica

```bash
# Application Insights
az monitor app-insights component create \
  --app appi-rag-stg \
  --resource-group rg-rag-dev \
  --location westeurope
```

Integrar OpenTelemetry en la API para enviar trazas, métricas y logs.

### Entregables fase 2

- [ ] Azure AI Search en Basic con semantic ranking activado
- [ ] Hybrid search funcionando
- [ ] API desplegada en Container Apps con autenticación Entra ID
- [ ] Compresión vectorial activada
- [ ] Application Insights integrado con dashboards básicos
- [ ] Documentación de uso para stakeholders
- [ ] **Evaluación A/B**: vector puro vs hybrid vs hybrid+semantic ranking
- [ ] Métricas de calidad mejoradas: target recall@5 >85%, precision@5 >70%

### Hito de salida de fase 2

POC validado con métricas. Decisión go/no-go para producción basada en datos reales.

---

## 6. Fase 3 — Producción (Semana 5-10)

**Objetivo**: sistema productivo con SLA, alta disponibilidad y operabilidad.
**Coste**: ~$1.000-3.000/mes según escala.

### Paso 3.1 — Provisionar tier Standard (S1)

```bash
az search service create \
  --name srch-rag-prd-001 \
  --resource-group rg-rag-prd \
  --sku Standard \
  --location westeurope \
  --replica-count 2 \
  --partition-count 1
```

**Por qué 2 réplicas mínimo**: con 1 réplica no hay SLA. Con 2 réplicas tienes 99.9% SLA de lectura. Con 3, 99.9% de lectura y escritura.

**Capacidades S1**:
- 25 GB de storage por partición
- Hasta 12 particiones y 12 réplicas (36 SUs max)
- Coste base: ~$245/SU/mes → 2 SUs ≈ $490/mes

### Paso 3.2 — Red privada con Private Endpoints

Para producción, exponer servicios solo dentro de una VNet:

```bash
# Crear VNet
az network vnet create \
  --name vnet-rag-prd \
  --resource-group rg-rag-prd \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-services \
  --subnet-prefix 10.0.1.0/24

# Private Endpoint para Azure AI Search
az network private-endpoint create \
  --name pe-srch-rag-prd \
  --resource-group rg-rag-prd \
  --vnet-name vnet-rag-prd \
  --subnet snet-services \
  --private-connection-resource-id <SEARCH_RESOURCE_ID> \
  --group-id searchService \
  --connection-name pe-srch-connection
```

Repetir para Azure OpenAI, Blob Storage y Key Vault.

### Paso 3.3 — Migración de datos a producción

**Estrategia recomendada**: aliases de índices.

```python
# 1. Crear nuevo índice productivo con la versión nueva
create_index("documentos-prd-v1")

# 2. Reindexar todo el corpus
ingest_all_documents(target_index="documentos-prd-v1")

# 3. Swap atómico del alias
update_alias(alias="documentos-prd", target="documentos-prd-v1")

# 4. (Opcional) Mantener v0 como rollback durante 1 semana
```

Esto permite reindexar masivamente **sin downtime** y con rollback inmediato.

### Paso 3.4 — RBAC con Entra ID en lugar de API keys

Eliminar uso de API keys en código. Usar Managed Identity de Container Apps:

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# El SDK obtiene tokens automáticamente
search_client = SearchClient(
    endpoint=SEARCH_ENDPOINT,
    index_name="documentos-prd",
    credential=credential,
)
```

Roles RBAC necesarios sobre Azure AI Search:
- `Search Index Data Reader` para la API en runtime
- `Search Index Data Contributor` para el job de ingesta
- `Search Service Contributor` solo para administradores

### Paso 3.5 — Backups y disaster recovery

Azure AI Search **no tiene backups nativos automatizados**. Estrategias:

1. **Backup del índice**: script periódico que exporta el contenido del índice a Blob Storage.
2. **Reindexación desde fuente**: mantener los documentos originales en Blob Storage versionados, lo que permite reconstruir el índice en cualquier momento (recomendado).
3. **Multi-región** (si SLA crítico): replicar índice en una segunda región.

### Paso 3.6 — Alertas y SLOs

Configurar alertas en Application Insights:

| Métrica | Umbral | Severidad |
|---|---|---|
| Latencia p95 > 3s | Sostenido 5 min | Warning |
| Error rate > 2% | Sostenido 5 min | Critical |
| Azure OpenAI 429 (throttling) | Cualquier ocurrencia | Warning |
| Search query latency > 1s p95 | Sostenido 10 min | Warning |
| Coste diario > $50 | Diario | Warning |

### Paso 3.7 — Tests de carga

Antes de go-live, validar la capacidad con k6 o Locust:

```python
# Locustfile ejemplo
from locust import HttpUser, task

class RAGUser(HttpUser):
    @task
    def query(self):
        self.client.post("/query", json={
            "question": "¿Cuál es la política de vacaciones?"
        })
```

Targets típicos:
- 100 queries/min sostenidas sin degradación
- p95 latency < 4s end-to-end (incluye LLM)
- 0% errors bajo carga objetivo

### Entregables fase 3

- [ ] Azure AI Search S1 con 2+ réplicas
- [ ] Private Endpoints configurados
- [ ] Managed Identity en uso (no API keys en código)
- [ ] Backups vía reindexación desde Blob versionado
- [ ] Application Insights con alertas
- [ ] Tests de carga pasados
- [ ] Runbook operativo documentado
- [ ] Formación al equipo de operaciones

---

## 7. Cronograma y plan de proyecto

| Semana | Hito principal | Responsable |
|---|---|---|
| 0 | Aprobaciones, presupuesto, DPIA | Manager + Legal |
| 1 | Resource Group, IaC base, free tier provisioning | DevOps + ML Eng |
| 2 | Pipeline ingesta + consulta + golden dataset | ML Eng |
| 3 | Upgrade a Basic + semantic ranking + hybrid | ML Eng |
| 4 | API en Container Apps + Entra ID + observabilidad | ML Eng + DevOps |
| 5 | Provisionar S1 + Private Endpoints + RBAC | DevOps |
| 6 | Migración índices producción + Managed Identity | ML Eng + DevOps |
| 7 | Alertas, backups, runbook | DevOps + ML Eng |
| 8 | Tests de carga + ajustes | ML Eng |
| 9 | UAT con usuarios piloto | Producto + ML Eng |
| 10 | Go-live | Todos |

---

## 8. Estimaciones de coste

### 8.1. Escenario base (5.000 queries/día, 500k chunks)

| Concepto | Coste mensual | Coste anual |
|---|---|---|
| Azure AI Search S1, 2 SUs | $490 | $5.880 |
| Azure OpenAI embeddings (text-embedding-3-small) | $30 | $360 |
| Azure OpenAI generación (GPT-5, sin optimizar) | $2.500 | $30.000 |
| Azure OpenAI generación (con caching + routing inteligente) | $700-1.000 | $8.400-12.000 |
| Blob Storage + Container Apps + observabilidad | $150 | $1.800 |
| **Total optimizado año 1** | | **~$16.000-20.000** |
| **Total sin optimizar año 1** | | **~$38.000** |

### 8.2. Palancas de optimización clave

1. **Prompt caching**: -90% en tokens de input cacheados (system prompt + few-shot fijos). Ahorro típico 40-60% en LLM.
2. **Batch API**: -50% en input/output para tareas asíncronas (re-indexación nocturna, evaluaciones).
3. **Routing multi-modelo**: usar Haiku 4.5 o GPT-mini para clasificación/routing, modelo flagship solo para generación final. Ahorro típico 30-50%.
4. **Reducir top-K**: pasar de top-10 a top-5 chunks reduce tokens de input casi a la mitad.
5. **Compresión vectorial**: -50-75% storage en Search.

### 8.3. Reservas y descuentos

- **Azure Reserved Capacity** para AI Search: hasta -33% comprometiendo 1 o 3 años.
- **Azure OpenAI Provisioned Throughput Units (PTUs)**: para volúmenes muy altos puede ser más barato que pago por uso. Estudiar a partir de ~$5.000/mes en OpenAI.

---

## 9. Riesgos principales y mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Cuotas TPM insuficientes en producción | Alta | Alto | Pedir aumento de cuota en semana 1, no esperar a estar bloqueados |
| Calidad de retrieval baja en dominio específico | Media | Alto | Golden dataset desde fase 1, iterar chunking y embeddings |
| Coste LLM se dispara | Media | Alto | Caching + routing multi-modelo desde fase 2, alertas de coste |
| Datos sensibles filtrados en respuestas | Baja | Crítico | Filtrado por permisos en retrieval (security trimming), no solo en respuesta |
| Vendor lock-in Azure | Media | Medio | Abstracción con LangChain, pipeline de ingesta agnóstico |
| Reindexación masiva bloquea producción | Media | Alto | Aliases de índices con swap atómico |
| Throttling de Azure OpenAI en picos | Alta | Medio | Retry con backoff exponencial, circuit breaker, fallback a modelo secundario |

---

## 10. Decisiones que se piden al management

Para arrancar el proyecto, se necesita decisión sobre:

1. **Presupuesto aprobado**: $20.000-25.000 año 1 (escenario base optimizado).
2. **Región de despliegue**: West Europe (recomendado por residencia EU y disponibilidad de modelos).
3. **Modelo LLM principal**: GPT-5 estándar (equilibrio coste/calidad) o GPT-5 mini (más barato, calidad algo menor).
4. **Modelo de red**: pública con Entra ID + WAF (más simple) vs Private Endpoints (más seguro, más complejo).
5. **Corpus inicial**: definir qué documentos entran en fase 1 (recomendado: 1.000-5.000 documentos representativos).
6. **Política de retención**: cuánto tiempo se guardan queries de usuarios y respuestas (impacta GDPR).
7. **Equipo asignado**: 1 ML Eng dedicado, 0.5 DevOps, acceso a arquitecto cloud.

---

## 11. Próximos pasos inmediatos

1. **Semana 0** — Aprobaciones internas (presupuesto, legal, seguridad).
2. **Semana 0** — Crear suscripción Azure dedicada o Resource Group con tags de coste.
3. **Semana 1** — Solicitar acceso Azure OpenAI + cuotas TPM iniciales.
4. **Semana 1** — Kick-off técnico con el equipo, repos creados, IaC inicial.
5. **Semana 2** — Primer prototipo funcional para demo interna.

---

## Anexo A — Glosario para no técnicos

- **RAG (Retrieval-Augmented Generation)**: técnica que combina búsqueda de información en documentos propios con un LLM para generar respuestas basadas en esa información.
- **Embeddings**: representación numérica del significado de un texto, que permite buscar por similitud semántica en lugar de palabras exactas.
- **Vector database**: base de datos especializada en buscar por similitud entre embeddings.
- **LLM (Large Language Model)**: modelo de lenguaje grande, como GPT-5 o Claude, capaz de generar texto en lenguaje natural.
- **Semantic ranking**: re-ordenado de resultados de búsqueda usando comprensión semántica avanzada.
- **Hybrid search**: combinación de búsqueda por palabras clave (BM25) y búsqueda vectorial.
- **TPM (Tokens per minute)**: límite de capacidad de Azure OpenAI, medida en tokens procesables por minuto.
- **SLA (Service Level Agreement)**: compromiso contractual de disponibilidad del servicio.
- **Private Endpoint**: conexión privada dentro de Azure que evita exponer servicios a internet público.
- **Managed Identity**: identidad de Azure que permite a servicios autenticarse sin guardar contraseñas o claves.
