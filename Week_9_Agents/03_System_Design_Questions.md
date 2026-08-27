# 🏗️ Week 9 — System Design Questions

> **Focus:** Enterprise agent platform, browser agent security, coding agent architecture, multi-agent orchestration, production observability, safety systems
>
> **How to use:** 45-60 min whiteboard rounds. These are staff+ questions where the interviewer expects you to draw architecture, discuss trade-offs, and dive into critical components — especially safety, cost, and observability.

---

## Q1. Design an Enterprise Agent Platform Serving 100+ Internal Teams ⭐⭐⭐⭐

**Prompt:** "Your company has 100+ internal teams. Many want to build AI agents. Design a platform that lets them build safely, tracks costs, enforces governance, and doesn't require each team to reinvent basics."

**Architecture:**

```
┌───────────────── ENTERPRISE AGENT PLATFORM ─────────────────────┐
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           AGENT REGISTRY                                     │ │
│  │  All agents registered with metadata:                        │ │
│  │  - Team ownership                                            │ │
│  │  - Use case, risk classification                             │ │
│  │  - Tool permissions                                          │ │
│  │  - Cost budgets                                              │ │
│  │  - Compliance category (internal/customer-facing/regulated)  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │  TOOL CATALOG                                                │ │
│  │  Central registry of approved tools:                         │ │
│  │  - Enterprise data connectors (Salesforce, Slack, Jira)      │ │
│  │  - Internal APIs                                             │ │
│  │  - Public APIs (vetted)                                      │ │
│  │  Every tool has: permissions, cost model, docs, examples     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │  RUNTIME PLATFORM                                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │ Agent        │  │ Cost Guard   │  │ Circuit      │     │ │
│  │  │ Executor     │  │              │  │ Breakers     │     │ │
│  │  │              │  │ Per-team,    │  │              │     │ │
│  │  │ Runs agent   │  │ per-user,    │  │ Cost, error, │     │ │
│  │  │ code in      │  │ per-execution│  │ latency      │     │ │
│  │  │ sandboxed    │  │ limits       │  │ triggers     │     │ │
│  │  │ environment  │  │              │  │              │     │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │ │
│  │                                                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │ HIL Gateway  │  │ Model Router │  │ Tool Gateway │     │ │
│  │  │              │  │              │  │              │     │ │
│  │  │ Approval     │  │ Chooses LLM  │  │ Enforces     │     │ │
│  │  │ requests →   │  │ per task     │  │ tool         │     │ │
│  │  │ Slack        │  │ (cost/       │  │ permissions  │     │ │
│  │  │              │  │ capability)  │  │              │     │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │  OBSERVABILITY                                               │ │
│  │  - Full agent traces (LangSmith / Langfuse)                  │ │
│  │  - Cost dashboards per team                                  │ │
│  │  - Safety incidents                                          │ │
│  │  - Trajectory quality metrics                                │ │
│  │  - Alert routing (Slack, PagerDuty)                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│  ┌───────────────────────────┼───────────────────────────────┐  │
│  │  GOVERNANCE LAYER                                            │ │
│  │  - Agent approval workflow (before deploying to prod)        │ │
│  │  - Risk assessments                                          │ │
│  │  - Audit logs (immutable, cryptographically signed)          │ │
│  │  - Compliance reports (SOC 2, EU AI Act)                     │ │
│  │  - Incident response procedures                              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

**1. Tool catalog is central, agents are decentralized.**
Teams build their own agents. They compose from the SHARED tool catalog. This prevents "everyone builds a Salesforce integration" while enabling team autonomy.

**2. Tiered risk classification.**
- **Green (internal, read-only):** Fast approval, high autonomy
- **Yellow (internal, writes):** Standard approval, HIL for destructive
- **Red (customer-facing / regulated):** Rigorous approval, mandatory HIL, incident response

Every agent gets classified before deployment.

**3. Cost governance from day 1.**
Every team has a budget. Agents can't exceed. Cost tracked per agent, per user, per execution. Weekly cost reports to each team.

**4. Central observability, team-scoped dashboards.**
Platform team runs central Langfuse/LangSmith. Each team sees only their own agents' traces. Cross-agent analytics available to platform team.

**5. Approval workflow before production.**
Deployment gate: security review + compliance review + performance test. Only approved agents run in prod.

**6. Kill switches at three levels.**
- Per-execution (interrupt current agent)
- Per-agent (disable specific agent)
- Platform-wide (emergency shutdown of all agents)

Every agent developer knows the kill switch exists.

**Interview signal:** Discussing "risk tiers with different approval paths" shows enterprise operational thinking, not just technical.

---

## Q2. Design a Browser Agent System With Enterprise Security ⭐⭐⭐⭐

**Prompt:** "Design a browser agent system for enterprise. Users authenticate via SSO. Agent can navigate web on user's behalf. Security-first design."

**Threat model — what can go wrong:**

1. Prompt injection via malicious web content
2. Credential theft (agent authenticates on hostile sites)
3. CSRF-like attacks (agent has authenticated sessions)
4. Data exfiltration (agent uploads confidential data)
5. Autonomous financial actions (unauthorized purchases)
6. Impersonation (agent posts as the user)

**Architecture:**

```
┌────────────── ISOLATED BROWSER SANDBOX ──────────────┐
│                                                        │
│  Container (per-user, disposable)                     │
│  ├── Playwright/Chromium (no persistent storage)      │
│  ├── No file system access (except designated dir)    │
│  ├── Network filtered via proxy                       │
│  └── Memory limits, CPU limits                        │
│                                                        │
└───────────────────────┬──────────────────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │  NETWORK PROXY                  │
        │  - Domain allowlists per task   │
        │  - DNS filtering                │
        │  - Traffic logging              │
        │  - Content sanitization         │
        └───────────────┬────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │  AGENT CONTROLLER               │
        │  - LLM makes decisions          │
        │  - Actions validated pre-exec   │
        │  - Risk-based HIL routing       │
        │  - Full audit trail             │
        └───────────────┬────────────────┘
                        │
    ┌───────────────────┼─────────────────┐
    ▼                   ▼                 ▼
┌──────────┐  ┌────────────────┐  ┌──────────────┐
│ SSO      │  │ Credential     │  │ Data         │
│ (OAuth)  │  │ Vault          │  │ Boundary     │
│          │  │                │  │              │
│ Never    │  │ Per-domain     │  │ What data    │
│ shares   │  │ credential     │  │ can leave    │
│ tokens   │  │ isolation      │  │ enterprise?  │
│ with     │  │                │  │              │
│ agent    │  │                │  │              │
└──────────┘  └────────────────┘  └──────────────┘
```

**Security controls:**

**1. Per-execution sandbox.**
Fresh Chromium container per task. No persistent storage. Destroyed after execution. Prevents cross-task contamination.

**2. Network filtering.**
Domain allowlist per task (defined by user or config). Reject navigation to unlisted domains. DNS filtering blocks known malicious hosts.

**3. Credential isolation.**
Never share user's actual credentials with agent. Use OAuth tokens with limited scope. Time-bounded tokens. Per-domain credential access.

**4. Data boundary enforcement.**
Classify data by sensitivity. Agent can only see data appropriate for the task. Upload attempts to external sites logged and reviewed.

**5. Content sanitization.**
Web content passed to LLM is sanitized: HTML stripped, potential prompt injections detected, delimiters clear (this is DATA, not INSTRUCTIONS).

**6. Risk-based HIL.**
- Read-only actions: auto
- Form fills (non-sensitive): auto with logging
- Login attempts: HIL
- Financial transactions: MANDATORY HIL
- Downloads to enterprise: HIL
- Uploads from enterprise: HIL

**7. Audit everything.**
Every URL visited, every click, every form filled, every data touched. Immutable audit log. Retained per compliance requirements.

**8. Timeout enforcement.**
Task timeout (10 min default). Session timeout (1 hour). Idle timeout (5 min). Prevents runaway agents.

**9. Kill switch.**
User can abort any agent execution. Immediate termination. Cleanup of sandbox.

**Compliance considerations:**

- **SOC 2:** Audit logs, access controls, incident response
- **GDPR:** User consent for agent actions, data minimization, right to review agent decisions
- **EU AI Act:** Browser agent likely falls under "high-risk" — need risk assessment, monitoring, human oversight

**Interview signal:** Discussing "network filtering + credential isolation + risk-based HIL" as layered defense shows security-first design.

---

## Q3. Design a Coding Agent Architecture (Devin/Cursor-style) ⭐⭐⭐⭐

**Prompt:** "Design a coding agent that can implement features, run tests, and submit PRs. Must be safe (won't break production), efficient (respects developer time), and integrable with existing dev workflow."

**Architecture:**

```
┌──── CODING AGENT PLATFORM ─────────────────────┐
│                                                  │
│  User Request: "Implement feature X"             │
│              │                                   │
│              ▼                                   │
│  ┌──────────────────────────────┐              │
│  │  PLANNING LAYER              │              │
│  │  1. Read codebase context    │              │
│  │  2. Understand feature       │              │
│  │  3. Generate implementation  │              │
│  │     plan                     │              │
│  │  4. Estimate scope, cost     │              │
│  └────────────┬─────────────────┘              │
│               │                                  │
│               ▼                                  │
│  ┌──────────────────────────────┐              │
│  │  HUMAN APPROVAL              │              │
│  │  Show plan to user           │              │
│  │  User approves/modifies      │              │
│  └────────────┬─────────────────┘              │
│               │                                  │
│               ▼                                  │
│  ┌──────────────────────────────┐              │
│  │  ISOLATED DEV ENVIRONMENT    │              │
│  │  ├── Docker container         │              │
│  │  ├── Fresh git branch         │              │
│  │  ├── Access to codebase       │              │
│  │  ├── Language servers         │              │
│  │  ├── Test runners             │              │
│  │  └── NO production access     │              │
│  └────────────┬─────────────────┘              │
│               │                                  │
│               ▼                                  │
│  ┌──────────────────────────────┐              │
│  │  EXECUTION LOOP              │              │
│  │  ┌───────────────────────┐   │              │
│  │  │ Write code           │   │              │
│  │  │       ↓              │   │              │
│  │  │ Run tests            │   │              │
│  │  │       ↓              │   │              │
│  │  │ Fix errors           │   │              │
│  │  │       ↓              │   │              │
│  │  │ Iterate until pass   │   │              │
│  │  └───────────────────────┘   │              │
│  │  Max iterations: 30           │              │
│  │  Max cost: $5.00              │              │
│  │  Max time: 1 hour             │              │
│  └────────────┬─────────────────┘              │
│               │                                  │
│               ▼                                  │
│  ┌──────────────────────────────┐              │
│  │  QUALITY GATES               │              │
│  │  - All tests pass?           │              │
│  │  - Lint clean?               │              │
│  │  - No security issues?       │              │
│  │  - Coverage acceptable?      │              │
│  └────────────┬─────────────────┘              │
│               │                                  │
│               ▼                                  │
│  ┌──────────────────────────────┐              │
│  │  PR CREATION                  │              │
│  │  - Push branch                │              │
│  │  - Create PR with description │              │
│  │  - @-mention user for review  │              │
│  │  - Human REQUIRED to merge   │              │
│  └──────────────────────────────┘              │
└─────────────────────────────────────────────────┘
```

**Key design decisions:**

**1. Never execute in production.**
Agent operates in isolated Docker container. Reads codebase, writes to feature branch. No access to production DBs, servers, or credentials.

**2. Test-driven execution.**
Agent's success signal is passing tests. Ensures functional correctness (not just "looks right"). Test failures become the reflection signal for retries.

**3. Bounded execution.**
Hard limits on: iterations, cost, wall-clock time. Prevents runaway coding sessions.

**4. Human required for merge.**
Agent creates PR. Human reviews. Never auto-merges. Prevents catastrophic changes making it to main branch.

**5. Explicit tool set.**
Tools available: `read_file`, `write_file`, `run_tests`, `run_lint`, `git_diff`, `search_code`. NOT `deploy`, `run_production`, `delete_file` (dangerous ops require HIL).

**6. Codebase context management.**
Coding agents need context (functions, types, existing patterns). Vector search over codebase, symbol indexing (LSP), plus targeted file reads. Balance: too little context = wrong code; too much = expensive.

**7. Rollback ready.**
All work on branch. If agent produces bad code, delete branch. No damage.

**8. Cost transparency.**
Show developer estimated cost before starting ("This task will use ~$1.50 in agent time"). Show actual cost after ("Used $1.42 across 15 iterations").

**9. Failure escalation.**
If agent can't complete after N iterations, produce partial work + explanation. Human can pick up where agent left off.

**Enterprise considerations:**

- **Access control:** Agent should only see codebases the user has access to
- **Audit:** Every code change, test run, tool call logged
- **Compliance:** For regulated industries, agent-generated code needs identifiable provenance
- **Model choice:** For code, prefer stronger models (Claude Sonnet, GPT-4o). Cost worth it.

**Interview signal:** Discussing "test-driven execution loop" as core mechanic shows understanding of what makes coding agents work.

---

## Q4. Design a Multi-Agent Customer Support Orchestration System ⭐⭐⭐⭐

**Prompt:** "Design a multi-agent system for enterprise customer support. Routes tickets to specialists. Handles escalation. Learns from human interactions."

**Architecture:**

```
User submits ticket
       │
       ▼
┌──────────────────────────┐
│  Triage Agent            │
│  Classifies ticket:      │
│  - Category (billing,    │
│    technical, general)   │
│  - Priority (urgency)    │
│  - Sentiment             │
│  - Language              │
└──────────┬───────────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Billing │  │Tech    │  │Product │  │Escalat.│
│Agent   │  │Agent   │  │Agent   │  │Agent   │
│        │  │        │  │        │  │        │
│Refunds,│  │Errors, │  │Feature │  │Human   │
│invoices│  │setup,  │  │questns │  │handoff │
│        │  │bugs    │  │        │  │        │
└───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
    │           │           │           │
    ▼           ▼           ▼           ▼
┌────────────────────────────────────────┐
│  Response Composer Agent                │
│  Combines specialist inputs into        │
│  polished response                      │
└──────────────┬─────────────────────────┘
               │
               ▼
┌────────────────────────────────────────┐
│  Quality Check                          │
│  - Faithfulness (grounded in policy?)  │
│  - Tone (matches brand voice?)         │
│  - Completeness (addresses question?)  │
│  - Safety (no promises we can't keep?) │
└──────────────┬─────────────────────────┘
               │
       Pass  ┌─┴─┐  Fail
             │   │
             ▼   ▼
       Send to  Human review
       user     required
```

**Key design decisions:**

**1. Triage before specialization.**
Cheap triage agent classifies before invoking expensive specialists. Prevents wrong specialists getting invoked.

**2. Specialists have narrow tools.**
Billing agent has access to: invoices, refund tool, subscription tool. NOT: user data, internal systems, other tenants. Least privilege.

**3. Escalation is first-class.**
Not a failure mode — a designed path. Complex tickets, angry customers, or novel cases → immediate escalation. Explicit handoff with context.

**4. Response composer separates concerns.**
Specialists produce content. Composer handles tone, format, brand voice. Keeps specialists focused on correctness.

**5. Quality gates before sending.**
Every response passes: faithfulness check (matches policy?), safety check (no unauthorized promises?), tone check. Failures → human review.

**6. Learning loop.**
Human corrections feed back into:
- Golden dataset (train future evals)
- Prompt improvements (agents get better)
- Policy updates (surface unclear policies)

**Enterprise concerns:**

- **Liability:** AI-made promises are contractually binding in many jurisdictions. Quality gates critical.
- **Consistency:** Different agents shouldn't give contradictory answers to similar questions. Central knowledge base.
- **Cost:** Multi-agent per ticket = 3-6x more expensive than single agent. Justified only if quality is meaningfully better.
- **Compliance:** GDPR (customer data), audit trail (regulated industries), disclosure (users know AI is involved).

---

## Q5. Design an Agent Safety and Kill Switch System ⭐⭐⭐⭐

**Prompt:** "Design the safety infrastructure that prevents agent-caused incidents. Include: permission systems, kill switches, incident response."

**Layered defense architecture:**

```
Layer 1: PRE-EXECUTION VALIDATION
├── User authorized to invoke this agent?
├── Cost budget available?
├── Rate limit not exceeded?
├── Agent not in blocked state?
└── Task doesn't match blocked patterns?

    ↓ (all pass)

Layer 2: EXECUTION SANDBOXING
├── Isolated environment (container/namespace)
├── Network filtering
├── File system restrictions
├── Resource limits (CPU, memory, time)
└── No default access to sensitive systems

    ↓

Layer 3: PER-ACTION AUTHORIZATION
├── Every tool call checks permissions
├── Destructive actions require HIL
├── Novel action patterns → alert
├── Data access classified & logged
└── Timeout on individual actions

    ↓

Layer 4: MONITORING & DETECTION
├── Real-time cost tracking
├── Behavior anomaly detection
├── Loop detection
├── Safety violation patterns
└── User complaint signal

    ↓ (if anomaly detected)

Layer 5: KILL SWITCHES
├── Per-execution: interrupt current agent
├── Per-user: block all agents for this user
├── Per-agent-type: disable all instances of X
├── Platform-wide: emergency full shutdown
└── Automatic triggers on severity
```

**Kill switch specifics:**

**Per-execution kill switch.**
Any running agent can be halted. Trigger: user click, timeout, cost overrun, safety violation.

```python
# Simplified implementation
class AgentExecutor:
    def __init__(self):
        self.kill_signals = {}  # execution_id -> should_halt
    
    def check_kill(self, execution_id):
        return self.kill_signals.get(execution_id, False)
    
    def request_kill(self, execution_id, reason):
        self.kill_signals[execution_id] = True
        audit_log.record("kill_requested", execution_id, reason)
    
    # Agent code checks this between every step
```

**Per-agent kill switch.**
Disable an agent type across all instances. Trigger: safety incident, quality regression, security vulnerability.

**Platform-wide kill switch.**
Halt all agents everywhere. Trigger: major incident, active attack, data breach investigation. Requires: authenticated exec approval + auditable trigger.

**Automatic triggers:**

- Cost anomaly (10x normal) → auto-kill affected executions
- Cross-tenant access attempt → auto-kill + escalate
- Safety violation → auto-kill + notify security team
- Loop detected (>50 iterations without progress) → auto-kill

**Incident response flow:**

```
Alert fires (safety violation on agent X)
    ↓
Auto-response:
├── Kill affected executions
├── Disable agent X globally
├── Preserve state for forensics
├── Notify security + agent owner
└── Log incident
    ↓
Human response (within 15 min):
├── Assess scope (how many users affected?)
├── Determine severity
├── Communicate (Slack, exec if severe)
└── Begin RCA
    ↓
Resolution:
├── Fix root cause
├── Test fix
├── Redeploy with verification
├── Post-mortem
└── Documentation update
```

**Interview signal:** Discussing "layered defense with 5 layers, each catching different failure modes" shows security-mature thinking.

---

## Q6-Q16: Additional System Design Questions (Condensed)

### Q6. Design an Agent Cost Attribution System ⭐⭐⭐
Every agent execution tagged with: team, user, agent type, feature. Aggregate hourly. Dashboards per team. Chargeback billing. Anomaly alerts (team's cost 5x normal). Weekly cost reports auto-generated.

### Q7. Design a Multi-Agent Debugging Platform ⭐⭐⭐⭐
Full trace of multi-agent conversation. Message flow visualization. Root cause finder: given a failed task, which agent's decision started the failure cascade? Time-travel debugging: replay from any point with modified state.

### Q8. Design an Agent Evaluation Infrastructure ⭐⭐⭐⭐
Golden task datasets per agent type. Trajectory analysis (not just output). Continuous eval in CI/CD. Regression detection. A/B testing framework. Cost per successful completion tracked over time. Regression alerts before deploy.

### Q9. Design an Autonomous Research Agent Platform ⭐⭐⭐⭐
Long-running (hours to days). Goal-directed with sub-goal generation. Periodic milestones for human review. Data persistence across sessions. Cost budget: $50-500 per research task. Output: structured research report with citations, methodology, confidence levels.

### Q10. Design an Agent-to-Agent Communication Protocol ⭐⭐⭐
Standard message format across frameworks (potential MCP-like standard). Message: sender, recipient, task, context, response_format, priority, timeout. Transport-agnostic (HTTP, WebSocket, message queue). Enterprise concern: audit-log every inter-agent message.

### Q11. Design a Human-Agent Handoff System ⭐⭐⭐
Agents work autonomously until human intervention needed. Handoff to human includes: full context, agent's understanding, blocking issue, suggested actions. Human works, returns to agent when done. Agent picks up with human's outputs. Bidirectional collaboration.

### Q12. Design an Agent Testing Framework (Beyond Standard Testing) ⭐⭐⭐
Adversarial tests (jailbreaks, prompt injections). Trajectory tests (was the path optimal?). Cost tests (agent within budget?). Safety tests (no unauthorized actions?). Chaos tests (tools fail randomly, does agent recover?). Long-running tests (24-hour marathon).

### Q13. Design a Regulated-Industry Agent System (Healthcare/Finance) ⭐⭐⭐⭐
Full audit trail (7-year retention). Deterministic tool responses (no non-deterministic paths). HIPAA/PCI compliance for data. Model card + agent card documentation. Regulatory review before deployment. Every decision explainable. Human oversight required for patient/customer-facing actions.

### Q14. Design an Agent Marketplace for Enterprise ⭐⭐⭐
Curated catalog of pre-built agents. Approval process for marketplace listing. Version management. Rating/review from other teams. Cost transparency (avg cost per execution). Rollback (bad agent version reverted).

### Q15. Design an Agent Rate Limiter and Fair Queuing System ⭐⭐⭐
Per-user rate limits, per-tenant quotas, per-agent-type limits. Priority queuing (interactive requests vs batch). Fair share (no user monopolizes resources). Backpressure signaling (client knows when to retry). Circuit breakers on downstream services.

### Q16. Design an Explainability Layer for Agent Decisions ⭐⭐⭐⭐
For regulated/high-stakes decisions, agent must explain why. Capture: reasoning chain, evidence used, tools called, alternative considered. Present to user in accessible format. Enable "why did the agent decide X?" queries. Critical for EU AI Act right-to-explanation.
