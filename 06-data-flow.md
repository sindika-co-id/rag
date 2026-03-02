# 06 — Data Flow

This document traces the complete data flow for the two primary operations: **ingestion** (uploading a document) and **chat** (asking a question). It also covers evaluation metrics and migration history.

---

## Ingestion Flow

When a user uploads a PDF, the following sequence executes end-to-end:

```mermaid
sequenceDiagram
    participant User
    participant Gateway
    participant Worker
    participant Qdrant

    User->>Gateway: POST /api/ingest (PDF file)
    Gateway->>Worker: Proxy PDF to /api/ingest

    Note over Worker: Docling: Parse PDF → Markdown
    Note over Worker: Normalize: clean headers, footers
    Note over Worker: Compute doc_id (content hash)
    
    Worker->>Qdrant: Delete old chunks (if re-upload)
    
    Note over Worker: Schema v2: Split into chunks
    Note over Worker: Enrichment: Vision AI (if enabled)
    Note over Worker: BGE-M3: Generate embeddings

    loop Batch upsert (size=20)
        Worker->>Qdrant: Upsert vectors + metadata
    end

    Worker-->>Gateway: {filename, chunks, doc_id}
    Gateway-->>User: Status response
```

### Duplicate Detection

Documents are identified by a **deterministic content hash**. Re-uploading the same content returns `status: duplicate_skipped`. Re-uploading a modified version of the same filename deletes old chunks first, then re-ingests.

---

## Chat Flow

When a user asks a question, the following sequence executes:

```mermaid
sequenceDiagram
    participant User
    participant Gateway
    participant Router as Query Router (LLM)
    participant Worker
    participant Qdrant
    participant GA as Grounded Answer (LLM)
    participant CV as Citation Validator

    User->>Gateway: POST /api/chat

    Gateway->>Router: Classify intent
    
    alt SEARCH / SUMMARIZATION
        Router-->>Gateway: {intent: SEARCH, rewritten_query}
        Gateway->>Worker: /internal/retrieve-context
        
        Note over Worker: Rule-based query rewrite
        Note over Worker: BGE-M3 embed query
        
        Worker->>Qdrant: Hybrid search (Dense + Sparse)
        Qdrant-->>Worker: Raw candidates
        
        Note over Worker: RRF Fusion
        Note over Worker: Cross-Encoder Rerank → top-8
        Note over Worker: Table parent expansion
        Note over Worker: Diversity filter → top-5
        
        Worker-->>Gateway: Sources [S1]..[S5]
        Gateway->>GA: Context + question
        GA-->>Gateway: Answer with [S1], [S2] citations
        
        Gateway->>CV: Validate citations
        CV-->>Gateway: ✅ Valid
        Gateway-->>User: Answer + sources
        
    else CHIT_CHAT
        Router-->>Gateway: {intent: CHIT_CHAT}
        Note over Gateway: Chit-Chat Agent responds directly
        Gateway-->>User: Casual response (no search)
        
    else AMBIGUOUS
        Router-->>Gateway: {intent: AMBIGUOUS}
        Gateway-->>User: Clarification request
    end
```

### Conversation History

Follow-up questions include conversation history (up to `CONVERSATION_HISTORY_MAX_MESSAGES = 8`). The Router uses this history to resolve pronouns and context:

- User: `"Apa sanksi merokok?"` → SEARCH
- User: `"Bagaimana kalau pelanggaran ketiga?"` → Router understands "pelanggaran ketiga" refers to smoking violations from the previous turn

---

## Evaluation

The system measures answer quality across three dimensions:

### Retrieval Quality

| Metric | What It Measures | Good Value |
|--------|-----------------|------------|
| **Context Precision** | Are the retrieved chunks relevant? | > 0.7 |
| **Context Recall** | Are all relevant chunks found? | > 0.6 |

### Answer Quality

| Metric | What It Measures | Good Value |
|--------|-----------------|------------|
| **Faithfulness** | Is the answer grounded in sources? | > 0.8 |
| **Answer Relevance** | Does the answer address the question? | > 0.7 |
| **Groundedness** | Word overlap between answer and sources | > 0.4 |

### Testing Approach

Evaluation uses a dataset of question-answer-context triples. Each test case specifies:
- **Question:** input query
- **Ground truth:** expected answer
- **Expected sources:** which documents should be retrieved

---

## Migration History

The system evolved from a Python monolith to the current two-service architecture:

| Version | Architecture | Key Change |
|---------|-------------|------------|
| v1 | Python monolith (Streamlit + LangChain) | Single file, basic RAG |
| v2 | Python monolith (FastAPI + LlamaIndex) | Structured pipeline, hybrid search |
| v3 (current) | Python Worker + .NET Gateway | Enterprise features: citation validation, multi-agent, Polly resilience |

### Why the Migration?

| Problem in v2 | Solution in v3 |
|---------------|----------------|
| LLM errors crash the whole app | Gateway isolates LLM concerns with retry/timeout |
| No citation validation | .NET Gateway adds regex + LLM validation loop |
| Single-provider lock-in | AgentOrchestration library supports Gemini/OpenAI/Claude |
| No structured logging | .NET structured logging + InvocationId tracing |

**Next:** [06 — Configuration Reference](07-configuration.md) documents every setting in one page.
