# 13 — Example: Customization

This example demonstrates how to customize the RAG system's behavior by modifying configuration files. No code changes are required for most adjustments.

---

## Customization 1: Swap the LLM Model

### Change All Agents to GPT-4o-mini

Edit `gateway/appsettings.json`:

```json
{
  "Gateway": {
    "RouterModel": "gpt-4o-mini",
    "ChitChatModel": "gpt-4o-mini",
    "GroundedAnswerModel": "gpt-4o-mini"
  }
}
```

Add your OpenAI key to `gateway/appsettings.Development.json`:

```json
{
  "OpenAI": {
    "ApiKey": "sk-proj-..."
  }
}
```

### Use a Local LLM (Ollama)

For complete offline operation:

1. Install [Ollama](https://ollama.com) and pull a model:

```bash
ollama pull qwen2.5:14b
```

2. Update Gateway config to point to Ollama:

```json
{
  "Gateway": {
    "GroundedAnswerModel": "qwen2.5:14b"
  }
}
```

| Model | Context Window | Privacy | Hardware Required |
|-------|---------------|---------|-------------------|
| `gemini-2.5-flash` | 1M tokens | Cloud API | Internet only |
| `gpt-4o-mini` | 128K tokens | Cloud API | Internet only |
| `qwen2.5:14b` | 128K tokens | 100% local | 16GB+ VRAM GPU |
| `qwen2.5:32b` | 128K tokens | 100% local | 24GB+ VRAM GPU |

---

## Customization 2: Tune Chunking Parameters

Edit `worker/app/core/config.py`:

### Large Documents (Books, Legal Texts)

```python
CHUNK_SIZE = 2048       # Larger chunks = more context per chunk
CHUNK_OVERLAP = 400     # More overlap = better continuity
```

### Short Documents (Memos, SOPs)

```python
CHUNK_SIZE = 512        # Smaller chunks = more precise search
CHUNK_OVERLAP = 100     # Less overlap needed for short docs
```

> **Note:** After changing chunking parameters, you must re-ingest all documents:
> ```bash
> curl -X DELETE http://localhost:8080/api/reset
> # Re-upload all PDFs
> ```

---

## Customization 3: Tune Retrieval Parameters

Edit `worker/app/core/config.py`:

### More Keyword-Focused Search

```python
HYBRID_ALPHA = 0.1      # 90% keyword, 10% semantic
```

Best for: technical documents with specific codes, article numbers, regulations.

### More Semantic Search

```python
HYBRID_ALPHA = 0.7      # 30% keyword, 70% semantic
```

Best for: narrative documents, meeting minutes, essays.

### Quick Reference: Alpha Values

| Value | Keyword/Semantic | Best For |
|-------|-----------------|----------|
| 0.0 | 100% keyword | Article numbers, exact codes |
| 0.3 | 70/30 (default) | Mixed documents |
| 0.5 | 50/50 | General purpose |
| 0.7 | 30/70 | Narrative content |
| 1.0 | 100% semantic | Conceptual questions |

### Adjust Number of Sources

```python
MAX_SOURCES = 8          # More sources (default: 5)
MAX_SOURCES_PER_DOC = 3  # Allow more chunks per document (default: 2)
RERANK_TOP_K = 12        # Keep more candidates from reranking (default: 8)
```

---

## Customization 4: Enable Vision AI

For PDFs containing charts, graphs, or diagrams:

Edit `worker/app/core/config.py`:

```python
ENABLE_PICTURE_DESCRIPTION = True
PICTURE_DESCRIPTION_MODEL = "gemini-2.0-flash"  # Cloud vision model
PICTURE_DESCRIPTION_ONLY_CHARTS = True           # Skip photos, only charts
```

**Impact:**
- ✅ Charts become searchable via text descriptions
- ⚠️ Ingestion time increases (1-3s per image)
- ⚠️ Requires Gemini API key with vision access

---

## Customization 5: Change Query Rewrite Mode

Edit `worker/app/core/config.py`:

### Use LLM for Query Rewriting

```python
QUERY_REWRITE_MODE = "llm"    # Better for complex queries
```

| Mode | Speed | Cost | Quality |
|------|-------|------|---------|
| `rule` | <1ms | Free | Good for typical queries |
| `llm` | 1-3s | API cost/query | Better for ambiguous queries |

---

## Customization 6: Adjust Timeouts

Edit `gateway/appsettings.json`:

```json
{
  "Gateway": {
    "WorkerTimeoutSeconds": 1200
  }
}
```

Increase `WorkerTimeoutSeconds` if ingesting very large PDFs (200+ pages).

---

## Summary: Configuration Locations

| What to Change | Where | Requires Re-Ingest? |
|---------------|-------|:---:|
| LLM model | `appsettings.json` | No |
| Chunk size/overlap | `config.py` | ✅ Yes |
| Retrieval (alpha, top-k) | `config.py` | No |
| Vision AI | `config.py` | ✅ Yes |
| Query rewrite mode | `config.py` | No |
| Timeouts | `appsettings.json` | No |
| API keys | `.env` + `appsettings.Development.json` | No |

---

## Key Takeaways

| Concept | What You Learned |
|---------|-----------------|
| **No-code tuning** | Most parameters adjustable via config files |
| **Re-ingest required** | Only when chunking or enrichment parameters change |
| **Alpha balance** | The single most impactful tuning knob for search quality |
| **Local LLM option** | Full offline operation with Ollama |

**Back to:** [Document Index](concept.md)
