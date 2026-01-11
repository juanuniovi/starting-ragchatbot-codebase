# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Start the development server (from project root)
cd backend && uv run uvicorn app:app --reload --port 8000

# Or use the shell script
./run.sh

# Alternative without uv (requires venv with dependencies)
source .venv/bin/activate && cd backend && python -m uvicorn app:app --reload --port 8000
```

Server runs at http://localhost:8000 (web UI) and http://localhost:8000/docs (API docs).

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot for course materials with a FastAPI backend and vanilla JS frontend.

### Data Flow

```
User Query → Frontend → POST /api/query → RAGSystem.query()
    → SessionManager (get conversation history)
    → AIGenerator (Claude API with tools)
    → Claude decides to use search_course_content tool
    → CourseSearchTool.execute()
    → VectorStore.search() (ChromaDB semantic search)
    → Results formatted and returned to Claude
    → Claude generates final answer
    → Response with sources returned to frontend
```

### Backend Components (`backend/`)

- **app.py**: FastAPI server with `/api/query` and `/api/courses` endpoints. Loads documents from `../docs` on startup.
- **rag_system.py**: Main orchestrator connecting all components. Initializes DocumentProcessor, VectorStore, AIGenerator, SessionManager, and ToolManager.
- **document_processor.py**: Parses course files (expects `Course Title:`, `Course Instructor:`, `Lesson N:` format), chunks text into ~800 char segments with 100 char overlap.
- **vector_store.py**: ChromaDB wrapper with two collections: `course_catalog` (course metadata for fuzzy name matching) and `course_content` (searchable chunks). Uses `all-MiniLM-L6-v2` embeddings.
- **ai_generator.py**: Claude API integration with tool calling. Handles tool execution loop when Claude uses `search_course_content`.
- **search_tools.py**: Defines `CourseSearchTool` (Anthropic tool format) and `ToolManager` for registering/executing tools.
- **session_manager.py**: In-memory conversation history per session (max 2 exchanges retained).
- **config.py**: Configuration via environment variables. Requires `ANTHROPIC_API_KEY` in `.env`.
- **models.py**: Pydantic models for Course, Lesson, CourseChunk.

### Frontend (`frontend/`)

Static files served by FastAPI. Uses `marked.js` for markdown rendering. Maintains session ID for conversation context.

### Key Configuration (config.py)

- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `CHUNK_SIZE`: 800 chars
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges

### Course Document Format (`docs/`)

```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: Introduction
Lesson Link: [url]
[content...]

Lesson 1: [title]
[content...]
```
