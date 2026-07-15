# 🏗️ Week 4 — System Design Questions

> **Focus:** End-to-end AI application architecture, multi-tool orchestration, deployment, observability, scaling from prototype to production
>
> **How to use:** 30-45 minute whiteboard exercises. Start with requirements, draw architecture, deep-dive into components.

---

## Q1. Design a Production-Ready Multi-Tool AI Assistant ⭐⭐⭐

**Prompt:** "Design an AI assistant that supports 15+ tools (calculator, weather, web search, email, calendar, database queries, code execution, file operations). It needs to serve 1000 concurrent users with conversation memory, tool orchestration, and a web UI."

**Architecture:**

```
┌──────────────────── Frontend (Next.js / Streamlit) ─────────────────┐
│  Chat interface · Tool result display · Session management          │
│  Conversation export · Analytics sidebar                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │ REST / WebSocket
┌────────────────────────────▼────────────────────────────────────────┐
│                    API Gateway (FastAPI)                             │
│  Auth · Rate limiting · Session management · Request logging        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│              Orchestration Layer                                     │
│  ┌─────────────────────┐  ┌───────────────────────┐                │
│  │  Conversation Memory │  │  Context-Aware Router  │               │
│  │  (Redis per session) │  │  (LLM-based routing)   │               │
│  └─────────────────────┘  └───────────┬───────────┘                │
│                                       │                             │
│  ┌────────────────────────────────────▼──────────────────────────┐  │
│  │  Tool Registry (15+ tools)                                    │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐     │  │
│  │  │Calc  │ │Weath.│ │Search│ │Email │ │Cal.  │ │DB    │ ... │  │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────┐  ┌───────────────────────┐                │
│  │  Validation Layer    │  │  Response Formatter    │               │
│  │  (Pydantic schemas)  │  │  (Markdown + citations)│               │
│  └─────────────────────┘  └───────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│              Observability Layer                                     │
│  Structured logging · Trace IDs · Cost tracking · Export · Alerting │
└─────────────────────────────────────────────────────────────────────┘
```

**Key decisions:**

**Memory in Redis, not in-process:** At 1000 concurrent users, in-memory `dict` doesn't scale. Redis gives per-session key-value storage with TTL (auto-expire stale sessions), survives server restarts, and enables horizontal scaling (any server can serve any session).

**Two-stage routing for 15+ tools:** With 15+ tool descriptions in the prompt, the LLM takes longer and makes more mistakes. Solution: first classify CATEGORY (math, weather, communication, data, code), then route within that category. Each stage uses a smaller, more focused prompt.

**Rate limiting per user per tool:** Some tools are expensive (code execution, web search). Rate limit: 60 requests/min overall, 10/min for expensive tools, 5/min for write operations (email, calendar).

---

## Q2. Design the Conversation Memory Architecture for a Multi-User AI App ⭐⭐⭐

**Prompt:** "Your AI assistant has 10,000 daily active users. Each user expects their conversation to persist across sessions (close the tab, come back tomorrow, continue). Design the memory architecture."

**Storage design:**

```
┌─────────────────────────────────────────────────────────────┐
│  Tier 1: Hot Memory (Redis)                                  │
│  - Active session context (last 20 turns)                    │
│  - TTL: 30 minutes of inactivity → evict to Tier 2          │
│  - Key: session:{user_id}:{session_id}                       │
│  - Format: JSON array of Turn objects                        │
└────────────────────────┬────────────────────────────────────┘
                         │ On TTL expiration or session end
┌────────────────────────▼────────────────────────────────────┐
│  Tier 2: Warm Storage (PostgreSQL)                           │
│  - Complete conversation history per user                    │
│  - Indexed by user_id, session_id, timestamp                 │
│  - Used for: "continue last conversation", analytics         │
└────────────────────────┬────────────────────────────────────┘
                         │ For semantic search ("what did I ask about X?")
┌────────────────────────▼────────────────────────────────────┐
│  Tier 3: Semantic Memory (Vector DB)                         │
│  - Embedded turns for retrieval                              │
│  - "Remember when I asked about weather in Paris last week?" │
│  - Filtered by user_id (tenant isolation)                    │
└─────────────────────────────────────────────────────────────┘
```

**Session resume flow:**
1. User opens app → check Redis for active session → if found, load hot context
2. If no active session → check PostgreSQL for last session → load summary + last 5 turns into Redis
3. User says "what did we discuss about X?" → vector search across all past sessions for user

**Cost at scale:**
- Redis: 10K users × ~5KB per session = 50MB (trivial)
- PostgreSQL: 10K users × 100 sessions × 20 turns × ~1KB = 20GB (manageable)
- Vector DB: 10K users × 2000 embedded turns × 6KB = 120GB (moderate)

---

## Q3. Design a Testing Strategy for an End-to-End AI Application ⭐⭐⭐⭐

**Prompt:** "Your AI assistant has a UI, 10 tools, conversation memory, and serves customers. Design a comprehensive testing strategy that gives you confidence to deploy on Friday afternoon."

**Testing pyramid:**

```
                    ╱╲
                   ╱  ╲        Human QA (weekly)
                  ╱    ╲        — 10 testers, 50 scenarios
                 ╱──────╲
                ╱        ╲      Eval Suite (on PR merge)
               ╱ LLM Eval ╲     — 200+ golden test cases
              ╱────────────╲    — Tool selection accuracy > 95%
             ╱              ╲   — Response quality > 85%
            ╱  End-to-End    ╲
           ╱   Integration    ╲  E2E Tests (on every PR)
          ╱────────────────────╲ — Full conversation flows
         ╱                      ╲— Multi-turn context tests
        ╱    Component Tests     ╲ Component Tests (on every commit)
       ╱    (mocked LLM)         ╲ — Router logic, memory management
      ╱────────────────────────────╲— Validation, error handling
     ╱                              ╲
    ╱       Unit Tests               ╲ Unit Tests (on every commit)
   ╱       (deterministic)            ╲ — Tool functions, formatters
  ╱────────────────────────────────────╲— Parsers, exporters
```

**Layer details:**

```python
# Unit: deterministic, fast, test pure functions
def test_calculator_division_by_zero():
    result = calculator("1/0")
    assert result["success"] == False
    assert "division" in result["error"].lower()

# Component: mock the LLM, test YOUR logic
def test_memory_enforces_token_budget(mocker):
    memory = ConversationMemory(max_tokens=100)
    for i in range(50):
        memory.add("user", f"Message {i} with enough text to consume tokens")
    assert memory._total_tokens() <= 100

# Integration: real LLM call, test full pipeline
@pytest.mark.integration
def test_full_conversation_flow():
    session = AISession()
    r1 = session.send("What's the weather in Paris?")
    assert "temperature" in r1.lower() or "°" in r1
    r2 = session.send("What about London?")
    assert "London" in r2  # Context carried over

# Eval: statistical quality measurement
@pytest.mark.eval
def test_tool_selection_accuracy():
    correct = 0
    for case in eval_dataset:
        result = router.route(case["query"])
        if result.tool == case["expected_tool"]:
            correct += 1
    accuracy = correct / len(eval_dataset)
    assert accuracy >= 0.95, f"Tool accuracy {accuracy:.1%} below 95%"
```

---

## Q4. Design a Deployment Pipeline for a Streamlit + FastAPI AI App ⭐⭐⭐

**Prompt:** "You have a Streamlit frontend and a FastAPI backend. Design the deployment architecture for a production app with 500 daily users."

```
┌──────────────── CI/CD (GitHub Actions) ──────────────────────┐
│                                                               │
│  On PR:     lint → unit tests → component tests               │
│  On merge:  + integration tests → eval suite → build → deploy │
│  Weekly:    + full eval suite with LLM-as-judge               │
│                                                               │
└───────────────────────────┬───────────────────────────────────┘
                            │
              ┌─────────────┼─────────────────┐
              ▼                               ▼
┌──────────────────────┐       ┌──────────────────────────┐
│  Streamlit Cloud      │       │  Railway / Render         │
│  (or Vercel for React)│       │  (FastAPI backend)        │
│                       │       │                           │
│  Frontend container   │──────→│  API container            │
│  - Chat UI            │  REST │  - LLM orchestration      │
│  - Session management │       │  - Tool execution         │
│  - Export/analytics   │       │  - Memory (Redis)         │
└──────────────────────┘       │  - Observability          │
                               └──────────────────────────┘
                                         │
                               ┌─────────┼─────────┐
                               ▼         ▼         ▼
                           [Redis]   [Postgres]  [LLM API]
                           Sessions  History     OpenAI/Anthropic
```

**Environment strategy:**
```
development  → local machine, SQLite, mock LLM responses
staging      → Railway, real APIs with staging keys, low rate limits
production   → Railway, production keys, full rate limits, monitoring
```

---

## Q5. Design an Observability System for an AI Assistant ⭐⭐⭐

**Prompt:** "Users report the AI gives wrong answers sometimes but you can't reproduce it. Design an observability system that lets you debug any conversation after the fact."

**Solution: Full conversation replay**

Every conversation gets a trace stored in append-only log:

```json
{
  "trace_id": "conv_abc123",
  "user_id": "user_456",
  "turns": [
    {
      "turn_id": 1,
      "user_input": "What's 25% of 480?",
      "routing": {
        "model": "gpt-4o-mini",
        "decision": "calculator",
        "confidence": 0.95,
        "alternatives": ["general_chat"],
        "latency_ms": 340
      },
      "tool_execution": {
        "tool": "calculator",
        "input": {"expression": "0.25 * 480"},
        "output": 120.0,
        "success": true,
        "latency_ms": 2
      },
      "generation": {
        "model": "gpt-4o-mini",
        "input_tokens": 450,
        "output_tokens": 25,
        "response": "25% of 480 is exactly 120.",
        "latency_ms": 890
      },
      "total_latency_ms": 1232,
      "cost_usd": 0.0003
    }
  ]
}
```

**What this enables:**
- "The AI said X was wrong" → pull trace → see exactly what tool returned and how the LLM used it
- "Why is this slow?" → latency breakdown per step shows the bottleneck
- "What model version was used?" → every call is tagged
- "Can I reproduce this?" → replay the exact inputs to routing and generation

---

## Q6. Design a Multi-Tool Parallel Execution System ⭐⭐⭐

**Prompt:** "The user asks 'What's the weather in Paris and convert 72°F to Celsius?' — this needs two tools. How do you detect and execute parallel tool calls?"

**Detection:** The router identifies that the query requires multiple independent tools:
```python
class MultiToolDecision(BaseModel):
    tools: list[ToolCall]
    parallel: bool  # Can these run simultaneously?
    dependencies: dict[str, list[str]]  # tool_name → depends_on
```

**Execution:**
```python
async def execute_tools(decision: MultiToolDecision) -> list[ToolResult]:
    if decision.parallel and not decision.dependencies:
        # Independent tools — run simultaneously
        results = await asyncio.gather(*[
            execute_tool(tc) for tc in decision.tools
        ])
    else:
        # Sequential — results feed into next tool
        results = []
        for tc in decision.tools:
            result = await execute_tool(tc)
            results.append(result)
    
    return results
```

**Why this matters:** Sequential execution of 2 tools at 2s each = 4s latency. Parallel = 2s. At 5 tools, the difference is 10s vs 2s. User experience difference between "this is fast" and "this is broken."

---

## Q7-Q15: Additional System Design Questions (Condensed)

### Q7. Design a Cost Control System for an AI App ⭐⭐⭐
Per-user daily token budgets, per-tool cost tracking, alerts at 80% budget, hard cutoff at 100%. Dashboard showing cost per user, per tool, per day. Model routing (expensive model only for complex queries).

### Q8. Design a Conversation Branching System ⭐⭐⭐⭐
Users can "go back" to any turn and ask a different question, creating a conversation tree. Each branch maintains independent context. UI shows branch points and allows switching. Key: tree structure with immutable nodes, pointer to "current branch."

### Q9. Design an A/B Testing Framework for Prompts in a Live AI App ⭐⭐⭐
Sticky user assignment (same user always sees same variant). Split by user_id hash. Track: response quality, tool accuracy, user feedback, latency, cost. Auto-rollback if quality drops >5%. Minimum 1000 requests per variant for statistical significance.

### Q10. Design a Graceful Degradation System When the LLM API Goes Down ⭐⭐⭐
Tier 1: Switch to fallback model (gpt-4o → gpt-4o-mini). Tier 2: Switch to different provider (OpenAI → Anthropic). Tier 3: Return cached responses for common queries. Tier 4: Static fallback ("AI is temporarily unavailable. Here are common help topics..."). Each tier activates automatically based on health checks.

### Q11. Design a Plugin Architecture for Adding New Tools Without Deploying ⭐⭐⭐⭐
Tools defined as YAML config files or stored in database. Hot-reload: scan config directory periodically, register new tools without restart. Schema validation on registration. Permission system per tool. Key: separate tool definition from tool execution.

### Q12. Design a Session Handoff System (AI → Human Agent) ⭐⭐⭐
Detect escalation signals (user frustrated, AI confidence low, 3+ failed attempts). Package conversation context into handoff payload (summary, entities, attempted solutions). Route to available human agent with context pre-loaded. AI becomes "assistant to the human agent" — suggesting responses, pulling data.

### Q13. Design a Multi-Language AI Assistant ⭐⭐⭐
Detect input language. Route to language-specific system prompt. Tools return data in user's language (or translate results). Key challenge: routing accuracy drops in non-English languages — may need language-specific few-shot examples in router prompt.

### Q14. Design an Offline-First AI App ⭐⭐⭐
Cache frequently used tool results locally. Queue requests when offline, execute when connection returns. Local-only tools (calculator, converter) work without network. Key: service worker for caching, IndexedDB for conversation persistence, sync queue for pending requests.

### Q15. Design a White-Label AI Assistant Platform ⭐⭐⭐⭐
Multi-tenant: each customer gets custom branding, system prompts, tool access, and model selection. Config-driven: everything customizable via dashboard (no code changes). Shared infrastructure, isolated data. Billing per tenant based on usage. Key: tenant resolution from subdomain/API key, config loaded at request time.
