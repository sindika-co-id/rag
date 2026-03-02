# RAG System — Documentation

This is the architecture and design documentation for the **RAG System** — an enterprise document Q&A system that combines retrieval and AI generation.

## Document Index

| # | Document | Topic |
|---|----------|-------|
| 00 | [Overview](00-overview.md) | Introduction, architecture, glossary |
| 01 | [Ingestion Pipeline](01-ingestion-pipeline.md) | Parser, chunking, enrichment, embedding, vector storage |
| 02 | [Retrieval Pipeline](02-retrieval-pipeline.md) | Query routing, hybrid search, fusion, reranking |
| 03 | [Generation Pipeline](03-generation-pipeline.md) | Multi-agent orchestration, citation validation |
| 04 | [Application](04-application.md) | Project structure, setup, code tour |
| 05 | [Data Flow](05-data-flow.md) | End-to-end ingestion and chat flows, evaluation |
| 06 | [Configuration Reference](06-configuration.md) | All settings in one page |
| 07 | [Observability](07-observability.md) | Structured logging and debugging |
| 08 | [Reliability](08-reliability.md) | Retry, timeout, rate limiting |
| 09 | [Troubleshooting](09-troubleshooting.md) | Symptom → diagnosis → solution |
| 10 | [Example: Hello RAG](10-example-hello-rag.md) | Smallest working flow from zero |
| 11 | [Example: Ingestion](11-example-ingestion.md) | Detailed ingestion walkthrough |
| 12 | [Example: Query](12-example-query.md) | Detailed query walkthrough |
| 13 | [Example: Customization](13-example-customization.md) | Swap models, tune retrieval |

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
