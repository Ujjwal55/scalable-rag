# Architecture Overview

This project is an Enterprise Agentic RAG (Retrieval-Augmented Generation) system.

## Core Technologies

1. **Package Manager & Environment**: `uv`
2. **Vector Database**: Qdrant (Cloud/Cluster)
3. **Embeddings**: 
   - Primary: Google Gemini (`gemini-embedding-2-preview`, 3072 dimensions)
   - Fallback: Local Sentence Transformers (`all-mpnet-base-v2`, 768 dimensions)
4. **Agent Framework & Memory**: LangGraph with Postgres Checkpointer
5. **Semantic Reranking**: Jina Reranker v3 (`jina-reranker-v3`) via API
6. **LLM Gateway**: Portkey (caching, fallbacks, retry logic)
7. **Observability & Logging**: Pydantic Logfire, Prometheus (metrics)
8. **API & Security**: FastAPI with Guardrails and Redis-backed slowapi rate limiting
9. **UI**: Streamlit

## Directory Structure

```text
app/
├── config.py              # Environment configuration & Settings singleton
├── main.py                # FastAPI entry point, metrics, guardrails, rate limits
├── ui/                    # Streamlit frontend (app.py)
├── agents/                # LangGraph Agent workflow
│   ├── graph.py           # StateGraph builder and checkpointer init
│   ├── state.py           # AgentState TypedDict definitions
│   └── nodes/             # Graph nodes (planner, retriever, responder)
├── ingestion/             # Pipeline for processing and embedding documents
│   ├── chunking/          # Text splitting logic
│   ├── loaders/           # File-specific parsers (PDF, HTML, Text, Office)
│   └── processor.py       # Main orchestration script for universal ingestion
└── services/              # Shared business logic and external integrations
    └── retrieval/         # Embedding generation, Qdrant querying, Jina Reranking
```

## Configuration (`app/config.py`)

The application is configured using a `.env` file mapped to a `Settings` class.

**Key Variables:**
- `GEMINI_API_KEY`: Required for Gemini embeddings.
- `QDRANT_CLUSTER_ENDPOINT` & `QDRANT_API_KEY`: Required for vector storage.
- `GROQ_API_KEY` & `GROQ_FALLBACK_API_KEY`: Used for generation.
- `LOGFIRE_TOKEN` & `LOGFIRE_BASE_URL`: Observability (if missing, `processor.py` implements a fallback logic to infer the URL).

## Observability

The project uses `logfire` extensively. Instead of standard Python logging or `print` statements, always use `logfire.info()`, `logfire.error()`, and `with logfire.span("Span Name"):` to ensure telemetry is captured properly.
