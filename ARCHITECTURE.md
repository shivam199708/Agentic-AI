# Agentic AI Research Agent - Deep Architecture Guide

## Table of Contents
1. [System Architecture Overview](#system-architecture-overview)
2. [Layered Architecture](#layered-architecture)
3. [Data Flow Deep Dive](#data-flow-deep-dive)
4. [Component Interactions](#component-interactions)
5. [Concurrency & Threading](#concurrency--threading)
6. [State Management](#state-management)
7. [Error Handling Strategy](#error-handling-strategy)
8. [Deployment Architecture](#deployment-architecture)

---

## System Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                    │
│                                                                           │
│  ┌──────────────────────┐      ┌──────────────────────┐                 │
│  │   Web Browser UI     │      │   API Clients        │                 │
│  │  (index.html)        │      │  (curl, SDK, etc)    │                 │
│  └──────────┬───────────┘      └──────────┬───────────┘                 │
│             │                             │                              │
│             └─────────────┬───────────────┘                              │
│                           │                                              │
└───────────────────────────┼──────────────────────────────────────────────┘
                            │ HTTP/WebSocket
                            ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                      API LAYER (FastAPI)                                 │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Routes:                                                           │  │
│  │  - GET  /                    → Serve HTML UI                       │  │
│  │  - GET  /api                 → Health check                        │  │
│  │  - POST /generate_report     → Create new task                    │  │
│  │  - GET  /task_progress/{id}  → Real-time status                   │  │
│  │  - GET  /task_status/{id}    → Final results                      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                  ↓                                        │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │         Request Handlers & Business Logic                         │  │
│  │  - Input validation (Pydantic models)                             │  │
│  │  - CORS middleware                                                │  │
│  │  - Static file serving                                            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER (main.py)                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Task Management:                                                  │  │
│  │  - generate_report()     → Create & initialize task               │  │
│  │  - run_agent_workflow()  → Execute multi-step research            │  │
│  │  - update_step_status()  → Track progress                         │  │
│  │  - format_history()      → Prepare context for agents             │  │
│  │                                                                    │  │
│  │  State Management:                                                │  │
│  │  - task_progress = {}    → In-memory task tracking                │  │
│  │  - Database (SQLAlchemy) → Persistent storage                     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└────────────┬─────────────────────────────────────────────────────────────┘
             │
             └─────────────────────────────────────────────────┬───────────┐
                                                               │           │
┌─────────────────────────────────────────────────────┐       │           │
│        AGENT ORCHESTRATION LAYER                    │       │           │
│        (planning_agent.py)                          │       │           │
│  ┌─────────────────────────────────────────────┐    │       │           │
│  │  planner_agent(topic)                       │    │       │           │
│  │  ├─ Sends prompt to LLM (GPT-4o mini)      │    │       │           │
│  │  ├─ Gets back: List[str] of steps          │    │       │           │
│  │  ├─ Validates and enforces contract        │    │       │           │
│  │  └─ Returns structured plan                 │    │       │           │
│  │                                             │    │       │           │
│  │  executor_agent_step(step, history)        │    │       │           │
│  │  ├─ Identifies step type (research/etc)   │    │       │           │
│  │  ├─ Calls appropriate agent                │    │       │           │
│  │  └─ Returns: title, agent_name, output    │    │       │           │
│  └─────────────────────────────────────────────┘    │       │           │
└──────────────────┬─────────────────────────────────┘       │           │
                   │                                         │           │
                   ↓                                         │           │
┌─────────────────────────────────────────────────────┐      │           │
│      AGENT IMPLEMENTATION LAYER                     │      │           │
│      (agents.py)                                    │      │           │
│  ┌─────────────────────────────────────────────┐    │      │           │
│  │  research_agent(prompt)                     │    │      │           │
│  │  ├─ Tool: tavily_search_tool                │    │      │           │
│  │  ├─ Tool: arxiv_search_tool                 │    │      │           │
│  │  ├─ Tool: wikipedia_search_tool             │    │      │           │
│  │  └─ Returns: formatted research findings    │    │      │           │
│  │                                             │    │      │           │
│  │  writer_agent(prompt)                       │    │      │           │
│  │  ├─ Synthesizes research into report        │    │      │           │
│  │  ├─ Includes citations [1], [2]...         │    │      │           │
│  │  └─ Returns: Markdown formatted report      │    │      │           │
│  │                                             │    │      │           │
│  │  editor_agent(prompt)                       │    │      │           │
│  │  ├─ Reviews and improves report             │    │      │           │
│  │  ├─ Preserves all citations                 │    │      │           │
│  │  └─ Returns: enhanced report                │    │      │           │
│  └─────────────────────────────────────────────┘    │      │           │
└──────────────────┬─────────────────────────────────┘      │           │
                   │                                        │           │
                   ↓                                        │           │
┌─────────────────────────────────────────────────────┐     │           │
│      TOOL INTEGRATION LAYER                         │     │           │
│      (research_tools.py)                            │     │           │
│  ┌─────────────────────────────────────────────┐    │     │           │
│  │  tavily_search_tool(query)                  │    │     │           │
│  │  ├─ Makes HTTP calls to Tavily API         │    │     │           │
│  │  ├─ Returns: web search results             │    │     │           │
│  │  └─ Format: [{"title": "...", "url": ...}] │    │     │           │
│  │                                             │    │     │           │
│  │  arxiv_search_tool(query)                   │    │     │           │
│  │  ├─ Queries arXiv API                       │    │     │           │
│  │  ├─ Fetches & extracts PDF text             │    │     │           │
│  │  └─ Format: [{"title": "...", "pdf": ...}] │    │     │           │
│  │                                             │    │     │           │
│  │  wikipedia_search_tool(query)               │    │     │           │
│  │  ├─ Searches Wikipedia API                  │    │     │           │
│  │  └─ Format: [{"summary": "...", ...}]       │    │     │           │
│  └─────────────────────────────────────────────┘    │     │           │
└──────────────────┬─────────────────────────────────┘     │           │
                   │                                       │           │
                   ↓                                       │           │
┌─────────────────────────────────────────────────────┐    │           │
│      EXTERNAL SERVICES                              │    │           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │    │           │
│  │  Tavily API  │  │  arXiv API   │  │ Wikipedia│  │    │           │
│  │   (REST)     │  │   (REST)     │  │ (Python) │  │    │           │
│  └──────────────┘  └──────────────┘  └──────────┘  │    │           │
│                                                     │    │           │
│  ┌─────────────────────────────────────────────┐   │    │           │
│  │  LLM Services (via aiSuite)                 │   │    │           │
│  │  - OpenAI API (GPT-4o mini, o4-mini)        │   │    │           │
│  │  - Claude (supported by aiSuite)            │   │    │           │
│  │  - Gemini (supported by aiSuite)            │   │    │           │
│  └─────────────────────────────────────────────┘   │    │           │
└─────────────────────────────────────────────────────┘    │           │
                                                           │           │
└───────────────────────────────────────────────────────────┤           │
                                                            ↓
┌──────────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  SQLAlchemy ORM:                                               │  │
│  │  - Task model (SQLite/PostgreSQL)                              │  │
│  │  - Persists: id, prompt, status, result, timestamps            │  │
│  │                                                                │  │
│  │  In-Memory State:                                              │  │
│  │  - task_progress = {task_id: {steps: [...]}}                  │  │
│  │  - Real-time tracking during workflow                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Layered Architecture

### Layer 1: Presentation Layer (Frontend)
- **Location**: `templates/index.html`
- **Technology**: HTML, CSS, JavaScript
- **Responsibilities**:
  - Render research interface
  - Collect user input
  - Poll API for real-time updates
  - Display step-by-step progress
  - Render final Markdown report

**Key Features**:
```javascript
// Polling mechanism (every 2 seconds)
setInterval(fetchProgress, 2000)

// Step tracking with expandable substeps
step.substeps.map(sub => render_substep(sub))

// Report export
downloadMarkdown() / downloadHTML()
```

### Layer 2: API Layer (FastAPI)
- **Location**: `main.py` (lines 60-90)
- **Technology**: FastAPI + Pydantic
- **Responsibilities**:
  - Route requests to appropriate handlers
  - Validate input using Pydantic models
  - Manage middleware (CORS, static files)
  - Handle HTTP responses

**Endpoints**:
```python
GET  /                    # HTML page
GET  /api                 # Health check
POST /generate_report     # Start workflow
GET  /task_progress/id    # Real-time status
GET  /task_status/id      # Final results
```

### Layer 3: Service Layer (Business Logic)
- **Location**: `main.py` (lines 82-198)
- **Technology**: Python, threading
- **Responsibilities**:
  - Create and manage tasks
  - Coordinate agent workflow
  - Update progress tracking
  - Persist results to database

**Key Functions**:
```python
generate_report(req)          # Initiate workflow
run_agent_workflow(...)       # Execute agents
update_step_status(...)       # Track progress
format_history(...)           # Context preparation
```

### Layer 4: Agent Orchestration Layer
- **Location**: `src/planning_agent.py`
- **Technology**: Python + aiSuite
- **Responsibilities**:
  - Generate research plan using LLM
  - Coordinate agent execution
  - Maintain execution history
  - Enforce plan constraints

**Key Functions**:
```python
planner_agent(topic)              # Generate plan
executor_agent_step(step, history) # Execute single step
_ensure_contract(steps)            # Validate plan
```

### Layer 5: Agent Implementation Layer
- **Location**: `src/agents.py`
- **Technology**: Python + aiSuite + Tool calling
- **Responsibilities**:
  - Implement specific agent behaviors
  - Manage tool selection and calling
  - Format outputs appropriately

**Agents**:
```python
research_agent()   # Gathers information
writer_agent()     # Creates report
editor_agent()     # Reviews & improves
```

### Layer 6: Tool Integration Layer
- **Location**: `src/research_tools.py`
- **Technology**: HTTP clients, API SDKs
- **Responsibilities**:
  - Interface with external APIs
  - Handle rate limiting and retries
  - Extract and format data
  - Error handling

**Tools**:
```python
tavily_search_tool()     # Web search
arxiv_search_tool()      # Academic papers
wikipedia_search_tool()  # General knowledge
```

### Layer 7: Data Layer
- **Location**: `main.py` (lines 32-50)
- **Technology**: SQLAlchemy ORM + PostgreSQL
- **Responsibilities**:
  - Persist task data
  - Maintain data consistency
  - Enable historical queries

**Data Model**:
```python
class Task(Base):
    id: String (PK)
    prompt: Text
    status: String
    result: Text (JSON)
    created_at: DateTime
    updated_at: DateTime
```

---

## Data Flow Deep Dive

### Request Flow Sequence

```
1. USER SUBMITS PROMPT
   ├─ Browser: textarea.value
   └─ POST /generate_report { prompt: "..." }

2. API RECEIVES REQUEST
   ├─ FastAPI validates with PromptRequest model
   └─ Route handler: generate_report(req)

3. TASK INITIALIZATION
   ├─ Create UUID for task_id
   ├─ Save to DB: Task(id, prompt, status="running")
   ├─ Initialize in-memory: task_progress[task_id] = {}
   └─ Return: { task_id: "..." }

4. PLANNING PHASE
   ├─ Call: planner_agent(prompt)
   ├─ LLM receives detailed prompt
   ├─ LLM returns: ["Research agent: ...", "Writer agent: ...", ...]
   ├─ Parse and validate steps (enforce contract)
   ├─ Initialize progress tracking for each step
   └─ Return: List[str]

5. BACKGROUND WORKFLOW STARTS
   ├─ Thread: threading.Thread(target=run_agent_workflow)
   └─ Main thread returns immediately (non-blocking)

6. FRONTEND POLLING BEGINS
   ├─ Every 2 seconds: GET /task_progress/{task_id}
   ├─ Response: { steps: [...] }
   ├─ Update UI with current status
   └─ Continue until status = "done" or "error"

7. AGENT EXECUTION LOOP (in background thread)
   For each step in plan:
   │
   ├─ Update: step.status = "running"
   │
   ├─ Identify step type from step.title
   │
   ├─ Call appropriate agent:
   │  ├─ Research agent:
   │  │  ├─ Builds enriched prompt with history
   │  │  ├─ Sends to LLM
   │  │  ├─ LLM chooses tool(s) to call
   │  │  ├─ Execute tool calls
   │  │  ├─ Synthesize results
   │  │  └─ Return formatted output
   │  │
   │  ├─ Writer agent:
   │  │  ├─ Receives research findings
   │  │  ├─ LLM generates comprehensive report
   │  │  ├─ Includes citations [1], [2], ...
   │  │  ├─ Maintains References section
   │  │  └─ Return Markdown
   │  │
   │  └─ Editor agent:
   │     ├─ Reviews draft
   │     ├─ Improves clarity/structure
   │     ├─ Preserves citations
   │     └─ Return enhanced version
   │
   ├─ Add to execution_history[]
   │
   ├─ Create substep for display:
   │  ├─ title: "Called {agent_name}"
   │  ├─ content: formatted HTML output
   │  └─ status: "done"
   │
   ├─ Update: step.status = "done"
   │
   └─ Increment to next step

8. WORKFLOW COMPLETION
   ├─ Extract final report from execution_history
   ├─ Prepare result object:
   │  ├─ html_report: final markdown
   │  └─ history: all step details
   │
   ├─ Save to DB:
   │  ├─ task.status = "done"
   │  ├─ task.result = JSON.dumps(result)
   │  └─ task.updated_at = now()
   │
   └─ task_progress[task_id] remains for frontend

9. FRONTEND RETRIEVES FINAL REPORT
   ├─ GET /task_status/{task_id}
   ├─ DB query: Task.filter(Task.id == task_id)
   ├─ Parse task.result JSON
   ├─ Return: { status: "done", result: {...} }
   │
   ├─ Frontend renders:
   │  ├─ h4: "📄 Final Report"
   │  ├─ Download buttons
   │  └─ Parsed Markdown as HTML
   │
   └─ User can: read, download, or start new research
```

### State Transitions

```
Task Lifecycle:
    created
       ↓
    running ←→ (polling updates)
       ├─ step 1: pending → running → done
       ├─ step 2: pending → running → done
       ├─ step 3: pending → running → done
       └─ ...
       ↓
    done (or error)
       ↓
    completed (results available)

Step Lifecycle (for each step):
    pending
       ↓
    running → (substeps added)
       ↓
    done
    
    OR
    
    running → error (if exception)
```

---

## Component Interactions

### Interaction 1: API ↔ Service Layer

```python
# API receives request
@app.post("/generate_report")
def generate_report(req: PromptRequest):
    task_id = str(uuid.uuid4())
    
    # 1. Service layer creates task
    db = SessionLocal()
    db.add(Task(id=task_id, prompt=req.prompt, status="running"))
    db.commit()
    db.close()
    
    # 2. Service layer initializes progress tracking
    task_progress[task_id] = {"steps": []}
    
    # 3. Service layer gets plan
    initial_plan_steps = planner_agent(req.prompt)
    
    # 4. Initialize step tracking
    for step_title in initial_plan_steps:
        task_progress[task_id]["steps"].append({
            "title": step_title,
            "status": "pending",
            "substeps": []
        })
    
    # 5. Start background workflow
    thread = threading.Thread(
        target=run_agent_workflow,
        args=(task_id, req.prompt, initial_plan_steps)
    )
    thread.start()
    
    # 6. Return immediately
    return {"task_id": task_id}
```

### Interaction 2: Service Layer ↔ Agent Orchestration

```python
def run_agent_workflow(task_id, prompt, initial_plan_steps):
    steps_data = task_progress[task_id]["steps"]
    execution_history = []
    
    try:
        for i, plan_step_title in enumerate(initial_plan_steps):
            # Update UI
            update_step_status(i, "running", f"Executing: {plan_step_title}")
            
            # Call orchestration layer
            actual_step_description, agent_name, output = \
                executor_agent_step(plan_step_title, execution_history, prompt)
            
            # Store for next iteration
            execution_history.append([
                plan_step_title,
                actual_step_description,
                output
            ])
            
            # Update UI with results
            update_step_status(i, "done", "Completed: ...", {
                "title": f"Called {agent_name}",
                "content": format_output(output)
            })
```

### Interaction 3: Agent Orchestration ↔ Agent Implementation

```python
def executor_agent_step(step_title, history, prompt):
    # Build enriched context
    context = f"User Prompt:\n{prompt}\n\nHistory:\n..."
    enriched_task = f"{context}\n\nYour next task:\n{step_title}"
    
    # Route to appropriate agent based on step type
    if "research" in step_title.lower():
        content, _ = research_agent(prompt=enriched_task)
        return step_title, "research_agent", content
    
    elif "draft" in step_title.lower() or "write" in step_title.lower():
        content, _ = writer_agent(prompt=enriched_task)
        return step_title, "writer_agent", content
    
    elif "revise" in step_title.lower() or "edit" in step_title.lower():
        content, _ = editor_agent(prompt=enriched_task)
        return step_title, "editor_agent", content
```

### Interaction 4: Agent Implementation ↔ Tool Integration

```python
def research_agent(prompt, model="openai:gpt-4.1-mini"):
    # Build full prompt with tool descriptions
    full_prompt = f"""
    You have access to these tools:
    - tavily_search_tool: General web search
    - arxiv_search_tool: Academic papers
    - wikipedia_search_tool: General knowledge
    
    User request: {prompt}
    """
    
    messages = [{"role": "user", "content": full_prompt}]
    
    # Call LLM with tool definitions
    resp = client.chat.completions.create(
        model=model,
        messages=messages,
        tools=[arxiv_search_tool, tavily_search_tool, wikipedia_search_tool],
        tool_choice="auto",  # LLM decides which tools to call
        max_turns=5
    )
    
    # LLM automatically calls appropriate tools
    # Results are processed and returned
    return content, messages
```

### Interaction 5: Tool Integration ↔ External APIs

```python
# Tool makes actual API call
def tavily_search_tool(query, max_results=5):
    api_key = os.getenv("TAVILY_API_KEY")
    client = TavilyClient(api_key)
    
    # HTTP request to Tavily
    response = client.search(query=query, max_results=max_results)
    
    # Format response
    results = [{
        "title": r.get("title"),
        "content": r.get("content"),
        "url": r.get("url")
    } for r in response.get("results", [])]
    
    return results
```

---

## Concurrency & Threading

### Threading Model

```
Main Thread (FastAPI)
├─ Accept HTTP requests
├─ Parse and validate
├─ Call generate_report()
│  ├─ Create task in DB
│  ├─ Initialize progress tracking
│  ├─ SPAWN NEW THREAD → (see below)
│  └─ Return task_id immediately
│
└─ Continue accepting new requests

Worker Thread (per task)
├─ Execute run_agent_workflow()
├─ For each step:
│  ├─ Update task_progress[task_id] (shared state)
│  ├─ Call agent
│  ├─ Get results
│  └─ Update task_progress again
├─ When done, update DB
└─ Thread terminates

Frontend (Browser)
├─ Poll /task_progress/{task_id} every 2 seconds
├─ Read from task_progress dict (shared state)
└─ Display updates
```

### Shared State

**In-Memory (Race Condition Potential)**:
```python
task_progress = {
    "task-uuid-123": {
        "steps": [
            {"title": "...", "status": "running", "substeps": [...]},
            {"title": "...", "status": "pending", "substeps": []},
        ]
    },
    "task-uuid-456": {
        # another task
    }
}
```

**Persistent (Database)**:
```python
# SQLAlchemy handles concurrent access
db.query(Task).filter(Task.id == task_id).first()
db.commit()  # Thread-safe transactions
```

### Concurrency Considerations

```
RACE CONDITIONS (Potential):
❌ Multiple threads updating task_progress simultaneously
   → Could lose updates
   → Not critical since we poll DB eventually

SOLUTIONS:
✅ For production:
   - Use threading.Lock() for task_progress updates
   - Implement message queues (Celery, RabbitMQ)
   - Use pub/sub for real-time updates (WebSocket)

CURRENT APPROACH (Development):
✅ Works for single instance
✅ Frontend always has DB as source of truth
✅ task_progress is supplementary (real-time display only)
```

---

## State Management

### Dual State Storage

```
┌─────────────────────────────────────────────────────────────┐
│  IN-MEMORY STATE (task_progress dict)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Purpose: Real-time feedback to frontend                    │
│  Scope: Current runtime only                                │
│  Persistence: Lost on restart                               │
│  Access: Fast (no DB query)                                 │
│                                                              │
│  Structure:                                                  │
│  task_progress[task_id] = {                                 │
│    "steps": [                                               │
│      {                                                       │
│        "title": "Research agent: Use Tavily...",            │
│        "status": "done",                                    │
│        "description": "Completed: ...",                     │
│        "substeps": [                                        │
│          {                                                   │
│            "title": "Called research_agent",                │
│            "content": "<div>...HTML output...</div>"        │
│          }                                                   │
│        ],                                                    │
│        "updated_at": "2025-11-11T10:30:00Z"                 │
│      }                                                       │
│    ]                                                        │
│  }                                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  DATABASE STATE (SQLAlchemy + PostgreSQL)                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Purpose: Persistent storage & long-term history            │
│  Scope: Survives restarts                                   │
│  Persistence: Until explicitly deleted                      │
│  Access: Slower (DB query)                                  │
│                                                              │
│  Table: tasks                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ id (PK)  │ prompt │ status │ result │ created_at │... │  │
│  ├──────────┼────────┼────────┼────────┼────────────┤    │  │
│  │ uuid-123 │ "ML..."│ "done" │ {JSON} │ 2025-11... │    │  │
│  │ uuid-456 │ "AI..." │"error"│ null   │ 2025-11... │    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  result field contains:                                     │
│  {                                                          │
│    "html_report": "# Title\n\nContent...",                 │
│    "history": [                                             │
│      {"step": 1, "agent": "research", ...},                 │
│      {"step": 2, "agent": "writer", ...}                    │
│    ]                                                        │
│  }                                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### State Synchronization

```
Timeline of a Task:
│
├─ T0: User submits prompt
│  ├─ DB: Create Task(status="running")
│  ├─ Memory: task_progress[id] = {...}
│  └─ API: Return task_id
│
├─ T1: Frontend polls /task_progress
│  ├─ Memory: Return in-memory state
│  └─ Frontend: Display current step
│
├─ T2-T7: Agent workflow executes
│  ├─ Memory: Update every step completion
│  ├─ Frontend: Poll every 2s, display updates
│  └─ DB: No updates yet
│
├─ T8: Workflow completes
│  ├─ Memory: Final step marked "done"
│  ├─ DB: Update Task(status="done", result={...})
│  └─ Flush: memory ← DB (source of truth)
│
├─ T9: Frontend polls /task_progress
│  ├─ Memory: Still has step details
│  └─ Frontend: Display completion
│
├─ T10: Frontend fetches /task_status
│  ├─ DB: Query Task by id
│  └─ Return: {"status": "done", "result": {...}}
│
└─ T11: Frontend renders final report
   └─ Display report + download options
```

---

## Error Handling Strategy

### Error Propagation

```python
# Layer 1: Tool Integration Layer
def tavily_search_tool(query):
    try:
        response = client.search(query=query)
        return results
    except Exception as e:
        return [{"error": str(e)}]  # LLM-friendly error format

# Layer 2: Agent Implementation Layer  
def research_agent(prompt):
    try:
        tools = [arxiv_search_tool, tavily_search_tool, wikipedia_search_tool]
        resp = client.chat.completions.create(
            model=model,
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )
        return resp.choices[0].message.content
    except Exception as e:
        print(f"❌ Error: {e}")
        return f"[Model Error: {str(e)}]"

# Layer 3: Agent Orchestration Layer
def executor_agent_step(step_title, history, prompt):
    try:
        if "research" in step_title.lower():
            content, _ = research_agent(prompt=enriched_task)
            return step_title, "research_agent", content
        # ... other agents
    except ValueError as e:
        raise ValueError(f"Unknown step type: {step_title}")

# Layer 4: Service Layer
def run_agent_workflow(task_id, prompt, initial_plan_steps):
    try:
        for i, plan_step_title in enumerate(initial_plan_steps):
            actual_step_description, agent_name, output = \
                executor_agent_step(plan_step_title, execution_history, prompt)
            # ... process results
    
    except Exception as e:
        print(f"Workflow error for task {task_id}: {e}")
        
        # Update step status to error
        error_step_index = next(...)
        update_step_status(error_step_index, "error", 
                          f"Error during execution: {e}",
                          {"title": "Error", "content": str(e)})
        
        # Update DB
        db = SessionLocal()
        task = db.query(Task).filter(Task.id == task_id).first()
        task.status = "error"
        db.commit()
        db.close()

# Layer 5: API Layer
@app.get("/task_status/{task_id}")
def get_task_status(task_id: str):
    db = SessionLocal()
    task = db.query(Task).filter(Task.id == task_id).first()
    db.close()
    
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    
    return {
        "status": task.status,
        "result": json.loads(task.result) if task.result else None
    }
```

### Error Display Flow

```
Error occurs in agent layer
         │
         ↓
Caught and formatted with context
         │
         ↓
Stored in step.substeps as "Error" substep
         │
         ↓
task.status = "error" (DB)
         │
         ↓
Frontend detects status = "error"
         │
         ↓
Displays error message in UI
         │
         ↓
User can see exactly what failed and where
```

---

## Deployment Architecture

### Docker Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                  Docker Container                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Process 1: PostgreSQL                              │   │
│  │  ├─ Service: pg_ctlcluster 17 main start            │   │
│  │  ├─ Listen: 127.0.0.1:5432                          │   │
│  │  ├─ Database: appdb                                 │   │
│  │  └─ User: app (password: local)                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                        ↑                                     │
│                        │ (Unix socket)                       │
│                        │                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Process 2: FastAPI (Uvicorn)                       │   │
│  │  ├─ Service: uvicorn main:app --host 0.0.0.0       │   │
│  │  ├─ Port: 8000                                      │   │
│  │  ├─ Workers: 1 (sync)                               │   │
│  │  └─ Auto-reload: disabled (production)              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Shared Volume:                                              │
│  ├─ /app                        (application code)           │
│  ├─ /var/lib/postgresql/...     (database files)            │
│  └─ /tmp                        (temporary files)            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         ↑                                      ↑
         │ Port 8000 (mapped)                  │ Port 5432 (mapped)
         │                                     │
┌────────┴─────────────────────────────────────┴──────────────┐
│                   Host Machine                               │
│  http://localhost:8000 ← Browser connects here             │
└─────────────────────────────────────────────────────────────┘
```

### Environment Setup

```
HOST MACHINE:
├─ docker build -t agentic-ai .
│  ├─ Builds image from Dockerfile
│  ├─ Installs Python 3.11
│  ├─ Installs PostgreSQL service
│  ├─ Installs Python packages from requirements.txt
│  └─ Copies code into /app

└─ docker run -p 8000:8000 -p 5432:5432 \
             --env-file .env \
             --name agentic-ai \
             agentic-ai

   CONTAINER STARTS:
   ├─ Entrypoint: /entrypoint.sh
   ├─ Start Postgres
   ├─ Create user/database if needed
   ├─ Export DATABASE_URL
   ├─ Launch Uvicorn
   └─ Listen on 0.0.0.0:8000
```

### Configuration via Environment

```
.env file (host):
├─ OPENAI_API_KEY=sk-xxxx...
├─ TAVILY_API_KEY=tvly-xxxx...
├─ POSTGRES_USER=app
├─ POSTGRES_PASSWORD=local
└─ POSTGRES_DB=appdb

Passed to container:
├─ Via --env-file .env
├─ Read by:
│  ├─ entrypoint.sh (Postgres setup)
│  ├─ main.py (Database connection)
│  ├─ agents.py (LLM calls)
│  └─ research_tools.py (API keys)
```

---

## Key Architectural Decisions

### 1. **Layered Architecture**
**Why**: Separation of concerns, easier testing, scalability
**Tradeoff**: Slight performance overhead from multiple layers

### 2. **In-Memory + Database State**
**Why**: Fast real-time updates + persistent storage
**Tradeoff**: Potential race conditions (mitigated by polling)

### 3. **Threading for Long-Running Tasks**
**Why**: Non-blocking API, responsive UI
**Tradeoff**: Thread safety considerations

### 4. **LLM Tool Calling**
**Why**: Flexible agent behavior, no hardcoded tool selection
**Tradeoff**: Dependent on LLM capabilities (sometimes hallucinates tools)

### 5. **Single Container Deployment**
**Why**: Simple development and testing
**Tradeoff**: Not suitable for production (scale, resilience)

### 6. **Polling for Real-Time Updates**
**Why**: Simple implementation, works with HTTP
**Tradeoff**: Not true real-time (up to 2-second delay)

---

## Future Architectural Improvements

```
CURRENT:                          FUTURE:
├─ Single thread per task   →     ├─ Message queue (Celery)
├─ In-memory state          →     ├─ Redis cache
├─ Database-only persistence →     ├─ Event streaming
├─ HTTP polling             →     ├─ WebSocket real-time
├─ Single container         →     ├─ Microservices
├─ SQLite/PostgreSQL        →     └─ Distributed tracing
└─ Basic error handling           & resilience
```

---

## Architecture Summary

Your project uses a **layered, event-driven architecture** where:

1. **Frontend** collects user input
2. **API Layer** routes requests
3. **Service Layer** manages task lifecycle
4. **Orchestration** coordinates agents
5. **Agents** perform specific tasks
6. **Tools** fetch external data
7. **Database** persists results
8. **Threading** enables concurrency

This design is **simple, maintainable, and suitable for development/testing**, with clear upgrade paths to production-grade architectures.

