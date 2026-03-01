# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# From the repo root (recommended)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

The server starts at `http://localhost:8000`. The frontend is served as static files from `/`.
API docs available at `http://localhost:8000/docs`.

## Setup

```bash
uv sync                        # Install dependencies
cp .env.example .env           # Then add your ANTHROPIC_API_KEY
```

## Architecture

This is a full-stack RAG chatbot. The backend runs from the `backend/` directory; all Python imports are relative to it (no package structure).

### Key design decisions

**Two-phase Claude interaction** (`ai_generator.py`): Every query makes up to 2 API calls. The first call gives Claude access to the `search_course_content` tool. If Claude invokes it (`stop_reason == "tool_use"`), the tool executes, and a second call is made with the results injected as a `tool_result` message. If Claude answers from general knowledge, only one call is made.

**Two ChromaDB collections** (`vector_store.py`):
- `course_catalog` — one document per course (title, instructor, link). Used for fuzzy course name resolution via semantic search.
- `course_content` — chunked lesson text with metadata (`course_title`, `lesson_number`, `chunk_index`). Used for actual content retrieval.

**Document format** (`document_processor.py`): `.txt` files in `docs/` must follow this structure:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <title>
...
```
`course_title` (string) is the unique key across both collections — not a generated ID.

**Session history** (`session_manager.py`): Stored in-memory as a dict keyed by `session_N`. History is serialized as a plain string and injected into the system prompt. Max 2 exchanges retained (configurable via `config.MAX_HISTORY`).

**Tool extensibility** (`search_tools.py`): `Tool` is an ABC. New tools must implement `get_tool_definition()` (returns Anthropic tool schema) and `execute(**kwargs)`. Register with `ToolManager.register_tool()` and they're automatically included in API calls.

### Data flow

```
POST /api/query
  → RAGSystem.query()
    → SessionManager.get_conversation_history()
    → AIGenerator.generate_response()        # Claude call #1
      → [if tool_use] CourseSearchTool.execute()
        → VectorStore._resolve_course_name() # semantic match on course_catalog
        → VectorStore.search()               # semantic search on course_content
      → AIGenerator._handle_tool_execution() # Claude call #2 with results
    → SessionManager.add_exchange()
  → returns {answer, sources, session_id}
```

### Configuration (`backend/config.py`)

All tunable parameters live here: `CHUNK_SIZE` (800), `CHUNK_OVERLAP` (100), `MAX_RESULTS` (5), `MAX_HISTORY` (2), `CHROMA_PATH` (`./chroma_db`), `EMBEDDING_MODEL` (`all-MiniLM-L6-v2`), `ANTHROPIC_MODEL`.

ChromaDB persists to `backend/chroma_db/`. Delete this directory to reset the vector store and force re-ingestion of documents on next startup.

Documents are loaded on server startup (`app.py` `startup_event`). Already-indexed courses (matched by title) are skipped.
