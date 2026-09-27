# API and UI Architecture

The system is separated into a FastAPI backend providing the RAG endpoints and a Streamlit frontend providing the user chat interface.

## FastAPI Backend (`app/main.py`)
This is the primary entry point for the production application. It wraps the LangGraph agent in a robust, observable API.

- **Security & Rate Limiting**: 
  - Routes are protected by an optional Bearer Token (`RAG_API_KEY`).
  - Utilizes `slowapi` backed by Redis for strict rate-limiting (falls back to in-memory if Redis is offline).
- **Guardrails**: Integrated heavily with `NeMo Guardrails` (`guard()` function). Synchronously blocks harmful or out-of-scope prompts before they ever reach the LangGraph pipeline or LLM.
- **Metrics (Prometheus)**: Exposes a `/metrics` endpoint, instrumenting `rag_requests_total`, `rag_request_duration_seconds`, and `guardrails_blocks_total`.
- **Endpoints**:
  - `GET /` and `/health` for checks.
  - `GET /graph`: Generates a Mermaid PNG image of the LangGraph state machine.
  - `POST /query`: The main synchronous RAG endpoint. It triggers the LangGraph agent, collects the `final_answer`, `status`, `thought_process`, and `sources`, and returns them to the client.

## Streamlit UI (`app/ui/app.py`)
Provides a rich chat interface to interact with the backend API.

- **Initialization**: Automatically boots up Logfire to trace UI events (like `💬 User Chat Interaction`).
- **Session Management**: Uses `st.session_state` and UUIDs for `thread_id`, maintaining persistent conversational memory inside the backend Postgres checkpointer.
- **Visuals**:
  - A persistent sidebar shows Memory IDs, Logfire status, and a button to wipe history.
  - Streams the final response text back to the user token-by-token (simulated stream).
  - Uses nested expanders (`st.expander`) to cleanly display the thought process and the raw retrieved context chunks.
