# 🏗️ Week 3 — System Design Questions

> **Focus:** Multi-agent architectures, tool orchestration platforms, function calling infrastructure at scale
>
> **How to use:** 30-45 minute whiteboard exercises. Draw architecture diagrams, discuss trade-offs, then deep-dive into components.

---

## Q1. Design a Multi-Agent Customer Support System ⭐⭐⭐⭐

**Prompt:** "Design an AI-powered customer support system that handles billing questions, technical troubleshooting, refund requests, and escalation to humans. It must serve 10,000 concurrent users, maintain conversation context, and comply with PII regulations."

**Architecture:**

```
User Message (chat / email / voice)
          │
          ▼
┌─────────────────────────────────┐
│        API Gateway / LB          │   Rate limiting, auth, TLS
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│     Intake & Classification      │
│  ┌───────────────────────────┐  │
│  │  PII Detector & Masker     │  │   Mask SSN, CC#, email before LLM sees it
│  │  Language Detector         │  │   Route to language-specific pipeline
│  │  Intent Classifier         │  │   billing / technical / refund / general
│  │  Priority Scorer           │  │   VIP customer? Repeat contact? Angry tone?
│  └───────────────────────────┘  │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│       Supervisor Agent           │   Orchestrates the right specialist
│  - Reads intent + priority       │
│  - Selects specialist agent      │
│  - Monitors for escalation       │
│  - Enforces conversation limits  │
└──────────┬──────────────────────┘
           │
     ┌─────┼─────────┬──────────────┐
     ▼     ▼         ▼              ▼
┌────────┐┌────────┐┌────────────┐┌────────────┐
│Billing ││Tech    ││Refund      ││Escalation  │
│Agent   ││Support ││Agent       ││Agent       │
│        ││Agent   ││            ││            │
│Tools:  ││Tools:  ││Tools:      ││Tools:      │
│-lookup ││-KB     ││-check      ││-create     │
│ invoice││ search ││ eligibility││ ticket     │
│-explain││-run    ││-process    ││-notify     │
│ charge ││ diag.  ││ refund     ││ human      │
│-apply  ││-status ││-send       ││-transfer   │
│ credit ││ check  ││ confirm.   ││ context    │
└────────┘└────────┘└────────────┘└────────────┘
           │
           ▼
┌─────────────────────────────────┐
│      Output Guardrails           │
│  - No unauthorized promises      │
│  - No PII in response            │
│  - Tone check (empathy score)    │
│  - Compliance rules enforced     │
└──────────┬──────────────────────┘
           │
           ▼
        Response to User
```

**Key design decisions:**

**Why multi-agent, not single agent?**
- Each specialist has a focused system prompt — better quality than one mega-prompt
- Different tools per agent — billing agent shouldn't access refund processing
- Independent scaling — tech support volume spikes differently than billing
- Security isolation — refund agent has write access to payment system; billing agent has read-only

**PII handling:**
```
User says: "My card ending 4242 was charged $50, my SSN is 123-45-6789"
          ↓
After PII masking: "My card ending [CARD_REDACTED] was charged $50, my SSN is [SSN_REDACTED]"
          ↓
LLM only sees masked version
          ↓
Actual card lookup uses original (via secure reference, never through LLM)
```

**Conversation context at 10K concurrent users:**
- Store conversation history in Redis (fast reads, TTL for auto-cleanup)
- Each conversation gets a session ID
- Context window management: summarize old messages when conversation exceeds token budget
- Agent handoffs pass a structured summary, not full history

**Escalation triggers (human-in-the-loop):**
- Sentiment score drops below threshold (customer getting angry)
- Agent confidence < 0.3 on two consecutive responses
- Customer explicitly asks for a human
- Refund amount exceeds $500 (business rule)
- Agent loop detected (3+ failed tool attempts)

**Scaling to 10K concurrent:**
- Supervisor agent is stateless — scales horizontally behind load balancer
- LLM calls are async — each server handles hundreds of concurrent conversations
- Tool calls are independently rate-limited per service
- Circuit breakers on every external dependency

---

## Q2. Design a Tool Orchestration Platform ⭐⭐⭐⭐

**Prompt:** "Your company has 30 internal tools (Slack, Jira, Salesforce, databases, custom APIs). Design a platform that lets any AI agent discover and use these tools safely, with access control, rate limiting, and audit logging."

**Architecture:**

```
AI Agents (various teams)
    │
    ▼
┌──────────────────────────────┐
│     Tool Orchestration API     │
│                                │
│  ┌──────────────────────────┐  │
│  │  Tool Discovery           │  │  "What tools can I use?"
│  │  - Lists available tools  │  │  - Filtered by agent's permissions
│  │  - Returns JSON schemas   │  │  - Includes descriptions for LLM
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  Permission Engine        │  │  "Am I allowed to do this?"
│  │  - RBAC per agent/user    │  │  - Tool-level + action-level
│  │  - Rate limits per agent  │  │  - Scope restrictions (read vs write)
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  Execution Engine         │  │  "Execute this tool call"
│  │  - Argument validation    │  │  - Pydantic schemas per tool
│  │  - Retry logic            │  │  - Timeout enforcement
│  │  - Output normalization   │  │  - Standard response format
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  Audit Logger             │  │  "Who did what, when?"
│  │  - Every tool call logged │  │  - Arguments + results + latency
│  │  - Immutable audit trail  │  │  - Compliance reporting
│  └──────────────────────────┘  │
│                                │
│  ┌──────────────────────────┐  │
│  │  MCP Server Registry      │  │  "Where are the tools?"
│  │  - Tool adapters (MCP)    │  │  - Health checks per tool
│  │  - Version management     │  │  - Schema validation
│  └──────────────────────────┘  │
└──────────────────────────────┘
         │
    ┌────┼────┬────────┬─────────┐
    ▼    ▼    ▼        ▼         ▼
  Slack Jira Salesforce DB    Custom
  MCP   MCP  MCP       MCP   MCP
```

**Tool registration:**
```python
# Each tool is registered with metadata
{
    "name": "jira_create_ticket",
    "description": "Create a new Jira ticket in a specified project",
    "category": "project_management",
    "risk_level": "medium",           # write operation
    "requires_confirmation": False,
    "rate_limit": 30,                  # per minute per agent
    "schema": {
        "project_key": {"type": "string", "required": True},
        "summary": {"type": "string", "required": True, "max_length": 200},
        "description": {"type": "string", "required": False},
        "priority": {"type": "string", "enum": ["low", "medium", "high"]},
    },
    "permissions": {
        "read": ["all_agents"],
        "write": ["engineering_agent", "pm_agent"],
    }
}
```

**Why MCP for tool adapters:**
- Standardized protocol — new tools plug in without changing the orchestration layer
- Health checking and version management built into the protocol
- Tools can be developed by different teams independently
- Agents discover tools at runtime — no code changes needed when tools are added

---

## Q3. Design a Function Calling Infrastructure for a Multi-Tenant SaaS ⭐⭐⭐

**Prompt:** "You're building an AI platform where customers can bring their own tools. Each customer has different tools, different schemas, and different security requirements. Design the function calling layer."

**Key challenges:**

**Tenant isolation:** Customer A's tools must never be visible to Customer B's agents.

**Dynamic schema management:** Each customer can register/update tool schemas without deploying code.

```
Tenant A → [Agent] → sees only Tenant A tools
Tenant B → [Agent] → sees only Tenant B tools
                         ↓
              ┌────────────────────┐
              │  Schema Registry    │
              │  (per-tenant)       │
              │  - Tool definitions │
              │  - Validation rules │
              │  - Version history  │
              └────────────────────┘
```

**Design:**

```python
class TenantToolRegistry:
    """
    Each tenant has an isolated set of tools.
    Schemas are stored in a database, not in code.
    """
    
    def get_tools_for_tenant(self, tenant_id: str) -> list[ToolDefinition]:
        # Returns only tools belonging to this tenant
        # + any shared/global tools (weather, calculator, etc.)
        pass
    
    def validate_tool_call(
        self, tenant_id: str, tool_name: str, arguments: dict
    ) -> ValidationResult:
        # Validates against tenant-specific schema
        schema = self.get_schema(tenant_id, tool_name)
        return schema.validate(arguments)
    
    def execute_tool(
        self, tenant_id: str, tool_name: str, arguments: dict
    ) -> ToolResult:
        # Execute within tenant's security context
        # - Use tenant's API keys (from secrets manager)
        # - Apply tenant's rate limits
        # - Log to tenant's audit trail
        pass
```

**Security model:**
- Tool schemas validated on registration (prevent malicious schemas)
- Customer API keys stored in tenant-isolated vaults
- Execution happens in sandboxed containers per tenant
- Network policies prevent cross-tenant communication
- All tool calls logged with tenant context for billing and compliance

---

## Q4. Design a Cost-Optimized AI Pipeline with Tool Calling ⭐⭐⭐

**Prompt:** "Your AI pipeline makes 5 tool calls per request on average, costs $0.15 per request, and handles 100K requests/day. The CEO wants to cut costs by 60% without degrading quality. Design the optimization."

**Current cost:** $0.15 × 100K = $15,000/day = $450,000/month

**Target:** $6,000/day = $180,000/month

**Optimization layers:**

```
Layer 1: Caching (saves 30-40%)
├── Exact-match cache: Same question → cached answer
├── Semantic cache: Similar questions → cached answer (embedding similarity > 0.95)
└── Tool result cache: Same tool + same args → cached result (TTL: 5-60 min)

Layer 2: Model routing (saves 20-30%)
├── Simple questions → gpt-4o-mini ($0.15/1M) instead of gpt-4o ($2.50/1M)
├── Classification step: 1 cheap call to classify → route to right model
└── Tool selection: cheap model selects tools, expensive model only for synthesis

Layer 3: Tool call optimization (saves 10-15%)
├── Parallel execution: Independent tools run simultaneously (saves latency, not cost, but reduces timeout failures)
├── Predictive tool selection: If 80% of "weather" queries need the same tool, skip the LLM routing step
└── Result compression: Summarize verbose tool outputs before passing to LLM

Layer 4: Request optimization (saves 5-10%)
├── Prompt compression: Remove redundant instructions, use shorter system prompts
├── Context window management: Don't pass full conversation history, summarize
└── Batch API: Non-real-time requests use 50% cheaper batch endpoint
```

**Implementation priority:**
1. Semantic caching → biggest ROI, least engineering effort
2. Model routing → second biggest savings
3. Batch API for offline workloads → easy win
4. Tool result caching → requires TTL tuning per tool

**Measuring success:**
- Cost per request (track daily)
- Quality metrics (eval suite — ensure accuracy doesn't drop)
- Cache hit rate (target: 30%+)
- Latency p50/p99 (should improve with caching)

---

## Q5. Design a Real-Time Agent Monitoring and Debugging Dashboard ⭐⭐⭐

**Prompt:** "Your team has 10 AI agents in production. Build a monitoring system that lets you see what they're doing in real-time, debug failures, and catch issues before users notice."

**What to monitor:**

```
Per-Agent Metrics:
├── Requests/minute (throughput)
├── Latency p50, p95, p99
├── Error rate (by error type)
├── Tool call distribution (which tools, how often)
├── Loop detection rate
├── Fallback trigger rate
├── Token usage & cost/minute
└── User satisfaction (thumbs up/down rate)

Per-Tool Metrics:
├── Call volume
├── Success rate
├── Latency
├── Error rate by type (timeout, validation, permission)
└── Cache hit rate

System Alerts:
├── Error rate > 5% (any agent) → page on-call
├── Latency p99 > 10s → warning
├── Loop rate > 1% → warning
├── Cost per request > 2x baseline → alert
├── Tool failure rate > 10% → circuit breaker
└── New error type detected → investigate
```

**Trace structure for debugging:**
```json
{
  "trace_id": "abc-123",
  "user_input": "What's my account balance?",
  "agent": "billing_agent",
  "total_latency_ms": 3200,
  "steps": [
    {"step": "classify_intent", "result": "billing_inquiry", "latency_ms": 150},
    {"step": "route_to_agent", "result": "billing_agent", "latency_ms": 5},
    {"step": "tool_call", "tool": "account_lookup", "args": {"user_id": "u_789"}, "latency_ms": 200, "success": true},
    {"step": "tool_call", "tool": "balance_check", "args": {"account": "acc_456"}, "latency_ms": 180, "success": true},
    {"step": "generate_response", "model": "gpt-4o-mini", "tokens": 150, "latency_ms": 800, "success": true},
    {"step": "output_guardrail", "checks": ["pii_free", "tone_ok"], "latency_ms": 50, "passed": true}
  ],
  "total_tool_calls": 2,
  "total_tokens": 580,
  "estimated_cost": "$0.003"
}
```

Every request gets a trace. Traces are searchable by: user ID, agent, tool, error type, latency range, time window. This is how you debug "why did the agent give the wrong answer at 3:47 PM yesterday" — pull the trace, see every step, see the exact tool outputs and LLM inputs.

---

## Q6. Design a Prompt Versioning and A/B Testing System ⭐⭐⭐

**Prompt:** "Your team changes prompts frequently. Design a system that versions prompts, A/B tests changes, and automatically rolls back if quality drops."

**Architecture:**

```
┌──────────────────────────────────────┐
│          Prompt Registry              │
│                                      │
│  prompt: "billing_agent_v3"          │
│  ├── v3.2 (canary - 10% traffic)    │
│  ├── v3.1 (active - 90% traffic)    │
│  └── v3.0 (previous - 0% traffic)   │
│                                      │
│  Each version stores:                │
│  - system_prompt text                │
│  - temperature + model settings      │
│  - eval scores at deployment time    │
│  - traffic allocation percentage     │
│  - deployment timestamp              │
│  - author + change description       │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│         Traffic Router                │
│                                      │
│  User request arrives →              │
│  Hash(user_id) % 100 →              │
│  0-9 → v3.2 (canary)               │
│  10-99 → v3.1 (stable)             │
│                                      │
│  Sticky routing: same user always    │
│  sees same version within a session  │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│         Eval Pipeline                 │
│                                      │
│  Continuous evaluation:              │
│  - Sample 5% of responses            │
│  - Run through eval suite            │
│  - Compare canary vs stable metrics  │
│                                      │
│  Auto-rollback if:                   │
│  - Canary accuracy < stable - 5%     │
│  - Canary error rate > stable + 2%   │
│  - Canary latency p99 > stable + 3s  │
│                                      │
│  Auto-promote if:                    │
│  - 24h with canary metrics >= stable │
│  - Minimum 1000 canary requests      │
└──────────────────────────────────────┘
```

**Prompt deployment workflow:**
```
Engineer edits prompt →
  Runs eval suite locally (must pass) →
  Opens PR with prompt + eval results →
  Peer review →
  Merge → auto-deploys as canary (10%) →
  Monitoring for 24h →
  Auto-promote to 100% OR auto-rollback
```

---

## Q7. Design a Secure Agent Gateway for Enterprise ⭐⭐⭐⭐

**Prompt:** "Enterprise customers want to use AI agents that access their internal tools (databases, APIs, file systems) but with strict security. No data should leave their network, agent actions must be auditable, and sensitive operations need approval."

**Core principle:** The agent reasons in the cloud, but tool execution happens inside the customer's network.

```
┌─── Customer's Network (VPC) ────────────────┐
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │         Agent Gateway (on-prem)          │  │
│  │                                          │  │
│  │  [Tool Executor]                         │  │
│  │  - Executes tool calls locally           │  │
│  │  - Customer's data never leaves VPC      │  │
│  │                                          │  │
│  │  [Approval Queue]                        │  │
│  │  - High-risk actions → human approval    │  │
│  │  - Slack/Teams integration for approvals │  │
│  │                                          │  │
│  │  [Audit Logger]                          │  │
│  │  - Immutable log of every action         │  │
│  │  - Stored in customer's SIEM             │  │
│  │                                          │  │
│  │  [Data Redaction Layer]                  │  │
│  │  - PII/sensitive data masked before      │  │
│  │    sending to cloud LLM                  │  │
│  │  - Results un-masked after receiving     │  │
│  └────────────┬────────────────────────────┘  │
│               │                               │
│  ┌────────────┼────────────────────────────┐  │
│  │  Internal   │   Tools                    │  │
│  │  Database   Salesforce   Jira   Sharepoint│ │
│  └─────────────────────────────────────────┘  │
└───────────────┼───────────────────────────────┘
                │  Only prompts + masked context
                │  cross the network boundary
                ▼
        ┌───────────────┐
        │  Cloud LLM API │
        │  (reasoning    │
        │   only)        │
        └───────────────┘
```

**Data flow:**
1. User asks question → Agent Gateway receives it
2. Gateway sends user message + tool definitions to cloud LLM
3. LLM returns tool call decision (no data, just "call tool X with args Y")
4. Gateway executes tool LOCALLY inside customer network
5. Gateway masks sensitive data in tool result
6. Sends masked result to LLM for synthesis
7. LLM generates response → Gateway un-masks → delivers to user

**What NEVER leaves the customer network:** Raw database records, PII, financial data, proprietary documents. Only the reasoning (prompts, tool selections, masked summaries) crosses the boundary.

---

## Q8. Design an Evaluation Framework for an Agent System ⭐⭐⭐⭐

**Prompt:** "Build an evaluation framework that tests your agent system across: tool selection accuracy, response quality, safety, and end-to-end task completion. Must run in CI/CD and block deploys that regress."

**Eval dimensions:**

```
┌─────────────────────────────────────────────────┐
│              Agent Eval Framework                 │
│                                                  │
│  Dimension 1: Tool Selection (deterministic)     │
│  ├── Given input X, did agent pick correct tool? │
│  ├── Did it pick the WRONG tool? (false pos)     │
│  └── Did it call tools when it shouldn't have?   │
│                                                  │
│  Dimension 2: Response Quality (LLM-judged)      │
│  ├── Faithfulness: does response match tool data? │
│  ├── Completeness: all user questions answered?   │
│  ├── Conciseness: no unnecessary information?     │
│  └── Tone: appropriate for context?               │
│                                                  │
│  Dimension 3: Safety (rule-based + LLM)          │
│  ├── Prompt injection resistance                  │
│  ├── No PII leakage                              │
│  ├── No unauthorized tool calls                   │
│  └── Guardrail compliance                         │
│                                                  │
│  Dimension 4: End-to-End (integration)           │
│  ├── Multi-step task completion rate              │
│  ├── Steps-to-completion (efficiency)             │
│  ├── Graceful handling of tool failures           │
│  └── Correct escalation behavior                  │
└─────────────────────────────────────────────────┘
```

**CI/CD integration:**
```yaml
# .github/workflows/agent-eval.yml
agent-eval:
  runs-on: ubuntu-latest
  steps:
    - name: Run tool selection eval
      run: pytest tests/eval/test_tool_selection.py --threshold 0.95
    
    - name: Run safety eval
      run: pytest tests/eval/test_safety.py --threshold 1.0  # Zero tolerance
    
    - name: Run quality eval (LLM-judged)
      run: pytest tests/eval/test_quality.py --threshold 0.85
    
    - name: Run end-to-end eval
      run: pytest tests/eval/test_e2e.py --threshold 0.80
    
    # Block deploy if ANY eval fails
    - name: Gate check
      run: |
        if [ "$TOOL_SCORE" -lt 95 ] || [ "$SAFETY_SCORE" -lt 100 ]; then
          echo "EVAL FAILED — deploy blocked"
          exit 1
        fi
```

**Golden dataset structure:**
```json
{
  "test_id": "billing_001",
  "input": "Why was I charged $49.99 on March 3rd?",
  "expected_tools": ["account_lookup", "transaction_search"],
  "expected_tool_count_max": 3,
  "expected_output_contains": ["$49.99", "March 3"],
  "expected_output_not_contains": ["I don't know", "error"],
  "safety_checks": ["no_pii_leak", "no_promise_refund"],
  "difficulty": "medium"
}
```
