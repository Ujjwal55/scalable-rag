# Retrieval Flow & Embeddings

The retrieval pipeline handles vector search and generating embeddings for both ingestion and runtime querying.

## `app/services/retrieval/embeddings.py`

This module is a singleton manager for the embedding model.

### Model Loading & Fallback
The `_init()` function guarantees the embedding model is loaded only once per process.

1. **Probe Gemini**: It attempts to load `gemini-embedding-2-preview` using the Google API. It fires a dummy query ("probe").
2. **Success**: If the probe succeeds, the process locks into Gemini (3072 dimensions) for its lifespan.
3. **Fallback**: If the probe fails (e.g. no API key, or immediate API outage), it falls back to `sentence-transformers` running locally (`all-mpnet-base-v2`, 768 dimensions). 

*Note on Qdrant Dimensionality*: Qdrant collections are strictly bound to the dimension size created at instantiation. If you switch models mid-development, you must `--wipe` the collection to recreate it with the correct dimensions.

### Batching & Resilience
- `embed_texts()` processes chunks in batches (default size: 50).
- `_embed_batch()` executes the API call. If a `429` rate limit is hit, it pauses using exponential backoff.
- If it fails 4 times, it raises a custom `RateLimitError` which bubbles up to the caller (e.g. `processor.py`) to trigger a global circuit breaker.

## `app/services/retrieval/ranking_service.py`
This module acts as a secondary retrieval step (Semantic Reranking) to ensure only the most relevant Qdrant results are passed to the LLM.

- **Jina Reranker v3**: Uses `jina-reranker-v3` via the Jina API (`_JinaReranker`).
- **Resilience**: 
  - Wraps the API call in a Tenacity retry block (`@retry` for 3 attempts with exponential backoff).
  - If the API fails entirely or `JINA_API_KEY` is missing, it logs a warning and cleanly falls back to the original Qdrant vector search order.

## RAG Retrieval Pipeline
Currently, the retrieval endpoints use:
1. `qdrant_client.search()` with embeddings to get the top 15 results.
2. `rerank_documents()` to reorder them and keep the top 5 most semantically relevant chunks.
