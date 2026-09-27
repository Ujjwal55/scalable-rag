# Ingestion Flow

The ingestion pipeline is designed to be highly fault-tolerant, rate-limit aware, and robust against multiple file types. 

## Entry Point: `app/ingestion/processor.py`

The main orchestration script is `processor.py`. It is run via:
```bash
uv run python -m app.ingestion.processor <DIRECTORY_PATH> [explicit_source_type] [--wipe]
```

- **`run_universal_ingestion()`**: The top-level function. It checks if the provided directory contains subdirectories. If it does, it routes each subdirectory to `process_directory()`.
- **`process_directory()`**: Iterates through all files in a folder and calls `process_file()`. This function implements a **Circuit Breaker** to halt execution if rate limits persist across multiple files.
- **`process_file()`**: The core pipeline for a single file.

## Pipeline Steps (`process_file`)

1. **Routing & Parsing**: Based on file extension, the file is routed to a specific loader in `app/ingestion/loaders/`:
   - `.pdf`: `parse_pdf`
   - `.html`/`.htm`: `parse_html`
   - `.txt`/`.md`/`.csv`: `parse_text`
   - `.docx`/`.pptx`: `parse_office`
2. **Chunking**: `app/ingestion/chunking/splitter.py` splits the extracted text into manageable overlapping chunks.
3. **Local Checkpoint**: The chunks and metadata are saved locally as JSON to `processed_data/` as a backup.
4. **Embedding & Indexing**: The chunks are passed to `embed_texts()`, and the resulting vectors are inserted into Qdrant.

## Circuit Breaker & Rate Limiting

The ingestion script makes heavy use of the Google Gemini API, which is prone to `429 RESOURCE_EXHAUSTED` errors on free tiers.

- **Micro-Retries**: `embeddings.py` will catch a rate limit and attempt exponential backoff (1s -> 2s -> 4s).
- **Macro-Circuit Breaker**: If `embeddings.py` exhausts retries, it throws a `RateLimitError`. `process_directory()` catches this and implements strikes:
  - **Strike 1**: Pause 60s
  - **Strike 2**: Pause 120s
  - **Strike 3**: Fatal exit (prevents spamming API).
