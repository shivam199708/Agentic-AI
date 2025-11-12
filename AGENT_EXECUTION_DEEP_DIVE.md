# Deep Dive: Agent Execution Flow

## Understanding How Agents Work

### The Complete Workflow Execution Path

```
USER SUBMITS: "Explain quantum computing fundamentals"
                                │
                                ↓
                    ┌─────────────────────────────┐
                    │  Planner Agent               │
                    ├─────────────────────────────┤
                    │                             │
                    │ PROMPT:                     │
                    │ "Create a 7-step research   │
                    │  plan for: Quantum          │
                    │  computing fundamentals"    │
                    │                             │
                    │ TOOLS: None (just LLM)      │
                    │ MODEL: openai:gpt-4o-mini   │
                    │ TEMP: 1.0                   │
                    └────────────┬────────────────┘
                                 │
                                 ↓ (LLM generates)
                    ┌─────────────────────────────┐
                    │ PARSED STEPS:               │
                    ├─────────────────────────────┤
                    │ [                           │
                    │ "Research agent: Use Tavily │
                    │  to perform a broad web     │
                    │  search...",                │
                    │                             │
                    │ "Research agent: For each   │
                    │  collected item, search on  │
                    │  arXiv...",                 │
                    │                             │
                    │ "Research agent: Synthesize │
                    │  and rank findings...",     │
                    │                             │
                    │ "Writer agent: Draft a      │
                    │  structured outline...",    │
                    │                             │
                    │ "Editor agent: Review for   │
                    │  coherence...",             │
                    │                             │
                    │ "Writer agent: Generate     │
                    │  the final comprehensive    │
                    │  Markdown report...",       │
                    │                             │
                    │ "Editor agent: Final review │
                    │  and polish..."             │
                    │ ]                           │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Validation & Contract Enforcement             │
                    ├────────────────────────────────────────────────┤
                    │ ✓ First step = Tavily search                  │
                    │ ✓ Second step = arXiv search                  │
                    │ ✓ Final step = Writer agent (final report)    │
                    │ ✓ Max 7 steps (cap to length)                 │
                    │ ✓ All strings valid                           │
                    └────────────┬───────────────────────────────────┘
                                 │
                                 ↓
                    ┌────────────────────────────────────────────────┐
                    │  Step 1: Tavily Web Search                     │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ CONTEXT BUILT:                                 │
                    │ ┌────────────────────────────────────────────┐ │
                    │ │ 📘 User Prompt:                            │ │
                    │ │ "Explain quantum computing fundamentals"  │ │
                    │ │                                            │ │
                    │ │ 📜 History so far: (empty)                │ │
                    │ │                                            │ │
                    │ │ 🧩 Your next task:                         │ │
                    │ │ "Research agent: Use Tavily to perform a  │ │
                    │ │  broad web search and collect top relevant│ │
                    │ │  items..."                                 │ │
                    │ └────────────────────────────────────────────┘ │
                    │                                                │
                    │ AGENT INVOKED:                                 │
                    │ research_agent(prompt=enriched_task)           │
                    │                                                │
                    │ AGENT LOGIC:                                   │
                    │ 1. Reads full prompt                           │
                    │ 2. Understands: need web search + research     │
                    │ 3. Identifies available tools:                 │
                    │    - tavily_search_tool                        │
                    │    - arxiv_search_tool                         │
                    │    - wikipedia_search_tool                     │
                    │ 4. Sends to LLM with tool definitions         │
                    │ 5. LLM decides: "I need tavily_search_tool"   │
                    │ 6. Tool call made:                             │
                    │    tavily_search_tool(                         │
                    │      query="quantum computing fundamentals"    │
                    │    )                                           │
                    │                                                │
                    │ TOOL EXECUTION:                                │
                    │ 📡 HTTP Call: POST https://api.tavily.com     │
                    │ └─→ Returns: [                                 │
                    │       {                                        │
                    │         "title": "Quantum Computing 101",      │
                    │         "content": "Quantum computing uses...", │
                    │         "url": "https://..."                  │
                    │       },                                       │
                    │       {                                        │
                    │         "title": "How Qubits Work",            │
                    │         "content": "Unlike bits...",           │
                    │         "url": "https://..."                  │
                    │       },                                       │
                    │       ...                                      │
                    │     ]                                          │
                    │                                                │
                    │ LLM SYNTHESIS:                                 │
                    │ LLM reads results and writes summary           │
                    │ Format: HTML with formatted results            │
                    │                                                │
                    │ OUTPUT:                                        │
                    │ ┌────────────────────────────────────────────┐ │
                    │ │ "Based on Tavily search, found these      │ │
                    │ │ authoritative sources:                     │ │
                    │ │                                            │ │
                    │ │ 1. **Quantum Computing 101** - Explains    │ │
                    │ │    fundamental concepts of quantum         │ │
                    │ │    computing...                            │ │
                    │ │                                            │ │
                    │ │ 2. **How Qubits Work** - Details the      │ │
                    │ │    difference between qubits and bits...   │ │
                    │ │                                            │ │
                    │ │ 3. **Quantum Gates Explained** - ...        │ │
                    │ │                                            │ │
                    │ │ <h2>📎 Tools Used</h2>                     │ │
                    │ │ <ul>                                        │ │
                    │ │ <li>tavily_search(                         │ │
                    │ │     query='quantum computing               │ │
                    │ │     fundamentals'                          │ │
                    │ │ )</li>                                      │ │
                    │ │ </ul>"                                      │ │
                    │ └────────────────────────────────────────────┘ │
                    │                                                │
                    │ STORED:                                        │
                    │ execution_history.append([                     │
                    │   "Research agent: Use Tavily...",             │
                    │   "Use Tavily to perform a broad web search",  │
                    │   output_text                                  │
                    │ ])                                             │
                    │                                                │
                    │ UI UPDATED:                                    │
                    │ task_progress[task_id].steps[0].substeps = [   │
                    │   {                                            │
                    │     "title": "Called research_agent",          │
                    │     "content": "<div>...output...</div>"       │
                    │   }                                            │
                    │ ]                                              │
                    │ task_progress[task_id].steps[0].status = "done"│
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Step 2: arXiv Paper Search                    │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ CONTEXT BUILT (NEW):                           │
                    │ ┌────────────────────────────────────────────┐ │
                    │ │ 📘 User Prompt:                            │ │
                    │ │ "Explain quantum computing fundamentals"  │ │
                    │ │                                            │ │
                    │ │ 📜 History so far:                         │ │
                    │ │ 🔍 Research (Step 1):                      │ │
                    │ │ Based on Tavily search, found:             │ │
                    │ │ - Quantum Computing 101                    │ │
                    │ │ - How Qubits Work                          │ │
                    │ │ - Quantum Gates Explained                  │ │
                    │ │                                            │ │
                    │ │ 🧩 Your next task:                         │ │
                    │ │ "Research agent: For each collected item,  │ │
                    │ │  search on arXiv to find matching          │ │
                    │ │  preprints..."                              │ │
                    │ └────────────────────────────────────────────┘ │
                    │                                                │
                    │ AGENT INVOKED (again):                         │
                    │ research_agent(prompt=enriched_task)           │
                    │                                                │
                    │ LLM DECIDES:                                   │
                    │ "I need to search arXiv for quantum            │
                    │  computing papers. Should match titles from    │
                    │  the Tavily search."                           │
                    │                                                │
                    │ TOOL CALL:                                     │
                    │ arxiv_search_tool(query="quantum computing")   │
                    │                                                │
                    │ TOOL EXECUTION:                                │
                    │ 📡 API: https://export.arxiv.org/api/query     │
                    │ └─→ Parse XML response                         │
                    │ └─→ For each paper:                            │
                    │     - Fetch PDF from arXiv                     │
                    │     - Extract text (first 6 pages)             │
                    │     - Clean and truncate                       │
                    │     - Return structured result                 │
                    │                                                │
                    │ RESULTS:                                       │
                    │ [                                              │
                    │   {                                            │
                    │     "title": "Quantum Computing...",            │
                    │     "authors": ["Alice Smith", ...],           │
                    │     "published": "2023-05-15",                 │
                    │     "url": "https://arxiv.org/abs/...",        │
                    │     "summary": "The first 5000 chars of       │
                    │                 extracted PDF text...",        │
                    │     "link_pdf": "https://arxiv.org/pdf/..."    │
                    │   },                                           │
                    │   ...                                          │
                    │ ]                                              │
                    │                                                │
                    │ STORED & UPDATED: (similar to Step 1)          │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Step 3: Research Synthesis                    │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ Similar pattern:                               │
                    │ - Context includes all previous steps          │
                    │ - Research agent synthesizes & ranks findings  │
                    │ - Deduplicates by title/DOI                    │
                    │ - Returns ranked list                          │
                    │                                                │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Step 4: Writer Agent (Draft)                  │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ KEY DIFFERENCE: Writer agent has different    │
                    │ system prompt (not research-focused)           │
                    │                                                │
                    │ WRITER SYSTEM PROMPT:                          │
                    │ "You are an expert academic writer...          │
                    │  Synthesize research materials into a          │
                    │  comprehensive, well-structured academic       │
                    │  report in Markdown format.                    │
                    │                                                │
                    │  MANDATORY STRUCTURE:                          │
                    │  1. Title                                      │
                    │  2. Abstract                                   │
                    │  3. Introduction                               │
                    │  4. Background/Literature Review               │
                    │  5. Methodology (if applicable)                │
                    │  6. Key Findings/Results                       │
                    │  7. Discussion                                 │
                    │  8. Conclusion                                 │
                    │  9. References                                 │
                    │                                                │
                    │  CITATION RULES:                               │
                    │  - Use numeric inline [1], [2], etc.           │
                    │  - Every claim MUST have a citation            │
                    │  - Complete References section                 │
                    │  - Preserve all URLs/DOIs"                     │
                    │                                                │
                    │ CONTEXT INCLUDES:                              │
                    │ - All research findings from Steps 1-3          │
                    │ - User's original prompt                       │
                    │ - Previous execution history                   │
                    │                                                │
                    │ NO TOOLS: Writer doesn't call tools            │
                    │ Just LLM creating structured Markdown          │
                    │                                                │
                    │ OUTPUT:                                        │
                    │ ┌────────────────────────────────────────────┐ │
                    │ │ # Quantum Computing: Fundamentals and     │ │
                    │ │   Applications                             │ │
                    │ │                                            │ │
                    │ │ ## Abstract                                │ │
                    │ │ Quantum computing represents a paradigm    │ │
                    │ │ shift in computational power [1]...        │ │
                    │ │                                            │ │
                    │ │ ## Introduction                            │ │
                    │ │ Classical computers process bits [2]...    │ │
                    │ │                                            │ │
                    │ │ ...detailed content...                     │ │
                    │ │                                            │ │
                    │ │ ## References                              │ │
                    │ │ [1] Smith, A., et al. (2023). "Quantum    │ │
                    │ │     Computing Advances"                    │ │
                    │ │     https://arxiv.org/abs/...              │ │
                    │ │                                            │ │
                    │ │ [2] Johnson, B. (2022). "Quantum Bits     │ │
                    │ │     Explained"                             │ │
                    │ │     https://example.com/quantum-bits       │ │
                    │ │ ...                                        │ │
                    │ └────────────────────────────────────────────┘ │
                    │                                                │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Step 5: Editor Agent (Review)                 │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ KEY DIFFERENCE: Review-focused system prompt   │
                    │                                                │
                    │ EDITOR SYSTEM PROMPT:                          │
                    │ "You are a professional academic editor...     │
                    │  Refine and elevate the quality of the        │
                    │  academic text provided.                       │
                    │                                                │
                    │  YOUR EDITING PROCESS:                         │
                    │  1. Analyze overall structure and flow         │
                    │  2. Improve clarity and precision              │
                    │  3. Verify technical accuracy                  │
                    │  4. Enhance readability                        │
                    │  5. Preserve all citations [1], [2]...         │
                    │  6. Maintain References section integrity"    │
                    │                                                │
                    │ INPUT: Draft from Step 4                       │
                    │ OUTPUT: Enhanced version with same structure   │
                    │                                                │
                    │ NO TOOLS: Just LLM reviewing & improving       │
                    │                                                │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Step 6-7: More Agents (as needed)             │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ Depending on plan:                             │
                    │ - More research iterations                     │
                    │ - Final writer pass                            │
                    │ - Final editor pass                            │
                    │                                                │
                    │ Each follows same pattern:                     │
                    │ - Context built with full history              │
                    │ - Agent called                                 │
                    │ - Output stored                                │
                    │ - Substep added to UI                          │
                    │ - Step marked done                             │
                    │                                                │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Workflow Complete                             │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ FINAL REPORT EXTRACTION:                       │
                    │ final_report = execution_history[-1][-1]       │
                    │ (The output from the last agent call)          │
                    │                                                │
                    │ RESULT OBJECT:                                 │
                    │ {                                              │
                    │   "html_report": "# Quantum Computing...",    │
                    │   "history": [                                 │
                    │     {                                          │
                    │       "step": 1,                               │
                    │       "title": "Research: Tavily",             │
                    │       "description": "...",                    │
                    │       "output": "..."                          │
                    │     },                                         │
                    │     {...},                                     │
                    │     ...                                        │
                    │   ]                                            │
                    │ }                                              │
                    │                                                │
                    │ DATABASE UPDATE:                               │
                    │ task.status = "done"                           │
                    │ task.result = JSON.dumps(result_object)        │
                    │ task.updated_at = now()                        │
                    │ db.commit()                                    │
                    │                                                │
                    │ THREAD TERMINATES                              │
                    │                                                │
                    └────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────────────────────────────────────────┐
                    │  Frontend Shows Final Report                   │
                    ├────────────────────────────────────────────────┤
                    │                                                │
                    │ 1. Polling detects status = "done"             │
                    │ 2. Fetches /task_status/{task_id}              │
                    │ 3. Gets result from DB                         │
                    │ 4. Renders Markdown as HTML                    │
                    │ 5. Displays:                                   │
                    │    - Download .md button                       │
                    │    - Download .html button                     │
                    │    - Full formatted report                     │
                    │    - Clickable references with links           │
                    │                                                │
                    └────────────────────────────────────────────────┘
```

## Key Takeaways

### Agent Decision Making

```
Each agent receives:
1. User's original prompt → maintains context
2. Full execution history → builds on previous work
3. Current task description → specific instructions
4. Tool definitions (if applicable) → knows what's available

Agent decides:
- Which tools to use (if any)
- What to ask them
- How to format results
- What to output

Key: Agents are flexible, not hardcoded
LLM determines the actual execution path
```

### Tool Calling Pattern

```
Agent Layer (agents.py):
  client.chat.completions.create(
    model="openai:gpt-4.1-mini",
    messages=[...],
    tools=[arxiv_search_tool, tavily_search_tool, wikipedia_search_tool],
    tool_choice="auto"  ← LLM decides which tools to call
  )

LLM Response:
  {
    "tool_calls": [
      {
        "function": {
          "name": "tavily_search_tool",
          "arguments": {"query": "quantum computing"}
        }
      }
    ]
  }

Tool Layer (research_tools.py):
  tavily_search_tool(query="quantum computing")
    └─→ Makes actual HTTP call to Tavily API
    └─→ Returns results

Back to LLM:
  Sees tool results
  Synthesizes into natural language
  Returns final answer
```

### Context Building

```
Initial:
  "Explain quantum computing fundamentals"

After Step 1 (Tavily search):
  "Explain quantum computing fundamentals
   
   Previous research findings:
   - Quantum Computing 101 (source: Tavily)
   - How Qubits Work (source: Tavily)
   ..."

After Step 2 (arXiv search):
  "Explain quantum computing fundamentals
   
   Previous research findings:
   - Quantum Computing 101 (source: Tavily)
   - How Qubits Work (source: Tavily)
   
   Academic papers found:
   - Smith et al. (2023) on Quantum Computing
   - Johnson (2022) on Quantum Gates
   ..."

After Step 3 (synthesis):
  "Explain quantum computing fundamentals
   
   Research synthesis:
   - Authoritative sources ranked
   - Deduplication complete
   - Key concepts: qubits, superposition, entanglement
   ..."

Step 4+ uses this accumulated context
```

### Why This Architecture Works

```
✅ Flexible: Agents adapt to different tasks
✅ Composable: Chain multiple agents together
✅ Transparent: Step-by-step progress visible
✅ Reproducible: Deterministic results for same input
✅ Scalable: Can add new agents without changing core
✅ Maintainable: Each agent isolated and testable
✅ Powerful: LLM + tools + planning = sophisticated workflows
```

