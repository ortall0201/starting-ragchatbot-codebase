# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend<br/>(HTML/JS)
    participant API as FastAPI<br/>(/api/query)
    participant RAG as RAG System<br/>(rag_system.py)
    participant AI as AI Generator<br/>(Claude API)
    participant Tools as Search Tools<br/>(search_tools.py)
    participant Vector as Vector Store<br/>(ChromaDB)
    participant Session as Session Manager

    %% User initiates query
    User->>Frontend: Types query & clicks send
    Frontend->>Frontend: Disable UI, show loading
    Frontend->>Frontend: Display user message
    
    %% API call to backend
    Frontend->>+API: POST /api/query<br/>{query, session_id}
    API->>API: Validate request
    
    %% RAG system processing
    API->>+RAG: query(query, session_id)
    RAG->>Session: get_conversation_history(session_id)
    Session-->>RAG: Previous messages
    
    %% AI generation with tools
    RAG->>+AI: generate_response()<br/>+ tools + history
    AI->>AI: Build system prompt<br/>+ conversation context
    AI->>+AI: Claude API call<br/>with tools available
    
    %% Tool execution (if needed)
    alt Claude decides to search
        AI->>+Tools: execute search_course_content<br/>(query, course_name, lesson_number)
        Tools->>+Vector: search(query, filters)
        Vector->>Vector: 1. Resolve course name
        Vector->>Vector: 2. Build metadata filters
        Vector->>Vector: 3. Semantic search in ChromaDB
        Vector-->>-Tools: SearchResults<br/>(documents, metadata, distances)
        Tools->>Tools: Format results with<br/>course/lesson context
        Tools->>Tools: Track sources for UI
        Tools-->>-AI: Formatted search results
        AI->>AI: Process tool results
        AI->>+AI: Second Claude API call<br/>for final response
        AI-->>-AI: Final response text
    else Claude uses general knowledge
        AI-->>AI: Direct response from knowledge
    end
    
    AI-->>-RAG: Generated response
    RAG->>Tools: get_last_sources()
    Tools-->>RAG: Source list
    RAG->>Session: add_exchange(session_id, query, response)
    RAG-->>-API: (response, sources)
    
    %% API response
    API->>API: Build QueryResponse<br/>{answer, sources, session_id}
    API-->>-Frontend: JSON response
    
    %% Frontend display
    Frontend->>Frontend: Remove loading indicator
    Frontend->>Frontend: Display AI response<br/>(with markdown support)
    Frontend->>Frontend: Show sources section<br/>(if sources exist)
    Frontend->>Frontend: Re-enable UI controls
    Frontend-->>User: Complete response displayed

    %% Error handling paths
    Note over API,Vector: Error handling at each stage:<br/>- Network failures<br/>- API errors<br/>- Search failures<br/>- Tool execution errors
```

## Architecture Overview

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[Web Interface<br/>HTML/CSS/JS]
        Chat[Chat Interface]
        Stats[Course Statistics]
    end
    
    subgraph "API Layer" 
        FastAPI[FastAPI Server<br/>app.py]
        Endpoints["/api/query<br/>/api/courses"]
    end
    
    subgraph "RAG System Core"
        RAG[RAG Orchestrator<br/>rag_system.py]
        Session[Session Manager<br/>conversation history]
    end
    
    subgraph "AI & Tools"
        AI[AI Generator<br/>Claude API]
        ToolMgr[Tool Manager]
        SearchTool[Course Search Tool]
    end
    
    subgraph "Data Layer"
        Vector[Vector Store<br/>ChromaDB]
        DocProcessor[Document Processor]
        Models[Data Models<br/>Course/Lesson/Chunk]
    end
    
    subgraph "Course Data"
        Docs[Course Documents<br/>course1_script.txt<br/>course2_script.txt<br/>etc.]
    end
    
    %% Connections
    UI --> FastAPI
    FastAPI --> RAG
    RAG --> AI
    RAG --> Session
    AI --> ToolMgr
    ToolMgr --> SearchTool
    SearchTool --> Vector
    DocProcessor --> Vector
    DocProcessor --> Models
    Docs --> DocProcessor
    
    %% Styling
    classDef frontend fill:#e1f5fe
    classDef api fill:#f3e5f5  
    classDef core fill:#e8f5e8
    classDef ai fill:#fff3e0
    classDef data fill:#fce4ec
    
    class UI,Chat,Stats frontend
    class FastAPI,Endpoints api
    class RAG,Session core
    class AI,ToolMgr,SearchTool ai
    class Vector,DocProcessor,Models,Docs data
```

## Data Flow Summary

1. **User Query** → Frontend captures and sends to API
2. **API Routing** → FastAPI validates and routes to RAG system  
3. **RAG Orchestration** → Manages session context and coordinates components
4. **AI Processing** → Claude analyzes query and decides on tool usage
5. **Tool Execution** → Searches course content using vector similarity
6. **Vector Search** → ChromaDB finds relevant course chunks with metadata
7. **Response Generation** → AI synthesizes search results into answer
8. **Session Update** → Conversation history maintained for context
9. **API Response** → Structured response with answer, sources, session ID
10. **Frontend Display** → Renders response with markdown and source citations

## Key Features

- **Semantic Search**: Uses sentence transformers for meaningful content matching
- **Course Resolution**: Handles partial course name matches  
- **Session Continuity**: Maintains conversation context across queries
- **Tool Intelligence**: AI decides when to search vs. use general knowledge
- **Source Attribution**: Tracks and displays specific course/lesson sources
- **Error Resilience**: Graceful handling of failures at each layer