# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) chatbot** for course materials. It uses ChromaDB for vector storage, Anthropic's Claude API with tool calling, and provides a web interface for querying course content.

**Key architectural pattern**: This is an **agentic RAG system**. Claude autonomously decides whether to search the knowledge base based on the query type, rather than always retrieving documents. The AI uses tool calling to invoke searches only when needed.

## Running the Application

**Quick start:**
```bash
./run.sh
```

**Manual start:**
```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

**First-time setup:**
1. Install uv: `curl -LsSf https://astral.sh/uv/install.sh | sh`
2. Install dependencies: `uv sync`
3. Create `.env` file with: `ANTHROPIC_API_KEY=your_key_here`

The app runs at http://localhost:8000 and automatically loads course documents from `docs/` on startup.

## Architecture

### Request Flow (Query Processing)

```
Frontend → FastAPI → RAG System → AI Generator → Claude API
                          ↓              ↓
                   SessionManager   ToolManager
                                        ↓
                                  SearchTool → VectorStore (ChromaDB)
```

**Two-phase AI generation:**
1. **Initial call**: Claude receives query with tool definitions, decides whether to search
2. **Tool execution** (if needed): Search tool queries ChromaDB, returns formatted results
3. **Final call**: Claude receives search results and generates answer

See `query-flow-diagram.md` for detailed flow visualization.

### Core Components

**RAGSystem** (`rag_system.py`) - Main orchestrator
- Coordinates all components
- Manages document ingestion from `docs/` folder
- Handles query → answer flow
- Key method: `query(query, session_id)` returns `(answer, sources)`

**VectorStore** (`vector_store.py`) - Dual-collection ChromaDB setup
- `course_catalog` collection: Course metadata (titles, instructors, lessons)
- `course_content` collection: Chunked course content
- Semantic course name resolution (partial matches work)
- Unified search interface: `search(query, course_name=None, lesson_number=None)`

**DocumentProcessor** (`document_processor.py`) - Course document parser
- Expected format: 3-line header (title, link, instructor) + lesson sections
- Lessons marked as: `Lesson N: Title`
- Sentence-based chunking with overlap (800 chars, 100 char overlap)
- Adds contextual prefixes to chunks: "Course X Lesson N content: ..."

**AIGenerator** (`ai_generator.py`) - Claude API wrapper
- Temperature: 0 (deterministic)
- Max tokens: 800
- Handles tool calling flow: initial request → tool execution → final response
- System prompt emphasizes: direct answers, one search max, no meta-commentary

**SearchTool** (`search_tools.py`) - Tool implementation
- Tool name: `search_course_content`
- Parameters: `query` (required), `course_name`, `lesson_number` (optional)
- Formats results with source metadata for UI display
- Tracks sources in `last_sources` for retrieval

**SessionManager** (`session_manager.py`) - Conversation context
- Maintains conversation history (default: last 2 exchanges = 4 messages)
- Auto-prunes old messages to stay within limits
- History format: "User: ...\nAssistant: ..."

### Data Models (`models.py`)

- **Course**: `title` (unique ID), `course_link`, `instructor`, `lessons[]`
- **Lesson**: `lesson_number`, `title`, `lesson_link`
- **CourseChunk**: `content`, `course_title`, `lesson_number`, `chunk_index`

### Configuration (`config.py`)

Key settings in `Config` dataclass:
- Model: `claude-sonnet-4-20250514`
- Embeddings: `all-MiniLM-L6-v2`
- Chunk size: 800 chars, overlap: 100 chars
- Max results: 5, conversation history: 2 exchanges
- ChromaDB path: `./chroma_db`

## Course Document Format

Place `.txt`, `.pdf`, or `.docx` files in `docs/`. Expected format:

```
Course Title: Introduction to Python
Course Link: https://example.com/course
Course Instructor: Jane Doe

Lesson 0: Getting Started
Lesson Link: https://example.com/lesson0
Content for lesson 0...

Lesson 1: Variables
Content for lesson 1...
```

Documents are automatically processed on startup. Duplicates are skipped based on course title.

## Key Implementation Details

**Semantic Course Name Matching**: When a search specifies `course_name`, the system first searches the `course_catalog` collection to find the best matching course title, then filters content search by exact match.

**Tool Calling Pattern**: The system uses Anthropic's tool calling API. When `stop_reason == "tool_use"`, it executes the tool, appends results to message history, and makes a second API call without tools to get the final answer.

**Chunk Context Enhancement**: Each chunk is prefixed with course and lesson info to improve retrieval accuracy. First chunk of each lesson gets context; subsequent chunks in same lesson get minimal context.

**ChromaDB Collections**: Using two separate collections allows efficient course metadata search (for name resolution) separate from content search.

## Frontend

Vanilla JavaScript (`frontend/`) with:
- `script.js`: Handles query submission to `/api/query`, displays markdown responses
- `index.html`: Chat interface
- Uses `marked.js` for markdown rendering

## API Endpoints

- `POST /api/query`: Process user query
  - Request: `{query: str, session_id?: str}`
  - Response: `{answer: str, sources: str[], session_id: str}`

- `GET /api/courses`: Get course statistics
  - Response: `{total_courses: int, course_titles: str[]}`

- `GET /docs`: API documentation (auto-generated by FastAPI)
- always use uv to run the server do not use pip directly
- make sure to use uv to manage all dependencies
- use uv to run Python files