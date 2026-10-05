# ShopSphere — RAG Customer Support Chatbot

A Retrieval-Augmented Generation (RAG) chatbot for e-commerce customer support, built with **Spring Boot** and **Spring AI**.

It answers customer questions using information retrieved from PDF policy documents stored in a **Pinecone vector database**, with **OpenAI** providing both embeddings and final answers.

The project also includes a built-in web-based chat UI.

## Tech Stack

- **Java 21**
- **Spring Boot 4.1.1**
- **Spring AI 2.0.1**
- **OpenAI**
  - Chat model: `gpt-5-mini`
  - Embedding model: `text-embedding-3-small`
  - Embedding dimensions: `1024`
- **Pinecone** — vector database
  - Index: `shop-support`
  - Namespace: `policies`
- **Gradle** — build tool
- **HTML/CSS/JavaScript** — chat UI

## How It Works

The application follows a typical RAG pipeline:

1. On application startup, `ChatbotService.loadKnowledgeBase()` loads PDF policy documents from:

   ```text
   src/main/resources/knowledge/
   ```

2. The documents are split into **300-token chunks** using `TokenTextSplitter`.

3. Each chunk is converted into an embedding using OpenAI's `text-embedding-3-small` model.

4. The embeddings and document content are stored in the Pinecone `shop-support` index under the `policies` namespace.

5. When a user sends a question through:

   ```text
   GET /api/v1/query?question=...
   ```

   the application performs a similarity search and retrieves the **top 4 relevant chunks**.

6. The retrieved content is added to a grounded system prompt and sent to OpenAI to generate the final answer.

7. If the required information cannot be found in the company documents, the chatbot responds:

   > "I don't have that information in the company documents."

8. The built-in chat UI sends questions to the API and displays the generated response.

## RAG Architecture

```text
                    ┌─────────────────────┐
                    │    Policy PDFs      │
                    │ knowledge/*.pdf     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ TokenTextSplitter   │
                    │   300-token chunks  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       OpenAI        │
                    │ text-embedding-3-   │
                    │       small         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      Pinecone       │
                    │  shop-support       │
                    │  namespace: policies│
                    └──────────┬──────────┘
                               │
                User Question │
                               ▼
                    ┌─────────────────────┐
                    │ ChatbotController   │
                    │   /api/v1/query     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Similarity Search   │
                    │      Top K = 4      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Grounded Prompt   │
                    │ + Retrieved Context │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       OpenAI        │
                    │     gpt-5-mini      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Chat UI / User   │
                    └─────────────────────┘
```

## Prerequisites

Before running the application, make sure you have:

- Java 21
- An OpenAI API key
- A Pinecone API key
- A Pinecone index named `shop-support`
- Pinecone index dimension set to **1024**

The Pinecone dimension must match the embedding dimension configured in the application.

## Configuration

Set the required environment variables:

### Linux / macOS

```bash
export OPEN_AI_API_KEY=sk-...
export PINECONE_API_KEY=pcsk-...
```

### Windows PowerShell

```powershell
$env:OPEN_AI_API_KEY="sk-..."
$env:PINECONE_API_KEY="pcsk-..."
```

Place your policy PDF files in:

```text
src/main/resources/knowledge/
```

## Pinecone Configuration

The application expects the following Pinecone configuration:

```text
Index:     shop-support
Namespace: policies
Dimension: 1024
```

The index dimension must match:

```yaml
spring:
  ai:
    openai:
      embedding:
        model: text-embedding-3-small
        dimensions: 1024
```

## Run the Application

Run the application using Gradle:

```bash
./gradlew bootRun
```

On Windows:

```powershell
.\gradlew.bat bootRun
```

Once the application starts, open:

```text
http://localhost:8080/
```

to access the chat UI.

## API

### Query the chatbot

**Endpoint:**

```text
GET /api/v1/query
```

**Parameter:**

| Parameter | Type | Description |
|---|---|---|
| `question` | String | Customer question |

**Example:**

```bash
curl "http://localhost:8080/api/v1/query?question=What%20is%20the%20return%20policy"
```

**Response:**

```json
{
  "answer": "..."
}
```

## Project Structure

```text
src/
├── main/
│   ├── java/
│   │   └── com/parshu/rag_demo/
│   │       ├── controller/
│   │       │   └── ChatbotController.java
│   │       ├── service/
│   │       │   └── ChatbotService.java
│   │       └── RagDemoApplication.java
│   │
│   └── resources/
│       ├── knowledge/
│       │   └── *.pdf
│       ├── static/
│       │   └── index.html
│       └── application.yaml
│
└── build.gradle
```

### Main Components

- **`ChatbotController`** — exposes the `/api/v1/query` REST endpoint.
- **`ChatbotService`** — handles PDF ingestion, embedding generation, Pinecone storage, similarity search, and RAG response generation.
- **`static/index.html`** — web-based chat interface that calls the REST API.
- **`application.yaml`** — contains OpenAI and Pinecone configuration.
- **`knowledge/`** — contains the PDF policy documents used as the chatbot's knowledge base.

## Important Notes

- PDF documents are loaded from:

  ```text
  classpath*:knowledge/*.pdf
  ```

- Documents are split into **300-token chunks**.
- Retrieval uses **top K = 4** relevant chunks.
- The chatbot is designed to answer questions based on the retrieved company documents.
- The Pinecone index must use **1024 dimensions** because the application generates 1024-dimensional embeddings.
- API keys should be provided through environment variables and should **not be committed to Git**.

## Example Questions

You can ask questions such as:

```text
What is the return policy?
```

```text
How long do I have to return an item?
```

```text
What is the refund process?
```

```text
Do I need the original invoice to return a product?
```

The chatbot will retrieve the most relevant policy information from Pinecone before generating the answer.
