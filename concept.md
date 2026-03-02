# RAG System — Documentation

This is the architecture and design documentation for the **RAG System** — an enterprise document Q&A system that combines retrieval and AI generation.

## Document Index

| # | Document | Topic |
|---|----------|-------|
| 00 | [Overview](00-overview.md) | Introduction, architecture, glossary |
| 01 | [Concepts & Visual Guide](01-concepts.md) | Visual deep-dive: LLM, RAG, screenshots, code tour |
| 02 | [Ingestion Pipeline](02-ingestion-pipeline.md) | Parser, chunking, enrichment, embedding, vector storage |
| 03 | [Retrieval Pipeline](03-retrieval-pipeline.md) | Query routing, hybrid search, fusion, reranking |
| 04 | [Generation Pipeline](04-generation-pipeline.md) | Multi-agent orchestration, citation validation |
| 05 | [Application](05-application.md) | Project structure, setup, code tour |
| 06 | [Data Flow](06-data-flow.md) | End-to-end ingestion and chat flows, evaluation |
| 07 | [Configuration Reference](07-configuration.md) | All settings in one page |
| 08 | [Observability](08-observability.md) | Structured logging and debugging |
| 09 | [Reliability](09-reliability.md) | Retry, timeout, rate limiting |
| 10 | [Troubleshooting](10-troubleshooting.md) | Symptom → diagnosis → solution |
| 11 | [Example: Hello RAG](11-example-hello-rag.md) | Smallest working flow from zero |
| 12 | [Example: Ingestion](12-example-ingestion.md) | Detailed ingestion walkthrough |
| 13 | [Example: Query](13-example-query.md) | Detailed query walkthrough |
| 14 | [Example: Customization](14-example-customization.md) | Swap models, tune retrieval |

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Gateway | .NET (ASP.NET Core) |
| Worker | Python (FastAPI) |
| Vector DB | Qdrant |
| UI | Streamlit |
| Embedding | BAAI/bge-m3 |
| Reranker | BAAI/bge-reranker-base |
| LLM | Gemini 2.5 Flash (default) |
