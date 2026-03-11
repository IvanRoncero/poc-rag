# Guía de configuración local — PoC RAG con MuleSoft AI Chain + Ollama + Llama3

Este documento explica, paso a paso, cómo preparar el entorno local, instalar dependencias y ejecutar este proyecto.

## 1) ¿Qué es este proyecto?

Es una prueba de concepto de **RAG (Retrieval-Augmented Generation)** en Mule 4 que:

1. Crea un vector store local.
2. Ingesta un documento de texto (`txtpruebas.txt`).
3. Responde preguntas usando **Ollama** + **Llama3** y el contexto del documento.

Flujos HTTP disponibles:

- `POST /rag/init` → inicializa e ingesta el documento.
- `GET /rag/ask?question=...` → realiza una consulta RAG.
- `POST /rag/ask` con body JSON `{"question": "..."}` → realiza una consulta RAG.

---

## 2) Requisitos previos

Antes de ejecutar, instala lo siguiente:

- **Java 17** (requisito de la app Mule).
- **Maven 3.9+**.
- **Anypoint Studio** (recomendado para ejecutar el proyecto Mule localmente).
- **Ollama** (motor LLM local).
- Modelo **Llama3** descargado en Ollama.
- Conectividad a repositorios de MuleSoft/Anypoint Exchange para resolver dependencias Maven.

> Nota: el proyecto está configurado para Mule Runtime **4.11.0**.

---

## 3) Verificar versiones instaladas

Ejecuta:

```bash
java -version
mvn -version
ollama --version
```

Debes validar especialmente:

- Java 17 activo.
- Maven funcional.
- Ollama instalado y ejecutable.

---

## 4) Instalación de dependencias

### 4.1 Dependencias de sistema

Instala Java 17, Maven y Ollama en tu sistema operativo.

### 4.2 Dependencias de Maven (Mule + conectores)

El proyecto usa estas dependencias principales:

- `mule-http-connector`
- `mule-sockets-connector`
- `mule4-aichain-connector`

Se resuelven desde:

- `https://maven.anypoint.mulesoft.com/api/v3/maven`
- `https://repository.mulesoft.org/nexus/repository/releases/`

Si tu entorno exige autenticación para Anypoint Exchange, configura tu `~/.m2/settings.xml` con credenciales del servidor `anypoint-exchange-v3`.

Ejemplo mínimo (ajústalo con tus credenciales):

```xml
<settings>
  <servers>
    <server>
      <id>anypoint-exchange-v3</id>
      <username>TU_USUARIO_ANYPOINT</username>
      <password>TU_PASSWORD_O_TOKEN</password>
    </server>
  </servers>
</settings>
```

---

## 5) Preparar Ollama y el modelo

1. Inicia Ollama (si no está corriendo).
2. Descarga el modelo:

```bash
ollama pull llama3
```

3. Verifica que el modelo esté disponible:

```bash
ollama list
```

El proyecto apunta por defecto a:

- `OLLAMA_BASE_URL = http://localhost:11434`

---

## 6) Clonar/abrir el proyecto

Este repositorio tiene esta estructura relevante:

- `poc-rag/pom.xml`
- `poc-rag/mule-artifact.json`
- `poc-rag/src/main/mule/poc-rag.xml`
- `poc-rag/src/main/resources/application.yaml`
- `poc-rag/src/main/resources/llm-config.json`
- `poc-rag/src/main/resources/docs/txtpruebas.txt`

Puedes trabajar de dos formas:

### Opción A (recomendada): Anypoint Studio

1. Abrir Anypoint Studio.
2. `File > Import > Anypoint Studio Project from File System`.
3. Seleccionar la carpeta `poc-rag/`.
4. Esperar resolución de dependencias Maven.
5. Ejecutar como **Mule Application**.

### Opción B: Maven CLI (empaquetado)

Desde la raíz del repo:

```bash
mvn -f poc-rag/pom.xml clean package
```

Esto valida compilación/empaquetado; para ejecución local de flujos Mule, Studio suele ser la vía más directa.

---

## 7) Configuración de la aplicación

### 7.1 HTTP listener

Archivo: `poc-rag/src/main/resources/application.yaml`

Valores por defecto:

- `host: 0.0.0.0`
- `port: 8081`

Si el puerto está ocupado, cámbialo en ese archivo.

### 7.2 Configuración de Ollama

Archivo: `poc-rag/src/main/resources/llm-config.json`

Valor por defecto:

- `OLLAMA_BASE_URL: http://localhost:11434`

Si Ollama corre en otro host/puerto, actualiza este valor.

---

## 8) Ejecución y prueba funcional

Una vez la app esté arriba:

### 8.1 Inicializar el store RAG

```bash
curl -X POST http://localhost:8081/rag/init
```

Respuesta esperada: JSON con `ok: true`.

### 8.2 Consultar por GET

```bash
curl "http://localhost:8081/rag/ask?question=Hazme%20un%20resumen"
```

### 8.3 Consultar por POST

```bash
curl -X POST http://localhost:8081/rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"¿Qué dice el documento?"}'
```

---

## 9) Solución de problemas frecuentes

### Error al resolver dependencias Maven

- Verifica acceso a internet/corporativo.
- Revisa credenciales en `~/.m2/settings.xml` para `anypoint-exchange-v3`.
- Reintenta con `mvn -U -f poc-rag/pom.xml clean package`.

### Error de conexión con Ollama

- Verifica que Ollama esté levantado en `http://localhost:11434`.
- Ejecuta `ollama list` para confirmar disponibilidad.
- Revisa `llm-config.json`.

### Error al consultar `/rag/ask`

- Asegúrate de ejecutar primero `POST /rag/init` para crear/ingestar el store.

### Puerto ocupado

- Cambia `http.port` en `application.yaml`.

---

5. Probar `POST /rag/init`.
6. Probar `GET/POST /rag/ask`.

¡Listo! Con eso deberías poder levantar y usar la PoC RAG en local.
