# 💻 Week 9 — Technical / Coding Questions

> **Focus:** Build ReAct from scratch, tool selection, multi-agent coordination, HIL, circuit breakers, cost guards, LangGraph state machines, safety layers
>
> **How to use:** These are the live-code challenges for senior AI engineer roles. Build before reading. Every solution here addresses BOTH functionality AND enterprise concerns (cost, safety, debugging).

---

## Q1. Build a ReAct Agent From Scratch (No Framework) ⭐⭐⭐⭐

**Prompt:** Implement a complete ReAct agent with iteration limits, cost tracking, tool call history, and loop detection. No LangChain, no LangGraph — raw Python + LLM SDK.

**Solution:**

```python
from dataclasses import dataclass, field
from typing import Callable, Any
import json
import re

@dataclass
class ToolCall:
    tool_name: str
    arguments: dict
    result: Any
    error: str | None = None
    cost_usd: float = 0.0

@dataclass
class Step:
    thought: str
    tool_call: ToolCall | None = None
    final_answer: str | None = None

@dataclass
class AgentTrace:
    query: str
    steps: list[Step] = field(default_factory=list)
    final_answer: str | None = None
    total_cost_usd: float = 0.0
    total_llm_calls: int = 0
    terminated_reason: str = "completed"

class ReActAgent:
    """
    Enterprise-ready ReAct agent from scratch.
    Includes: iteration limits, cost caps, loop detection, structured tracing.
    """
    
    def __init__(
        self,
        llm_fn: Callable,
        tools: dict[str, Callable],
        tool_descriptions: dict[str, str],
        max_iterations: int = 10,
        max_cost_usd: float = 1.0,
        max_duplicate_calls: int = 2,
    ):
        self.llm_fn = llm_fn
        self.tools = tools
        self.tool_descriptions = tool_descriptions
        self.max_iterations = max_iterations
        self.max_cost_usd = max_cost_usd
        self.max_duplicate_calls = max_duplicate_calls
    
    def run(self, query: str) -> AgentTrace:
        trace = AgentTrace(query=query)
        history_str = ""  # Accumulated reasoning + observations
        
        for iteration in range(self.max_iterations):
            # Cost guard — check before each LLM call
            if trace.total_cost_usd >= self.max_cost_usd:
                trace.terminated_reason = "cost_limit_exceeded"
                break
            
            # Build prompt with history
            prompt = self._build_prompt(query, history_str)
            
            # LLM call
            response, cost = self._call_llm(prompt)
            trace.total_llm_calls += 1
            trace.total_cost_usd += cost
            
            # Parse response
            step = self._parse_response(response)
            
            # Check for final answer
            if step.final_answer:
                trace.steps.append(step)
                trace.final_answer = step.final_answer
                trace.terminated_reason = "completed"
                break
            
            # Loop detection — check for duplicate tool calls
            if step.tool_call and self._is_duplicate(step.tool_call, trace):
                trace.terminated_reason = "loop_detected"
                trace.steps.append(step)
                break
            
            # Execute tool
            if step.tool_call:
                tool_result = self._execute_tool(step.tool_call)
                step.tool_call = tool_result
                trace.steps.append(step)
                
                # Append to history for next iteration
                history_str += f"\nThought: {step.thought}"
                history_str += f"\nAction: {tool_result.tool_name}"
                history_str += f"\nAction Input: {json.dumps(tool_result.arguments)}"
                history_str += f"\nObservation: {json.dumps(tool_result.result) if tool_result.result else tool_result.error}"
        else:
            trace.terminated_reason = "max_iterations_reached"
        
        return trace
    
    def _build_prompt(self, query: str, history: str) -> str:
        tool_list = "\n".join([
            f"- {name}: {desc}" for name, desc in self.tool_descriptions.items()
        ])
        
        return f"""You are an AI assistant that answers questions using tools.

Available tools:
{tool_list}

Use this format:
Thought: [your reasoning about what to do next]
Action: [tool_name]
Action Input: {{"arg1": "value1"}}

OR if you have the answer:
Thought: I now have enough information.
Final Answer: [your final answer]

Question: {query}

{history}

Thought:"""
    
    def _parse_response(self, response: str) -> Step:
        # Extract thought
        thought_match = re.search(
            r"Thought:\s*(.+?)(?=\n(?:Action:|Final Answer:)|\Z)",
            response, re.DOTALL
        )
        thought = thought_match.group(1).strip() if thought_match else ""
        
        # Check for final answer
        final_match = re.search(r"Final Answer:\s*(.+?)$", response, re.DOTALL)
        if final_match:
            return Step(thought=thought, final_answer=final_match.group(1).strip())
        
        # Extract action + input
        action_match = re.search(r"Action:\s*(\w+)", response)
        input_match = re.search(
            r"Action Input:\s*(\{.+?\})",
            response, re.DOTALL
        )
        
        if action_match and input_match:
            try:
                arguments = json.loads(input_match.group(1))
            except json.JSONDecodeError:
                arguments = {"raw": input_match.group(1)}
            
            tool_call = ToolCall(
                tool_name=action_match.group(1).strip(),
                arguments=arguments,
                result=None,
            )
            return Step(thought=thought, tool_call=tool_call)
        
        # Neither final answer nor valid tool call — malformed response
        return Step(thought=thought, final_answer="[PARSE_ERROR] " + response[:200])
    
    def _is_duplicate(self, new_call: ToolCall, trace: AgentTrace) -> bool:
        """Detect if agent is stuck calling same tool with same args."""
        duplicates = 0
        for step in trace.steps:
            if step.tool_call and step.tool_call.tool_name == new_call.tool_name:
                if step.tool_call.arguments == new_call.arguments:
                    duplicates += 1
        return duplicates >= self.max_duplicate_calls
    
    def _execute_tool(self, tool_call: ToolCall) -> ToolCall:
        if tool_call.tool_name not in self.tools:
            tool_call.error = f"Tool '{tool_call.tool_name}' not found"
            return tool_call
        
        try:
            result = self.tools[tool_call.tool_name](**tool_call.arguments)
            tool_call.result = result
        except Exception as e:
            tool_call.error = f"{type(e).__name__}: {str(e)}"
        
        return tool_call
    
    def _call_llm(self, prompt: str) -> tuple[str, float]:
        """Call LLM and estimate cost. Replace with your provider's SDK."""
        # Placeholder: real implementation calls OpenAI/Anthropic/etc.
        response = self.llm_fn(prompt)
        estimated_cost = len(prompt) * 0.00001  # Placeholder cost
        return response, estimated_cost


# Usage
def search(query: str) -> str:
    return f"Results for '{query}': ..."

def calculate(expression: str) -> str:
    return str(eval(expression))

agent = ReActAgent(
    llm_fn=your_llm_function,
    tools={"search": search, "calculate": calculate},
    tool_descriptions={
        "search": "Search the web. Args: query (str)",
        "calculate": "Evaluate math. Args: expression (str)",
    },
    max_iterations=10,
    max_cost_usd=0.50,
)

trace = agent.run("What is the population of Tokyo divided by that of Osaka?")
print(f"Answer: {trace.final_answer}")
print(f"Cost: ${trace.total_cost_usd:.4f}")
print(f"Terminated: {trace.terminated_reason}")
```

**Enterprise features demonstrated:**
- Iteration limit (prevents infinite loops)
- Cost cap (prevents runaway spend)
- Loop detection (catches doom loops)
- Structured tracing (debug-friendly)
- Graceful error handling (agent doesn't crash on tool failures)

---

## Q2. Build a Multi-Agent Coordinator Pattern ⭐⭐⭐⭐

**Prompt:** Implement a coordinator agent that delegates tasks to specialist agents. Handle: task routing, result aggregation, error recovery.

**Solution:**

```python
from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class AgentResult:
    agent_name: str
    task: str
    result: Any
    success: bool
    cost_usd: float
    error: str | None = None

class Specialist:
    """A specialist agent with a specific role."""
    def __init__(self, name: str, role: str, agent_fn: Callable):
        self.name = name
        self.role = role
        self.agent_fn = agent_fn
    
    def execute(self, task: str) -> AgentResult:
        try:
            result, cost = self.agent_fn(task)
            return AgentResult(
                agent_name=self.name,
                task=task,
                result=result,
                success=True,
                cost_usd=cost,
            )
        except Exception as e:
            return AgentResult(
                agent_name=self.name,
                task=task,
                result=None,
                success=False,
                cost_usd=0.0,
                error=str(e),
            )

class CoordinatorAgent:
    """
    Central coordinator that routes tasks to specialists.
    Aggregates results, handles failures, tracks total cost.
    """
    
    def __init__(
        self,
        llm_fn: Callable,
        specialists: list[Specialist],
        max_total_cost_usd: float = 5.0,
    ):
        self.llm_fn = llm_fn
        self.specialists = {s.name: s for s in specialists}
        self.max_total_cost_usd = max_total_cost_usd
    
    def execute(self, user_request: str) -> dict:
        # Step 1: Plan — coordinator decides which specialists to invoke
        plan = self._plan_execution(user_request)
        
        results = []
        total_cost = 0.0
        
        # Step 2: Execute plan (possibly parallel)
        for task_spec in plan["tasks"]:
            if total_cost >= self.max_total_cost_usd:
                results.append(AgentResult(
                    agent_name=task_spec["specialist"],
                    task=task_spec["task"],
                    result=None,
                    success=False,
                    cost_usd=0.0,
                    error="cost_limit_exceeded",
                ))
                continue
            
            specialist = self.specialists.get(task_spec["specialist"])
            if not specialist:
                results.append(AgentResult(
                    agent_name=task_spec["specialist"],
                    task=task_spec["task"],
                    result=None,
                    success=False,
                    cost_usd=0.0,
                    error=f"unknown_specialist",
                ))
                continue
            
            result = specialist.execute(task_spec["task"])
            results.append(result)
            total_cost += result.cost_usd
        
        # Step 3: Synthesize — combine specialist results
        final_answer = self._synthesize(user_request, results)
        
        return {
            "user_request": user_request,
            "plan": plan,
            "specialist_results": results,
            "final_answer": final_answer,
            "total_cost_usd": total_cost,
            "specialists_invoked": len([r for r in results if r.success]),
            "failures": len([r for r in results if not r.success]),
        }
    
    def _plan_execution(self, user_request: str) -> dict:
        """Coordinator uses LLM to decide which specialists to invoke."""
        specialist_desc = "\n".join([
            f"- {name}: {s.role}" for name, s in self.specialists.items()
        ])
        
        prompt = f"""You are a coordinator that delegates tasks to specialists.

Available specialists:
{specialist_desc}

User request: {user_request}

Decide which specialists to invoke and what specific task to give each.
Respond in JSON:
{{
    "tasks": [
        {{"specialist": "name", "task": "specific task description"}}
    ]
}}"""
        
        response = self.llm_fn(prompt)
        import json
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"tasks": []}
    
    def _synthesize(self, request: str, results: list[AgentResult]) -> str:
        """Combine specialist outputs into final answer."""
        results_summary = "\n".join([
            f"[{r.agent_name}] {r.result if r.success else 'FAILED: ' + r.error}"
            for r in results
        ])
        
        prompt = f"""Synthesize these specialist results into a final answer.

Original request: {request}

Specialist results:
{results_summary}

Provide a coherent final answer to the user's request."""
        
        return self.llm_fn(prompt)


# Usage
research_agent = Specialist(
    name="researcher",
    role="Gather information from web and databases",
    agent_fn=lambda task: (f"Research on {task}", 0.05),
)

analyst_agent = Specialist(
    name="analyst",
    role="Analyze data and identify patterns",
    agent_fn=lambda task: (f"Analysis of {task}", 0.10),
)

writer_agent = Specialist(
    name="writer",
    role="Write formal reports and communications",
    agent_fn=lambda task: (f"Written output for {task}", 0.03),
)

coordinator = CoordinatorAgent(
    llm_fn=your_llm,
    specialists=[research_agent, analyst_agent, writer_agent],
    max_total_cost_usd=2.0,
)

result = coordinator.execute("Analyze the smartphone market and write a report")
```

**Enterprise features:**
- Cost cap across entire multi-agent execution
- Isolated specialists (failure of one doesn't crash others)
- Explicit planning (auditable what coordinator decided)
- Structured results for debugging

---

## Q3. Build a Human-in-the-Loop Agent with LangGraph ⭐⭐⭐⭐

**Prompt:** Implement an agent that pauses for human approval before destructive actions. Include: approval gates, timeout handling, resume from checkpoint.

**Solution:**

```python
from typing import TypedDict, Literal, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

class AgentState(TypedDict):
    user_request: str
    plan: list[dict]
    current_step: int
    step_results: list[dict]
    pending_approval: dict | None
    approval_received: bool
    approval_decision: Literal["approved", "rejected", "modified"] | None
    approval_feedback: str
    final_result: str | None
    error: str | None

class HILAgent:
    """
    Agent with human-in-the-loop for destructive actions.
    Uses LangGraph's native interrupt-and-resume via checkpointing.
    """
    
    # Actions that ALWAYS require human approval
    DESTRUCTIVE_ACTIONS = {
        "send_email",
        "delete_record",
        "make_payment",
        "post_to_social",
        "commit_to_repo",
    }
    
    def __init__(self, llm_fn, tools, checkpoint_conn_str):
        self.llm_fn = llm_fn
        self.tools = tools
        self.checkpointer = PostgresSaver.from_conn_string(checkpoint_conn_str)
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        workflow = StateGraph(AgentState)
        
        workflow.add_node("plan", self.plan_node)
        workflow.add_node("check_approval", self.check_approval_node)
        workflow.add_node("execute_step", self.execute_step_node)
        workflow.add_node("await_approval", self.await_approval_node)
        workflow.add_node("apply_feedback", self.apply_feedback_node)
        workflow.add_node("finalize", self.finalize_node)
        
        workflow.set_entry_point("plan")
        workflow.add_edge("plan", "check_approval")
        
        workflow.add_conditional_edges(
            "check_approval",
            self.route_after_approval_check,
            {
                "needs_approval": "await_approval",
                "auto_execute": "execute_step",
                "done": "finalize",
            }
        )
        
        workflow.add_conditional_edges(
            "await_approval",
            self.route_after_approval,
            {
                "approved": "execute_step",
                "rejected": "finalize",
                "modified": "apply_feedback",
                "waiting": END,  # Pause graph — human hasn't responded yet
            }
        )
        
        workflow.add_edge("apply_feedback", "check_approval")
        workflow.add_edge("execute_step", "check_approval")
        workflow.add_edge("finalize", END)
        
        return workflow.compile(checkpointer=self.checkpointer)
    
    def plan_node(self, state: AgentState) -> AgentState:
        """Generate execution plan."""
        prompt = f"""Break down this request into steps.
Each step has an action and arguments.
Mark destructive steps (send email, delete, payment) explicitly.

Request: {state['user_request']}

Return JSON: {{"steps": [{{"action": "...", "args": {{...}}, "destructive": bool}}]}}"""
        
        import json
        plan = json.loads(self.llm_fn(prompt))
        return {"plan": plan["steps"], "current_step": 0, "step_results": []}
    
    def check_approval_node(self, state: AgentState) -> AgentState:
        """Check if next step needs human approval."""
        if state["current_step"] >= len(state["plan"]):
            return {"pending_approval": None}
        
        current = state["plan"][state["current_step"]]
        
        if current.get("destructive") or current["action"] in self.DESTRUCTIVE_ACTIONS:
            return {"pending_approval": current}
        
        return {"pending_approval": None}
    
    def route_after_approval_check(self, state: AgentState) -> str:
        if state["current_step"] >= len(state["plan"]):
            return "done"
        if state["pending_approval"]:
            return "needs_approval"
        return "auto_execute"
    
    def await_approval_node(self, state: AgentState) -> AgentState:
        """Pause execution and notify human."""
        if state["approval_received"]:
            # Human has responded, continue
            return {}
        
        # Post approval request to human (Slack, dashboard, etc.)
        self._notify_human(state["pending_approval"])
        
        # Graph will pause here — resumed externally when human decides
        return {"approval_received": False}
    
    def route_after_approval(self, state: AgentState) -> str:
        if not state["approval_received"]:
            return "waiting"
        
        return state["approval_decision"]
    
    def execute_step_node(self, state: AgentState) -> AgentState:
        """Execute the current step."""
        current = state["plan"][state["current_step"]]
        
        try:
            tool = self.tools.get(current["action"])
            if not tool:
                raise ValueError(f"Unknown tool: {current['action']}")
            
            result = tool(**current["args"])
            
            return {
                "step_results": state["step_results"] + [{
                    "step": state["current_step"],
                    "action": current["action"],
                    "result": result,
                    "success": True,
                }],
                "current_step": state["current_step"] + 1,
                "approval_received": False,
                "approval_decision": None,
            }
        except Exception as e:
            return {"error": str(e)}
    
    def apply_feedback_node(self, state: AgentState) -> AgentState:
        """Human modified the plan — adjust and re-check."""
        # Use human feedback to modify the current step
        current = state["plan"][state["current_step"]]
        prompt = f"""User modified the plan. Update the step.

Original step: {current}
User feedback: {state['approval_feedback']}

Return updated step JSON."""
        
        import json
        updated = json.loads(self.llm_fn(prompt))
        new_plan = state["plan"][:]
        new_plan[state["current_step"]] = updated
        
        return {
            "plan": new_plan,
            "approval_received": False,
            "approval_decision": None,
        }
    
    def finalize_node(self, state: AgentState) -> AgentState:
        summary = f"Executed {len(state['step_results'])} steps"
        return {"final_result": summary}
    
    def _notify_human(self, pending_action: dict):
        """Notify human of pending approval (Slack, email, etc.)."""
        # In production: post to Slack, send email, update dashboard
        print(f"⚠️  Approval needed: {pending_action}")
    
    # Public API
    def start(self, user_request: str, thread_id: str) -> dict:
        """Start a new agent execution."""
        config = {"configurable": {"thread_id": thread_id}}
        return self.graph.invoke({"user_request": user_request}, config=config)
    
    def approve(self, thread_id: str, decision: str, feedback: str = "") -> dict:
        """Human provides approval decision — resumes graph."""
        config = {"configurable": {"thread_id": thread_id}}
        return self.graph.invoke(
            {
                "approval_received": True,
                "approval_decision": decision,
                "approval_feedback": feedback,
            },
            config=config,
        )
```

**Enterprise features demonstrated:**
- Native checkpointing (pause/resume across long time periods)
- Explicit destructive action list (auditable)
- Approval, rejection, AND modification support (not just yes/no)
- Full state serialization for compliance

---

## Q4. Build an Agent Cost Guard and Circuit Breaker ⭐⭐⭐⭐

**Prompt:** Implement cost tracking + circuit breakers for a production agent system. Multi-tenant. Enforce per-user, per-tenant, and per-agent limits.

**Solution:**

```python
import time
from dataclasses import dataclass, field
from collections import defaultdict, deque
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"   # Normal operation
    OPEN = "open"       # Blocking calls
    HALF_OPEN = "half_open"  # Testing recovery

@dataclass
class CircuitBreaker:
    name: str
    failure_threshold: int = 5
    reset_timeout_seconds: int = 60
    failures: deque = field(default_factory=lambda: deque(maxlen=100))
    state: CircuitState = CircuitState.CLOSED
    opened_at: float = 0
    
    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
            self.failures.clear()
    
    def record_failure(self):
        self.failures.append(time.time())
        # Check if we've hit threshold in the last minute
        cutoff = time.time() - self.reset_timeout_seconds
        recent = [f for f in self.failures if f > cutoff]
        
        if len(recent) >= self.failure_threshold:
            self.state = CircuitState.OPEN
            self.opened_at = time.time()
    
    def is_available(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            if time.time() - self.opened_at > self.reset_timeout_seconds:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        
        # HALF_OPEN — allow one call to test
        return True

@dataclass
class CostTracker:
    user_id: str
    tenant_id: str
    daily_spend: float = 0.0
    monthly_spend: float = 0.0
    last_reset_day: str = ""

class AgentCostGuard:
    """
    Multi-tenant cost enforcement for agents.
    Tracks per-user, per-tenant, per-execution spending.
    """
    
    def __init__(
        self,
        user_daily_limit: float = 10.0,
        tenant_monthly_limit: float = 1000.0,
        per_execution_limit: float = 1.0,
    ):
        self.user_daily_limit = user_daily_limit
        self.tenant_monthly_limit = tenant_monthly_limit
        self.per_execution_limit = per_execution_limit
        
        self.user_costs: dict[str, CostTracker] = {}
        self.tenant_costs: dict[str, float] = defaultdict(float)
        self.circuit_breakers: dict[str, CircuitBreaker] = {}
    
    def can_execute(
        self,
        user_id: str,
        tenant_id: str,
        estimated_cost: float,
    ) -> tuple[bool, str]:
        """Check all limits before agent execution."""
        # Check per-execution limit
        if estimated_cost > self.per_execution_limit:
            return False, f"Estimated cost ${estimated_cost:.2f} exceeds per-execution limit"
        
        # Check user daily limit
        user = self._get_user_tracker(user_id, tenant_id)
        if user.daily_spend + estimated_cost > self.user_daily_limit:
            return False, f"Would exceed user daily limit (${self.user_daily_limit})"
        
        # Check tenant monthly limit
        if self.tenant_costs[tenant_id] + estimated_cost > self.tenant_monthly_limit:
            return False, f"Would exceed tenant monthly limit (${self.tenant_monthly_limit})"
        
        # Check circuit breakers
        breaker_key = f"{tenant_id}:{user_id}"
        breaker = self.circuit_breakers.get(breaker_key)
        if breaker and not breaker.is_available():
            return False, "Circuit breaker open — recent failures"
        
        return True, "OK"
    
    def record_execution(
        self,
        user_id: str,
        tenant_id: str,
        actual_cost: float,
        success: bool,
    ):
        """Record actual cost and outcome."""
        user = self._get_user_tracker(user_id, tenant_id)
        user.daily_spend += actual_cost
        user.monthly_spend += actual_cost
        self.tenant_costs[tenant_id] += actual_cost
        
        # Update circuit breaker
        breaker_key = f"{tenant_id}:{user_id}"
        if breaker_key not in self.circuit_breakers:
            self.circuit_breakers[breaker_key] = CircuitBreaker(name=breaker_key)
        
        breaker = self.circuit_breakers[breaker_key]
        if success:
            breaker.record_success()
        else:
            breaker.record_failure()
    
    def _get_user_tracker(self, user_id: str, tenant_id: str) -> CostTracker:
        key = f"{tenant_id}:{user_id}"
        if key not in self.user_costs:
            self.user_costs[key] = CostTracker(user_id=user_id, tenant_id=tenant_id)
        
        # Reset daily counters if new day
        today = time.strftime("%Y-%m-%d")
        tracker = self.user_costs[key]
        if tracker.last_reset_day != today:
            tracker.daily_spend = 0.0
            tracker.last_reset_day = today
        
        return tracker


# Integration with agent
class SafeAgentWrapper:
    def __init__(self, agent, cost_guard: AgentCostGuard):
        self.agent = agent
        self.cost_guard = cost_guard
    
    def execute(self, user_id: str, tenant_id: str, query: str, estimated_cost: float = 0.10):
        # Pre-check
        can_run, reason = self.cost_guard.can_execute(user_id, tenant_id, estimated_cost)
        if not can_run:
            return {"error": reason, "blocked": True}
        
        # Execute agent
        try:
            result = self.agent.run(query)
            self.cost_guard.record_execution(
                user_id, tenant_id, result.total_cost_usd, success=True,
            )
            return result
        except Exception as e:
            self.cost_guard.record_execution(user_id, tenant_id, 0.0, success=False)
            return {"error": str(e), "blocked": False}
```

**Enterprise features:**
- Multi-tenant isolation (per-user + per-tenant limits)
- Circuit breakers on repeated failures (prevents bad-state cascades)
- Time-based reset windows (daily, monthly)
- Pre-execution AND post-execution tracking

---

## Q5. Build a Safe Browser Agent With Domain Allowlist ⭐⭐⭐⭐

**Prompt:** Implement a browser agent that can navigate the web safely. Enforce domain allowlists, block dangerous actions, require HIL for form submissions.

**Solution:**

```python
from urllib.parse import urlparse
from dataclasses import dataclass
from enum import Enum

class ActionRisk(Enum):
    LOW = "low"        # Read-only (view page, get text)
    MEDIUM = "medium"  # Navigate (click links, scroll)
    HIGH = "high"      # Submit forms, fill inputs
    CRITICAL = "critical"  # Payment, credentials, downloads

@dataclass
class SafetyPolicy:
    allowed_domains: set[str]  # Only these domains can be visited
    blocked_domains: set[str]  # Never visit these
    max_pages_per_session: int = 20
    max_forms_per_session: int = 3
    require_hil_for: set[ActionRisk] = None
    
    def __post_init__(self):
        if self.require_hil_for is None:
            self.require_hil_for = {ActionRisk.HIGH, ActionRisk.CRITICAL}

class SafeBrowserAgent:
    """
    Browser agent with layered safety controls.
    Domain allowlists, action-level HIL, session limits.
    """
    
    def __init__(
        self,
        llm_fn,
        browser,  # e.g., playwright browser instance
        policy: SafetyPolicy,
        hil_callback,  # function to request human approval
    ):
        self.llm_fn = llm_fn
        self.browser = browser
        self.policy = policy
        self.hil_callback = hil_callback
        
        self.pages_visited = 0
        self.forms_submitted = 0
        self.action_log = []
    
    def execute(self, task: str) -> dict:
        """Execute a browser task safely."""
        state = {
            "task": task,
            "current_url": None,
            "results": [],
            "errors": [],
        }
        
        # Agent loop
        for iteration in range(50):  # Hard iteration limit
            action = self._decide_action(state)
            
            if action["type"] == "done":
                state["final_answer"] = action.get("answer", "")
                break
            
            # SAFETY CHECK 1: URL allowlist
            if action["type"] == "navigate":
                url = action["url"]
                domain = urlparse(url).netloc
                
                if domain in self.policy.blocked_domains:
                    state["errors"].append(f"BLOCKED: {domain} is on blocklist")
                    continue
                
                if self.policy.allowed_domains and domain not in self.policy.allowed_domains:
                    state["errors"].append(f"BLOCKED: {domain} not in allowlist")
                    continue
                
                if self.pages_visited >= self.policy.max_pages_per_session:
                    state["errors"].append("BLOCKED: page limit reached")
                    break
            
            # SAFETY CHECK 2: Risk-based HIL
            risk = self._assess_risk(action)
            if risk in self.policy.require_hil_for:
                approved, feedback = self.hil_callback(action, risk)
                if not approved:
                    state["errors"].append(f"HIL rejected: {feedback}")
                    continue
            
            # SAFETY CHECK 3: Form submission limit
            if action["type"] == "submit_form":
                if self.forms_submitted >= self.policy.max_forms_per_session:
                    state["errors"].append("BLOCKED: form submission limit")
                    continue
                self.forms_submitted += 1
            
            # Execute action
            try:
                result = self._execute_browser_action(action)
                state["results"].append(result)
                self.action_log.append({
                    "iteration": iteration,
                    "action": action,
                    "risk": risk.value,
                    "result": result,
                })
                
                if action["type"] == "navigate":
                    self.pages_visited += 1
                    state["current_url"] = action["url"]
            except Exception as e:
                state["errors"].append(str(e))
        
        return state
    
    def _assess_risk(self, action: dict) -> ActionRisk:
        """Classify action risk level."""
        action_type = action["type"]
        
        if action_type in ("get_text", "get_html", "screenshot", "read_element"):
            return ActionRisk.LOW
        
        if action_type in ("click_link", "scroll", "navigate"):
            # But check if navigating to sensitive area
            url = action.get("url", "")
            if any(kw in url.lower() for kw in ("checkout", "payment", "billing", "login")):
                return ActionRisk.CRITICAL
            return ActionRisk.MEDIUM
        
        if action_type in ("submit_form", "fill_input", "click_button"):
            return ActionRisk.HIGH
        
        if action_type in ("download_file", "upload_file", "enter_credentials"):
            return ActionRisk.CRITICAL
        
        return ActionRisk.HIGH  # Default: assume risky
    
    def _decide_action(self, state: dict) -> dict:
        """LLM decides next action."""
        # In production: format state, call LLM, parse response
        # Placeholder logic
        return {"type": "done", "answer": "task completed"}
    
    def _execute_browser_action(self, action: dict):
        """Execute the browser action via Playwright/Selenium/etc."""
        # Placeholder
        return {"success": True}


# Usage
policy = SafetyPolicy(
    allowed_domains={"example.com", "docs.example.com", "api.example.com"},
    blocked_domains={"malicious.com"},
    max_pages_per_session=15,
    max_forms_per_session=2,
    require_hil_for={ActionRisk.HIGH, ActionRisk.CRITICAL},
)

def approval_ui(action, risk):
    # In production: notify via Slack, wait for response
    print(f"Approval needed for {risk.value} action: {action}")
    return True, "approved"

agent = SafeBrowserAgent(
    llm_fn=your_llm,
    browser=playwright_browser,
    policy=policy,
    hil_callback=approval_ui,
)
```

**Enterprise features:**
- Domain allowlists (agent can't be redirected to attacker sites)
- Risk-based approval (auto for reads, HIL for writes)
- Session-level budgets (page limit, form limit)
- Full audit log of every action + risk assessment

---

## Q6-Q16: Additional Coding Challenges (Condensed)

### Q6. Build a Plan-and-Execute Agent ⭐⭐⭐
Generate full plan upfront (LLM call 1). Execute each step (parallel where possible). Re-plan if step fails. Contrast with ReAct: less flexible, more predictable, faster execution.

### Q7. Implement Reflexion Pattern ⭐⭐⭐⭐
Agent attempts task. On failure, generates reflection ("what went wrong, what to do differently"). Reflection injected as context for next attempt. Track: attempt count, per-attempt cost, whether reflection helped.

### Q8. Build a Tool Selection Accuracy Evaluator ⭐⭐⭐
For 100 golden queries with known-correct tool, run agent, check if right tool was called. Report per-tool precision/recall. Identify weak tools (frequent mis-selection). Regenerate descriptions for weak tools.

### Q9. Implement Agent Trajectory Logger ⭐⭐⭐
Log every decision: thought, action, tool, args, result, cost, latency. Structured JSON per step. Enable trajectory analysis, replay, debugging. Include trace_id for distributed tracing correlation.

### Q10. Build a Multi-Agent Message Bus ⭐⭐⭐⭐
Agents publish/subscribe to topics. Coordinator subscribes to results. Workers subscribe to work assignments. Message: sender, recipient, task, priority, timeout. Handle: agent failures, message loss, ordering.

### Q11. Implement Dynamic Tool Registration ⭐⭐⭐
Agent discovers tools at runtime via MCP or registration API. Tools have permissions attached. Agent's tool list is filtered by user permissions. Handle: tool version changes, deprecated tools, new tools mid-session.

### Q12. Build a Loop Detection System ⭐⭐⭐
Detect: repeated tool calls with same args (exact loops), semantic loops (different phrasings, same intent), meta-loops (thinking about the same thing repeatedly). Break loops by forcing strategy change or terminating.

### Q13. Implement Compensating Transactions for Agents ⭐⭐⭐⭐
Track all reversible actions. On failure/abort, execute compensating action in reverse order. Example: agent created 3 tickets, 2 emails, 1 slack message → compensate = delete ticket 3, unsend emails (if within window), delete slack message.

### Q14. Build an Agent A/B Testing Framework ⭐⭐⭐
Compare two agent implementations on same task. Metrics: success rate, cost, latency, trajectory quality. Sticky user assignment. Statistical significance testing. Report: which agent is better on which dimensions.

### Q15. Implement Prompt Injection Defense for Agents ⭐⭐⭐⭐
Sanitize tool outputs before injecting into agent context. Detect injection patterns (instruction-like content in data). Use structural delimiters (XML tags) to separate data from instructions. Post-generation validation: agent didn't leak system prompt.

### Q16. Build a Real-Time Agent Monitoring Dashboard ⭐⭐⭐
Live metrics: active agents, avg latency, success rate, cost/hour, error rate. Per-agent traces. Alert triggers: cost spike, error rate spike, safety violation. Integration with PagerDuty/Slack.
