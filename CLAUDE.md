# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Setup
```bash
# Install Python dependencies (requires Python 3.13+)
uv sync

# Set up environment variables
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start (if you need more control)
cd backend && uv run uvicorn app:app --reload --port 8000

# Access points:
# - Web UI: http://localhost:8000
# - API docs: http://localhost:8000/docs
```

### Windows Note
Windows users should use Git Bash to run shell scripts.

## Architecture Overview

### RAG System Design Pattern

This codebase implements a **tool-calling RAG architecture** where Claude decides when to search, rather than always searching first. The flow is:

1. User query → FastAPI → RAG System
2. RAG System sends query to Claude with tool definitions
3. Claude analyzes if search is needed
4. If yes: Claude calls `search_course_content` tool
5. Tool manager executes search → returns results to Claude
6. Claude synthesizes final answer from search results
7. Response + sources returned to UI

This differs from traditional RAG (always search first) by allowing Claude to answer general questions without searching.

### Two-Collection Vector Store Strategy

The `VectorStore` class uses **two separate ChromaDB collections**:

1. **`course_catalog`** - Stores course-level metadata
   - Document: Course title (searchable)
   - Metadata: Instructor, course link, lessons JSON, lesson count
   - ID: Course title (enables deduplication)
   - Purpose: Fuzzy course name resolution via semantic search

2. **`course_content`** - Stores chunked course content
   - Document: Text chunks (800 chars, 100 overlap)
   - Metadata: Course title, lesson number, chunk index
   - ID: `{course_title}_{chunk_index}`
   - Purpose: Semantic content search with filtering

**Why two collections?** When a user searches with a fuzzy course name (e.g., "AI basics" instead of exact title "Introduction to AI"), the system:
1. Queries `course_catalog` to find the best matching course title
2. Uses resolved title to filter searches in `course_content`
3. Prevents cross-course contamination in search results

### Component Interaction Flow

**Document Ingestion (app.py startup):**
```
docs/*.txt files
  → DocumentProcessor.process_course_document()
    → Parses structured format (headers + lessons)
    → Smart sentence-based chunking with context prefixes
  → RAGSystem.add_course_folder()
    → Checks existing_course_titles to prevent duplicates
    → VectorStore.add_course_metadata() (catalog collection)
    → VectorStore.add_course_content() (content collection)
```

**Query Processing:**
```
POST /api/query {query, session_id}
  → RAGSystem.query()
    → SessionManager.get_conversation_history() (last 2 exchanges)
    → AIGenerator.generate_response()
      → Claude receives query + tool definitions + history
      → Claude calls search_course_content tool (if needed)
      → ToolManager.execute_tool()
        → CourseSearchTool.execute()
          → VectorStore.search() with optional course/lesson filters
          → Returns SearchResults
      → Claude receives search results
      → Claude generates final answer
    → ToolManager.get_last_sources() (for UI citations)
    → SessionManager.add_exchange() (update history)
  → Returns {response, sources}
```

### Critical Configuration Points

**backend/config.py** - Centralized settings:
- `CHUNK_SIZE: 800` - Chunk length affects retrieval granularity
- `CHUNK_OVERLAP: 100` - Prevents context loss at boundaries
- `MAX_RESULTS: 5` - Search result limit
- `MAX_HISTORY: 2` - Conversation exchanges kept (2 exchanges = 4 messages)
- `EMBEDDING_MODEL: "all-MiniLM-L6-v2"` - Local embeddings, no API costs

### Document Format Requirements

Course documents in `docs/` must follow this structure:
```
Course Title: [exact title]
Course Link: [url]
Course Instructor: [name]

Lesson 1: [title]
Lesson Link: [url]
[content...]

Lesson 2: [title]
Lesson Link: [url]
[content...]
```

The parser (`document_processor.py`) expects this exact format. Deviations will cause parsing errors.

### Session Management Behavior

Sessions are **ephemeral** (in-memory only):
- Auto-created when client sends `session_id`
- Stores last `MAX_HISTORY` exchanges (default: 2)
- **Lost on server restart** - no persistence
- Frontend generates UUID and maintains it across page loads via localStorage

### Tool Calling Implementation

Tools are defined using Anthropic's tool use pattern in `search_tools.py`:
- `Tool` base class with `get_definition()` and `execute()` methods
- `CourseSearchTool` implements search with parameters: `query`, `course_name`, `lesson_number`
- `ToolManager` registers tools and handles execution
- Sources tracked separately for UI display

Claude's system prompt explicitly instructs it to search **at most once per query** to prevent over-searching.

### ChromaDB Persistence

ChromaDB data is stored in `backend/chroma_db/` (gitignored):
- **Persistent across restarts** - no need to re-index on startup
- Startup checks `existing_course_titles` to avoid duplicate ingestion
- Use `RAGSystem.add_course_folder(clear_existing=True)` to force rebuild

### Frontend Architecture

Single-page app with no frameworks:
- `script.js` manages state: sessionId (localStorage), loading states
- Markdown rendering via Marked.js
- Source citations displayed in collapsible `<details>` elements
- Suggested questions for quick start
- Course statistics fetched from `/api/courses`

## Key Files by Function

**Core Orchestration:**
- `backend/app.py` - FastAPI app, endpoints, startup document loading
- `backend/rag_system.py` - Main coordinator, ingestion + query logic

**Search & Retrieval:**
- `backend/vector_store.py` - ChromaDB wrapper, two-collection strategy
- `backend/search_tools.py` - Tool definitions and execution

**AI Integration:**
- `backend/ai_generator.py` - Claude API calls, tool calling loop

**Document Processing:**
- `backend/document_processor.py` - Parsing, sentence-based chunking

**State Management:**
- `backend/session_manager.py` - In-memory conversation history
- `frontend/script.js` - Client session persistence
