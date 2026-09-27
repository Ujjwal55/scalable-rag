# LangGraph Agent Workflow

This repository uses LangGraph to orchestrate the RAG (Retrieval-Augmented Generation) flow, providing a stateful, memory-aware conversational AI.

## The State (`app/agents/state.py`)
The system state is a `TypedDict` containing:
- `messages`: A list of conversation history (with `operator.add` so memory appends instead of overwriting).
- `current_query`: The active search intent.
- `documents`: The retrieved and reranked context blocks.
- `status` & `plan`: Telemetry and UI-facing thought processes.
- `final_answer`: The final synthesized LLM output.

## The Nodes

### 1. `planner_node` (`app/agents/nodes/planner.py`)
- Analyzes the full conversation history.
- Classifies intent:
  - Returns `"CONVERSATIONAL"` if the query is a greeting or relies entirely on memory.
  - Returns a refined search term if technical knowledge is required.

### 2. `retrieve_node` (`app/agents/nodes/retriever.py`)
- Executes only if the planner outputs a technical search term.
- Queries Qdrant (top 15).
- Reranks using Jina (top 5).

### 3. `generate_node` (`app/agents/nodes/responder.py`)
- Synthesizes the final answer using either the full memory (if conversational) or the strict RAG context + memory (if technical).
- Uses `portkey_client` (via `get_langchain_llm`) to natively track cache hits via the `x-portkey-cache-status` header.

## The Graph (`app/agents/nodes/graph.py`)
- Compiles the above nodes into a cyclic state graph.
- Implements a **Conditional Edge**: `planner -> retrieve` (if technical) or `planner -> responder` (if conversational).
- **Checkpointer Memory**: Uses a Postgres database connection pool (`langgraph.checkpoint.postgres`) via Neon to provide durable, production-ready memory. Falls back to in-memory `MemorySaver` if the database is unreachable.
