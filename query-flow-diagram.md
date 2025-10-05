# RAG System Query Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (script.js)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ 1. User types query
                                      ▼
                            ┌──────────────────┐
                            │  sendMessage()   │
                            │  - Disable UI    │
                            │  - Show loading  │
                            └──────────────────┘
                                      │
                                      │ POST /api/query
                                      │ {query, session_id}
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BACKEND API (app.py)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ 2. FastAPI endpoint
                                      ▼
                         ┌──────────────────────────┐
                         │  query_documents()       │
                         │  - Create/get session    │
                         │  - Call RAG system       │
                         └──────────────────────────┘
                                      │
                                      │ rag_system.query(query, session_id)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RAG ORCHESTRATOR (rag_system.py)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
          ┌─────────────────┐  ┌──────────┐  ┌─────────────────┐
          │ SessionManager  │  │  Prompt  │  │  ToolManager    │
          │ - Get history   │  │  Build   │  │  - Get tool     │
          └─────────────────┘  └──────────┘  │    definitions  │
                    │                 │       └─────────────────┘
                    │                 │                 │
                    └────────┬────────┴────────┬────────┘
                             │                 │
                             ▼                 ▼
                    ┌──────────────────────────────────┐
                    │  ai_generator.generate_response  │
                    │  - History + Query + Tools       │
                    └──────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AI GENERATOR (ai_generator.py)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ 3. First Claude API call
                                      ▼
                         ┌──────────────────────────┐
                         │   Anthropic Claude API   │
                         │  - System prompt         │
                         │  - Conversation history  │
                         │  - Tool definitions      │
                         └──────────────────────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                        ▼                           ▼
            ┌─────────────────────┐    ┌──────────────────────┐
            │  Direct Answer      │    │  stop_reason:        │
            │  (General question) │    │  "tool_use"          │
            └─────────────────────┘    │  (Course question)   │
                        │               └──────────────────────┘
                        │                           │
                        │                           │ 4. Tool execution
                        │                           ▼
                        │              ┌──────────────────────────┐
                        │              │  tool_manager.execute()  │
                        │              └──────────────────────────┘
                        │                           │
                        │                           ▼
                        │              ┌─────────────────────────────────────┐
                        │              │  SEARCH TOOL (search_tools.py)      │
                        │              │  - CourseSearchTool.execute()       │
                        │              │  - Parse: query, course, lesson     │
                        │              └─────────────────────────────────────┘
                        │                           │
                        │                           ▼
                        │              ┌─────────────────────────────────────┐
                        │              │  VECTOR STORE (vector_store.py)     │
                        │              │  - ChromaDB semantic search         │
                        │              │  - Filter by course/lesson          │
                        │              │  - Return top results + metadata    │
                        │              └─────────────────────────────────────┘
                        │                           │
                        │                           │ 5. Format results
                        │                           ▼
                        │              ┌──────────────────────────┐
                        │              │  Tool Result:            │
                        │              │  "[Course - Lesson N]    │
                        │              │   Content chunk..."      │
                        │              │                          │
                        │              │  + Store sources         │
                        │              └──────────────────────────┘
                        │                           │
                        │                           │ 6. Second Claude call
                        │                           ▼
                        │              ┌──────────────────────────┐
                        │              │   Anthropic Claude API   │
                        │              │  - Original messages     │
                        │              │  - Tool use request      │
                        │              │  - Tool results          │
                        │              └──────────────────────────┘
                        │                           │
                        │                           ▼
                        │              ┌──────────────────────────┐
                        │              │  Final Answer            │
                        │              │  (Based on search)       │
                        │              └──────────────────────────┘
                        │                           │
                        └───────────────┬───────────┘
                                        │
                                        ▼
                           ┌────────────────────────┐
                           │  Return to RAG System  │
                           │  - answer (string)     │
                           └────────────────────────┘
                                        │
                                        │ 7. Get sources & update history
                                        ▼
                           ┌────────────────────────────────┐
                           │  - tool_manager.get_sources()  │
                           │  - session_manager.add_exchange│
                           └────────────────────────────────┘
                                        │
                                        │ Return (answer, sources)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BACKEND API (app.py)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 8. JSON Response
                                        │ {answer, sources[], session_id}
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (script.js)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ 9. Display results
                                        ▼
                           ┌────────────────────────┐
                           │  - Remove loading      │
                           │  - Show answer         │
                           │  - Show sources        │
                           │  - Re-enable UI        │
                           └────────────────────────┘
```

## Key Decision Points

**Claude's Decision Logic:**
- **General question** (e.g., "What is Python?") → Direct answer, no search
- **Course-specific** (e.g., "What is MCP in lesson 2?") → Use search tool

**Search Tool Flow:**
```
search_course_content(query, course_name?, lesson_number?)
         ↓
Vector Store Search (ChromaDB)
         ↓
Results filtered & formatted with metadata
         ↓
Returned to Claude with source tracking
```

## Components Summary

| Component | Responsibility | Key Method |
|-----------|---------------|------------|
| Frontend | UI & API calls | `sendMessage()` |
| FastAPI | Route handling | `query_documents()` |
| RAG System | Orchestration | `query()` |
| AI Generator | Claude interaction | `generate_response()` |
| Search Tool | Vector search | `execute()` |
| Vector Store | ChromaDB queries | `search()` |
| Session Manager | Conversation history | `get_conversation_history()` |
