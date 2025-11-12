# Architecture Quick Reference

## System Components

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer | Component          | Technology        | Responsibility  │
├─────────────────────────────────────────────────────────────────┤
│ 7     │ Frontend          │ HTML/CSS/JS       │ UI, polling     │
│ 6     │ API              │ FastAPI           │ HTTP routing    │
│ 5     │ Service          │ Python threading  │ Task management │
│ 4     │ Orchestration    │ aiSuite + LLM     │ Plan & execute  │
│ 3     │ Agents           │ Python functions  │ Research/write  │
│ 2     │ Tools            │ HTTP clients      │ API integration │
│ 1     │ Data             │ SQLAlchemy + PG   │ Persistence     │
└─────────────────────────────────────────────────────────────────┘
```

## Request-Response Cycle

```
User Input:
"Research machine learning fundamentals"
         │
         ↓
   POST /generate_report
         │
         ├─→ Validate (Pydantic)
         ├─→ Create task in DB
         ├─→ Start planner_agent
         ├─→ Get 7-step plan
         ├─→ Spawn worker thread
         └─→ Return task_id immediately
         
Worker Thread executes in background:
         │
         ├─ Step 1: Research agent (Tavily search)
         ├─ Step 2: Research agent (arXiv search)
         ├─ Step 3: Research agent (Wikipedia)
         ├─ Step 4: Writer agent (draft)
         ├─ Step 5: Editor agent (review)
         ├─ Step 6: Writer agent (final)
         └─ Step 7: Complete
         
         ├─→ Update DB with results
         └─→ Thread terminates

Frontend polling:
GET /task_progress/{task_id}  (every 2 seconds)
         │
         └─→ Display step status + substeps
         
Final retrieval:
GET /task_status/{task_id}    (when done)
         │
         └─→ Display report + download options
```

## State Management

```
┌─────────────────────────────────────────────────────────────────┐
│ Storage | Scope          | Persistence | Speed  | Use Case      │
├─────────────────────────────────────────────────────────────────┤
│ Memory  │ Runtime only   │ Lost        │ Fast   │ Real-time UI  │
│ Database│ Permanent      │ Persisted   │ Slower │ Long-term     │
└─────────────────────────────────────────────────────────────────┘

Task object in DB:
{
  "id": "uuid-123",
  "prompt": "Research LLMs",
  "status": "running|done|error",
  "result": {
    "html_report": "# Title\n\n...",
    "history": [...]
  },
  "created_at": "2025-11-11T10:00:00",
  "updated_at": "2025-11-11T10:05:00"
}

In-memory progress:
task_progress = {
  "uuid-123": {
    "steps": [
      {
        "title": "Research agent: Use Tavily...",
        "status": "done",
        "description": "Completed: ...",
        "substeps": [
          {
            "title": "Called research_agent",
            "content": "<div>...results...</div>"
          }
        ],
        "updated_at": "2025-11-11T10:02:30Z"
      }
    ]
  }
}
```

## Data Flow Sequence

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. User → Prompt → API /generate_report                      │
│                    ↓                                           │
│  2. API → Task created in DB (status: running)                │
│       ↓                                                        │
│  3. API → task_progress[id] initialized                       │
│       ↓                                                        │
│  4. API → planner_agent() called                              │
│       ├─→ LLM generates 7-step plan                           │
│       ├─→ Validation & contract enforcement                   │
│       └─→ Returns List[str] of steps                          │
│       ↓                                                        │
│  5. API → run_agent_workflow() spawned (thread)               │
│       ├─→ Returns task_id to user immediately                 │
│       └─→ Continues in background                             │
│       ↓                                                        │
│  6. Frontend → Polls /task_progress/{id} (2s intervals)       │
│       │                                                       │
│       ├─→ Reads from task_progress dict                       │
│       ├─→ Displays step status                                │
│       └─→ Repeats until status = "done"                       │
│       ↓                                                        │
│  7. Worker thread → For each step:                            │
│       ├─→ Update step.status = "running"                      │
│       ├─→ Call agent (research/writer/editor)                │
│       ├─→ Agent uses tools (Tavily/arXiv/Wiki)               │
│       ├─→ Get results                                         │
│       ├─→ Add substep                                         │
│       └─→ Update step.status = "done"                         │
│       ↓                                                        │
│  8. Worker thread → Workflow complete:                        │
│       ├─→ Extract final report                                │
│       ├─→ Update DB (status: done, result: JSON)              │
│       └─→ Terminate thread                                    │
│       ↓                                                        │
│  9. Frontend → Gets /task_status/{id}:                        │
│       ├─→ Reads from DB                                       │
│       ├─→ Renders final report                                │
│       └─→ Shows download options                              │
│       ↓                                                        │
│ 10. User → Reads report / Downloads / Starts new search       │
│                                                                │
└─────────────────────────────────────────────────────────────────┘
```

## Concurrency Model

```
Main FastAPI Thread:
  ├─ Accept request 1 (user submits prompt)
  ├─ Create task 1
  ├─ SPAWN worker thread 1
  ├─ Accept request 2 (user submits another prompt)
  ├─ Create task 2
  ├─ SPAWN worker thread 2
  ├─ Accept request 3 (frontend polls /task_progress/1)
  ├─ Return current progress
  └─ ... continue accepting requests

Worker Thread 1 (Task 1):
  ├─ Execute research agent
  ├─ Execute writer agent
  ├─ Execute editor agent
  ├─ Update DB
  └─ Terminate

Worker Thread 2 (Task 2):
  ├─ Execute research agent
  ├─ Execute writer agent
  ├─ Execute editor agent
  ├─ Update DB
  └─ Terminate

Result: Multiple tasks can run concurrently
```

## Error Handling

```
Tool Layer:
  Error in API call
    │
    └─→ Return {"error": "message"} (graceful)

Agent Layer:
  Error in agent execution
    │
    └─→ Return error message (LLM-friendly format)

Orchestration Layer:
  Error in step execution
    │
    └─→ Return error to service layer

Service Layer:
  Error in workflow
    │
    ├─→ Mark failed step as "error"
    ├─→ Update DB: status = "error"
    ├─→ Store error message
    └─→ Terminate thread gracefully

API Layer:
  Error in request handling
    │
    └─→ HTTPException with status code

Frontend:
  Status = "error"
    │
    └─→ Display error to user
```

## Docker Architecture

```
┌──────────────────────────────────┐
│  Docker Image: agentic-ai        │
├──────────────────────────────────┤
│ FROM python:3.11-slim            │
│ RUN apt-get install postgres     │
│ COPY requirements.txt            │
│ RUN pip install -r ...           │
│ COPY . /app                      │
│ EXPOSE 8000 5432                 │
│ CMD ["/entrypoint.sh"]           │
└──────────────────────────────────┘
           │
           ↓ docker run
┌──────────────────────────────────────────────────┐
│  Docker Container                                │
├──────────────────────────────────────────────────┤
│ ┌──────────────────┐      ┌──────────────────┐  │
│ │ PostgreSQL       │      │ FastAPI          │  │
│ │ Port 5432        │      │ Port 8000        │  │
│ │ appdb (user:app) │      │ Uvicorn server   │  │
│ └──────────────────┘      └──────────────────┘  │
│         ↑                           ↑            │
│         └───────┬───────────────────┘            │
│                 │ SQLAlchemy ORM                 │
│              DATABASE_URL:                       │
│        postgresql://app:local@                   │
│        127.0.0.1:5432/appdb                      │
└──────────────────────────────────────────────────┘
           ↑                ↑
           │                └─→ Port 8000:8000
           └─→ Port 5432:5432
         (Host machine)
```

## Key Design Patterns

### 1. Layered Architecture
- Separation of concerns
- Each layer has specific responsibility
- Communication through well-defined interfaces

### 2. Agent Pattern
- Autonomous agents (research, writer, editor)
- Tool-using capability (Tavily, arXiv, Wikipedia)
- Planning-based execution

### 3. Thread-Per-Task
- Non-blocking API
- Async-like behavior with threading
- Scalable to moderate concurrency

### 4. Dual State
- In-memory for speed
- Database for persistence
- Frontend reads from both as needed

### 5. Tool Calling
- LLM decides which tools to use
- Agents are tool-aware
- Flexible research strategies

## Performance Characteristics

```
Operation          | Latency  | Bottleneck
───────────────────┼──────────┼─────────────────────────
API request        | < 100ms  | Network
Task creation      | < 50ms   | Database
Plan generation    | 2-5s     | LLM (GPT-4o mini)
Agent execution    | 10-60s   | LLM + tool calls
Workflow total     | 30-300s  | Agent execution
Report generation  | 5-15s    | LLM (writer agent)
DB persistence     | < 500ms  | Database write
Frontend polling   | < 50ms   | Network + cache
```

## Scaling Considerations

```
Current:
- Single instance
- In-memory state
- Sequential task execution
- ✓ Works for: Testing, small teams, demos

Scaling to production:
├─ Message Queue (Celery/RabbitMQ)
│  └─ Distribute tasks across workers
├─ Redis Cache
│  └─ Distributed in-memory state
├─ Async/Await
│  └─ Replace threading with asyncio
├─ WebSocket
│  └─ Real-time updates instead of polling
├─ Microservices
│  └─ Separate concerns into services
└─ Load Balancing
   └─ Distribute traffic across instances
```

## Tech Stack Summary

```
Frontend:
├─ HTML5, CSS3
├─ JavaScript (ES6+)
├─ Bootstrap 5.3 (UI framework)
└─ Marked.js (Markdown parser)

Backend:
├─ Python 3.11
├─ FastAPI (web framework)
├─ Pydantic (validation)
├─ SQLAlchemy (ORM)
├─ aiSuite (LLM abstraction)
├─ Tavily SDK (web search)
├─ arXiv API (academic search)
└─ Wikipedia API (knowledge base)

Infrastructure:
├─ Docker (containerization)
├─ PostgreSQL (database)
├─ Uvicorn (ASGI server)
└─ Python threading (concurrency)

External APIs:
├─ OpenAI API (GPT-4o mini)
├─ Tavily API (web search)
├─ arXiv API (papers)
└─ Wikipedia API (general knowledge)
```

