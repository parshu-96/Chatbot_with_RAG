```markdown
# ShopSphere — RAG Customer Support Chatbot

A Retrieval-Augmented Generation (RAG) chatbot for e-commerce customer support,
built with Spring Boot and Spring AI. It answers customer questions grounded in
PDF policy documents stored in a Pinecone vector database, with OpenAI providing
both embeddings and the final answers. Ships with a built-in chat UI.

## Tech Stack

- **Java 21**, **Spring Boot 4.1.1**, **Spring AI 2.0.1**
- **OpenAI** — chat model `gpt-5-mini`, embeddings `text-embedding-3-small` (1024 dims)
- **Pinecone** — vector store (index `shop-support`, namespace `policies`)
- **Gradle** — build tool

## How It Works

1. On startup, `ChatbotService.loadKnowledgeBase()` reads all PDFs in
   `src/main/resources/knowledge/`, splits them into 300-token chunks with
   `TokenTextSplitter`, and stores them in Pinecone.
2. `GET /api/v1/query?question=...` triggers a similarity search (top 4 chunks),
   builds a grounded system prompt with the retrieved context, and calls OpenAI.
3. If the answer isn't in the documents, the bot replies:
   *"I don't have that information in the company documents."*
4. A static chat UI at `index.html` calls the endpoint and renders the answer.

## Prerequisites

- Java 21
- A Pinecone index named `shop-support` with **1024 dimensions**
- OpenAI API key

## Setup

```bash
export OPEN_AI_API_KEY=sk-...
export PINECONE_API_KEY=pcsk-...
```

Place your policy PDFs in `src/main/resources/knowledge/`.

## Run

```bash
./gradlew bootRun
```

Then open `http://localhost:8080/` for the chat UI.

## API

| Method | Endpoint | Params | Response |
|--------|----------|--------|----------|
| GET | `/api/v1/query` | `question` (String) | `{ "answer": "..." }` |

Example:

```bash
curl "http://localhost:8080/api/v1/query?question=What%20is%20the%20return%20policy"
```

## Project Structure

- `ChatbotController` — REST endpoint `/api/v1/query`
- `ChatbotService` — PDF ingestion (`@PostConstruct`) + RAG query logic
- `static/index.html` — chat UI calling the API
- `application.yaml` — OpenAI & Pinecone configuration
```

---

## Details

Versions and dependencies come from [1](#1-0) . The endpoint contract is defined in [2](#1-1) , and the ingestion (300-token chunks) plus retrieval (topK=4, grounded system prompt) logic lives in [3](#1-2) . Env vars and index settings come from [4](#1-3) , and the frontend fetch call is in [5](#1-4) .

## Notes

- This version adds the `index.html` chat UI, which wasn't in context last time — it calls `/api/v1/query` directly via `fetch`.
- The Pinecone index dimension must match `embedding.dimensions: 1024` in `application.yaml`.
- PDFs are loaded from `classpath*:knowledge/*.pdf` per [6](#1-5) .

Wiki pages you might want to explore:
- [Overview (parshu-96/Chatbot_with_RAG)](/wiki/parshu-96/Chatbot_with_RAG#1)
- [REST API (/api/v1/query) (parshu-96/Chatbot_with_RAG)](/wiki/parshu-96/Chatbot_with_RAG#2.3)

