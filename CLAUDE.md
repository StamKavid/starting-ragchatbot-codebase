# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system that answers questions about course materials. The system uses Claude's tool-calling capability to autonomously decide when to search course content via semantic vector search.

**Key architectural pattern**: Tool-based RAG where Claude decides whether to search rather than always searching. The AI receives a `search_course_content` tool and autonomously determines if the query requires course material lookup or can be answered from general knowledge.

**IMPORTANT**: This project uses `uv` for package management and running commands. **Never use `pip` directly** - always use `uv` commands.

## Development Commands

### Running the Application

```bash
# Quick start (recommended)
./run.sh

# Manual start (from root)
cd backend && uv run uvicorn app:app --reload --port 8000

# Access points
# - Web UI: http://localhost:8000
# - API docs: http://localhost:8000/docs
```

### Dependency Management

```bash
# Install/sync dependencies
uv sync

# Add new dependencies (DO NOT use pip install)
uv add <package-name>

# Remove dependencies
uv remove <package-name>

# Run Python commands (DO NOT use python directly)
uv run python <script.py>
uv run uvicorn app:app --reload

# IMPORTANT: Always use 'uv' for all Python operations
# The project uses uv (not pip) for package management
# Dependencies are defined in pyproject.toml
```

### Environment Setup

Required `.env` file in root directory:
```
ANTHROPIC_API_KEY=your_key_here
```

## Architecture Overview

### Request Flow (Tool-Based RAG Pattern)

1. **Frontend** ([frontend/script.js](frontend/script.js)) → `POST /api/query` with query + session_id
2. **FastAPI** ([backend/app.py](backend/app.py)) → Routes to RAGSystem
3. **RAGSystem** ([backend/rag_system.py](backend/rag_system.py)) → Orchestrates components:
   - Retrieves conversation history from SessionManager
   - Passes query + tools to AIGenerator
4. **AIGenerator** ([backend/ai_generator.py](backend/ai_generator.py)) → Makes first Claude API call:
   - Includes `search_course_content` tool definition
   - Claude autonomously decides whether to use the tool
5. **If Claude calls tool** (stop_reason = "tool_use"):
   - ToolManager executes CourseSearchTool
   - VectorStore performs semantic search on ChromaDB
   - Results returned to Claude via second API call
   - Claude synthesizes answer from search results
6. **If Claude doesn't call tool** (stop_reason = "end_turn"):
   - Claude answers from general knowledge
   - No search performed
7. **Response Assembly**:
   - Sources extracted from tool execution
   - Session history updated (limited to last 2 exchanges = 4 messages)
   - QueryResponse returned to frontend

### Two-Tier Vector Storage

The system maintains **two ChromaDB collections** ([backend/vector_store.py](backend/vector_store.py)):

1. **`course_catalog`**: Stores course metadata for semantic course name matching
   - When user searches with partial course name (e.g., "MCP"), vector search finds best matching course title
   - Enables fuzzy course name matching instead of exact string matching

2. **`course_content`**: Stores chunked course content with metadata
   - Each chunk includes: content, course_title, lesson_number, chunk_index
   - Searchable with filters for course_title and lesson_number
   - Uses `all-MiniLM-L6-v2` embeddings

### Document Processing Pipeline

Documents in [docs/](docs/) are processed at startup ([backend/document_processor.py](backend/document_processor.py)):

1. **Metadata Extraction**: Parses structured headers for course title, instructor, course link
2. **Lesson Detection**: Regex pattern `Lesson N: Title` identifies lesson boundaries
3. **Chunking**: Splits content into ~800 character chunks with 100 char overlap
   - Sentence-based chunking (preserves sentence boundaries)
   - Chunks tagged with lesson context for better search relevance
4. **Deduplication**: Existing courses (by title) are skipped on subsequent runs

### Session Management

Conversation context is maintained per session ([backend/session_manager.py](backend/session_manager.py)):
- Session IDs created on first query: `session_1`, `session_2`, etc.
- History limited to **MAX_HISTORY = 2** exchanges (4 messages total)
- History formatted as string and injected into Claude's system prompt
- Prevents token overflow on long conversations

### Tool Execution Pattern

The tool system ([backend/search_tools.py](backend/search_tools.py)) uses a registration pattern:
- `ToolManager` maintains registry of available tools
- Each tool implements `Tool` abstract class with:
  - `get_tool_definition()`: Returns Anthropic tool schema
  - `execute(**kwargs)`: Performs tool action
- `CourseSearchTool` tracks `last_sources` for UI display
- After query completion, sources are retrieved and reset

## Key Configuration

All settings in [backend/config.py](backend/config.py):

```python
ANTHROPIC_MODEL = "claude-sonnet-4-20250514"
EMBEDDING_MODEL = "all-MiniLM-L6-v2"
CHUNK_SIZE = 800        # Characters per chunk
CHUNK_OVERLAP = 100     # Overlap between chunks
MAX_RESULTS = 5         # Top N search results
MAX_HISTORY = 2         # Conversation exchanges to remember
CHROMA_PATH = "./chroma_db"  # Vector DB storage
```

## Important Patterns to Preserve

### 1. Tool-Based Search Decision

The system prompt ([backend/ai_generator.py](backend/ai_generator.py):8-30) instructs Claude to:
- Use search tool **only** for course-specific questions
- Answer general knowledge questions without searching
- Perform **one search per query maximum**
- Never mention "based on search results" in responses

**Do not change this to "always search"** - the autonomous decision-making is intentional.

### 2. Two-Stage API Call for Tool Use

When Claude decides to use a tool:
1. First API call with tools → Returns tool_use
2. Execute tool(s) → Get results
3. Second API call without tools, with tool results in messages → Get final answer

This is the standard Anthropic tool-use pattern. Both calls use temperature=0 for deterministic responses.

### 3. Conversation History Injection

History is added to the system prompt, not as messages array:
```python
system_content = f"{SYSTEM_PROMPT}\n\nPrevious conversation:\n{conversation_history}"
```

This is because the messages array always starts fresh with just the current user query and tool exchanges.

### 4. Source Tracking

Sources come from tool execution, not direct vector search:
- `CourseSearchTool` stores sources in `last_sources` during formatting
- `ToolManager.get_last_sources()` retrieves them
- Sources are reset after each query to avoid leakage

## Data Models

[backend/models.py](backend/models.py) defines the core structures:
- **Course**: title (unique ID), course_link, instructor, lessons[]
- **Lesson**: lesson_number, title, lesson_link
- **CourseChunk**: content, course_title, lesson_number, chunk_index

**Course titles are unique identifiers** - they serve as both the display name and the primary key for filtering.

## Adding New Courses

Place course files in [docs/](docs/) with expected format:
```
Course Title: <title>
Instructor: <name>
Course Link: <url>

Lesson 1: <title>
<content>

Lesson 2: <title>
<content>
```

On next startup, the system will:
1. Detect new courses (by comparing titles to existing ChromaDB entries)
2. Process and chunk the content
3. Add to both `course_catalog` and `course_content` collections

Supported formats: `.txt`, `.pdf`, `.docx`

## Frontend Architecture

Simple vanilla JavaScript ([frontend/script.js](frontend/script.js)):
- No framework dependencies
- Uses `marked.js` for markdown rendering
- Maintains `currentSessionId` for conversation continuity
- Sources displayed in collapsible `<details>` element

Static files served via FastAPI's `StaticFiles` with no-cache headers for development.

## Troubleshooting

**ChromaDB persistence**: Data stored in `backend/chroma_db/` directory
- To reset: Delete the directory and restart
- Collections auto-created on first run

**Document not loading**: Check that:
- File is in `docs/` directory (relative to root)
- File has `.txt`, `.pdf`, or `.docx` extension
- Course title in file is unique (duplicates are skipped)

**Search returns no results**: Vector search requires semantic similarity
- Exact string matching not required
- Course name resolution uses fuzzy matching
- Check ChromaDB has content: `rag_system.vector_store.get_course_count()`
