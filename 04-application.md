# 04 — Application

This document covers the project structure, the two-service architecture, how to set up the application, and a code tour of the key modules.

---

## Two-Service Architecture

The RAG system runs as two independent services communicating over HTTP:

```
┌─────────────────────────────────────────────────┐
│              Gateway (.NET) :8080                 │
│                                                   │
│  Responsibilities:                                │
│  • Chat API (REST + SSE streaming)                │
│  • Query routing (LLM-based intent detection)     │
│  • Grounded answer generation (LLM)               │
│  • Citation validation & repair loop              │
│  • Document management (CRUD via gRPC to Qdrant)  │
│  • Rate limiting, retry, timeout (Polly)          │
└──────────────────┬──────────────────────────────┘
                   │ HTTP (:8000)
┌──────────────────▼──────────────────────────────┐
│              Worker (Python) :8000                │
│                                                   │
│  Responsibilities:                                │
│  • PDF parsing (Docling)                          │
│  • Schema v2 chunking                             │
│  • Vision AI enrichment                           │
│  • BGE-M3 embedding (Dense + Sparse)              │
│  • Hybrid search + RRF fusion                     │
│  • Cross-encoder reranking                        │
│  • Vector upsert/delete (REST to Qdrant)          │
└──────────────────┬──────────────────────────────┘
                   │ REST (:6333) + gRPC (:6334)
┌──────────────────▼──────────────────────────────┐
│              Qdrant (Vector DB)                   │
│              Docker container                     │
└─────────────────────────────────────────────────┘
```

### Why Two Services?

| Decision | Rationale |
|----------|-----------|
| .NET for Gateway | ASP.NET Core excels at HTTP handling, streaming, middleware (Polly), Gemini SDK |
| Python for Worker | ML ecosystem (PyTorch, FlagEmbedding, Docling) only available in Python |
| HTTP between them | Simple, debuggable, replaceable — no tight coupling |

---

## Project Structure

```
rag-system/
├── gateway/                      # .NET Gateway service
│   ├── Controllers/
│   │   └── RagController.cs      # REST endpoints (/api/chat, /api/ingest, etc.)
│   ├── Agents/
│   │   ├── QueryRouterAgent.cs   # Intent classification
│   │   ├── ChitChatAgent.cs      # Casual conversation
│   │   └── GroundedAnswerAgent.cs # Document-grounded answers
│   ├── Services/
│   │   ├── WorkerClient.cs       # HTTP client to Worker
│   │   ├── QdrantDocumentService.cs # Document CRUD via gRPC
│   │   └── CitationValidator.cs  # Citation audit loop
│   ├── appsettings.json          # Default configuration
│   └── Program.cs                # DI setup, middleware pipeline
│
├── worker/                       # Python Worker service
│   ├── app/
│   │   ├── core/
│   │   │   └── config.py         # All settings (Pydantic)
│   │   ├── features/
│   │   │   ├── ingestion/        # Parse, chunk, embed, upsert
│   │   │   └── retrieval/        # Search, rewrite, rerank
│   │   └── main.py               # FastAPI app
│   └── requirements.txt
│
├── streamlit_app.py              # Web UI
├── docker-compose.yml            # Qdrant container
├── .env.example                  # Environment variable template
└── docs/                         # This documentation
```

---

## Setup and Running

### Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.10+ | Worker runtime |
| .NET SDK | 10.0+ | Gateway runtime |
| Docker | Latest | Qdrant container |
| Google API Key | — | Gemini LLM access |

### Step 1: Configure Secrets

```bash
# Generate internal API key
openssl rand -hex 24
# Example: 711f9fdf363c689d23b475b47ae826327ce351e1496901ee

# Create .env from template
cp .env.example .env
```

Edit `.env`:

```env
INTERNAL_API_KEY="711f9fdf363c689d23b475b47ae826327ce351e1496901ee"
GOOGLE_API_KEY="AIzaSy..."
```

Create `gateway/appsettings.Development.json`:

```json
{
  "Gateway": {
    "InternalApiKey": "711f9fdf363c689d23b475b47ae826327ce351e1496901ee"
  },
  "Gemini": {
    "ApiKey": "AIzaSy..."
  }
}
```

> **Critical:** `INTERNAL_API_KEY` and `GOOGLE_API_KEY` must be identical in both `.env` and `appsettings.Development.json`.

### Step 2: Start Qdrant

```bash
docker compose up -d
# Verify: curl http://localhost:6333/healthz
```

### Step 3: Start Worker

```bash
cd worker
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python3 main.py
# Wait for: "Uvicorn running on http://0.0.0.0:8000"
```

### Step 4: Start Gateway

```bash
cd gateway
export $(grep -v '^#' ../.env | xargs)
dotnet run
# Wait for: "Now listening on: http://localhost:8080"
```

### Step 5: Start UI (Optional)

```bash
streamlit run streamlit_app.py
# Opens at: http://localhost:8501
```

---

## API Reference

### Ingestion

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/ingest` | Upload PDF(s) for ingestion |

### Chat

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/chat` | Non-streaming chat (full response) |
| `POST` | `/api/chat/stream` | Streaming chat (SSE, token-by-token) |

### Documents

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/documents` | List all ingested documents |
| `GET` | `/api/documents/{docId}` | Get document details |
| `DELETE` | `/api/documents/{docId}` | Delete a document |
| `DELETE` | `/api/documents/by-filename?filename=...` | Delete by filename |
| `DELETE` | `/api/reset` | Delete all documents |

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Gateway handles all LLM calls | Centralizes retry/rate-limit logic, single API key management |
| Worker handles all ML workloads | GPU/CPU-intensive tasks isolated from the HTTP-serving Gateway |
| Qdrant as standalone Docker | Simple deployment, persistent volume, accessible dashboard |
| Streamlit as optional UI | Rapid prototyping, not required for API-first usage |

**Next:** [05 — Data Flow](05-data-flow.md) traces end-to-end request flows through both services.
