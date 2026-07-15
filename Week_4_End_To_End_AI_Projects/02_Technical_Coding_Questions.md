# 💻 Week 4 — Technical / Coding Questions

> **Focus:** Conversation memory, Streamlit session state, tool orchestration, export systems, deployment configs
>
> **How to use:** Set a 20-minute timer per question. Build it before reading the solution.

---

## Q1. Build a Conversation Memory Manager with Token Budget ⭐⭐⭐

**Prompt:** Implement a `ConversationMemory` class that stores conversation turns, enforces a token budget, and supports multiple trimming strategies (sliding window, summary, and oldest-first).

**Solution:**

```python
from dataclasses import dataclass, field
from typing import Literal
from datetime import datetime

@dataclass
class Turn:
    role: str  # "user" or "assistant"
    content: str
    tool_used: str | None = None
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    token_estimate: int = 0
    
    def __post_init__(self):
        if not self.token_estimate:
            self.token_estimate = len(self.content) // 4  # ~4 chars per token

class ConversationMemory:
    """
    Production conversation memory with token budget enforcement.
    """
    
    def __init__(
        self,
        max_tokens: int = 4000,
        strategy: Literal["sliding_window", "oldest_first", "summary"] = "oldest_first",
        keep_system: bool = True,
    ):
        self.turns: list[Turn] = []
        self.max_tokens = max_tokens
        self.strategy = strategy
        self.keep_system = keep_system
        self.summary: str = ""
    
    def add(self, role: str, content: str, tool_used: str | None = None):
        turn = Turn(role=role, content=content, tool_used=tool_used)
        self.turns.append(turn)
        self._enforce_budget()
    
    def _enforce_budget(self):
        while self._total_tokens() > self.max_tokens and len(self.turns) > 2:
            if self.strategy == "sliding_window":
                self.turns.pop(0)
            elif self.strategy == "oldest_first":
                # Keep first turn (often important) and drop second-oldest
                if len(self.turns) > 3:
                    self.turns.pop(1)
                else:
                    self.turns.pop(0)
            elif self.strategy == "summary":
                self._summarize_old_turns()
                break
    
    def _summarize_old_turns(self):
        """Compress old turns into a summary, keep recent ones."""
        if len(self.turns) <= 4:
            return
        
        old_turns = self.turns[:-4]
        old_text = "\n".join([f"{t.role}: {t.content}" for t in old_turns])
        
        # In production: call LLM to summarize
        # For now: simple truncation
        self.summary = f"[Previous context: {old_text[:500]}...]"
        self.turns = self.turns[-4:]
    
    def _total_tokens(self) -> int:
        return sum(t.token_estimate for t in self.turns) + len(self.summary) // 4
    
    def get_messages(self) -> list[dict]:
        """Format for LLM API consumption."""
        messages = []
        
        if self.summary:
            messages.append({"role": "system", "content": f"Conversation context: {self.summary}"})
        
        for turn in self.turns:
            messages.append({"role": turn.role, "content": turn.content})
        
        return messages
    
    def get_recent(self, n: int = 5) -> list[Turn]:
        """Get last N turns for context injection."""
        return self.turns[-n:]
    
    def get_stats(self) -> dict:
        return {
            "total_turns": len(self.turns),
            "total_tokens": self._total_tokens(),
            "budget_remaining": self.max_tokens - self._total_tokens(),
            "has_summary": bool(self.summary),
            "tools_used": [t.tool_used for t in self.turns if t.tool_used],
        }
    
    def export_json(self) -> dict:
        return {
            "turns": [
                {
                    "role": t.role,
                    "content": t.content,
                    "tool_used": t.tool_used,
                    "timestamp": t.timestamp,
                    "tokens": t.token_estimate,
                }
                for t in self.turns
            ],
            "summary": self.summary,
            "stats": self.get_stats(),
        }
    
    def clear(self):
        self.turns = []
        self.summary = ""


# Usage
memory = ConversationMemory(max_tokens=2000, strategy="oldest_first")
memory.add("user", "What's the weather in Paris?")
memory.add("assistant", "It's 18°C and sunny in Paris.", tool_used="weather")
memory.add("user", "What about London?")
memory.add("assistant", "It's 14°C and cloudy in London.", tool_used="weather")

# Get messages formatted for LLM
messages = memory.get_messages()
# Get recent turns for context injection
recent = memory.get_recent(n=3)
```

---

## Q2. Build a Streamlit Chat Interface with Session State ⭐⭐

**Prompt:** Build a Streamlit chat UI that displays conversation history, handles user input, shows which tool was used per response, and persists across re-runs.

**Solution:**

```python
import streamlit as st
from conversation_memory import ConversationMemory

st.set_page_config(page_title="AI Assistant", page_icon="🤖", layout="wide")
st.title("🤖 AI Assistant")

# Initialize session state (persists across Streamlit re-runs)
if "memory" not in st.session_state:
    st.session_state.memory = ConversationMemory(max_tokens=4000)
if "processing" not in st.session_state:
    st.session_state.processing = False

# Sidebar — tool usage stats
with st.sidebar:
    st.header("📊 Session Stats")
    stats = st.session_state.memory.get_stats()
    st.metric("Total Turns", stats["total_turns"])
    st.metric("Tokens Used", f"{stats['total_tokens']}/{st.session_state.memory.max_tokens}")
    
    tool_counts = {}
    for tool in stats["tools_used"]:
        if tool:
            tool_counts[tool] = tool_counts.get(tool, 0) + 1
    if tool_counts:
        st.subheader("Tools Used")
        for tool, count in tool_counts.items():
            st.write(f"{'🧮' if tool == 'calculator' else '🌤️' if tool == 'weather' else '📖'} {tool}: {count}")
    
    if st.button("🗑️ Clear Conversation"):
        st.session_state.memory.clear()
        st.rerun()
    
    if st.button("📥 Export JSON"):
        import json
        data = st.session_state.memory.export_json()
        st.download_button(
            "Download", json.dumps(data, indent=2),
            file_name="conversation.json", mime="application/json"
        )

# Display conversation history
for turn in st.session_state.memory.turns:
    with st.chat_message(turn.role):
        st.write(turn.content)
        if turn.tool_used:
            st.caption(f"🔧 Used: {turn.tool_used}")

# User input
if prompt := st.chat_input("Ask me anything..."):
    # Display user message
    with st.chat_message("user"):
        st.write(prompt)
    st.session_state.memory.add("user", prompt)
    
    # Process and display response
    with st.chat_message("assistant"):
        with st.spinner("Thinking..."):
            # In production: call your AI pipeline here
            result = process_query(prompt, st.session_state.memory.get_recent())
        
        st.write(result["response"])
        if result.get("tool_used"):
            st.caption(f"🔧 Used: {result['tool_used']}")
    
    st.session_state.memory.add(
        "assistant", result["response"], tool_used=result.get("tool_used")
    )
```

---

## Q3. Build a Context-Aware Router That Handles Follow-Up Questions ⭐⭐⭐

**Prompt:** The user asks "What's the weather in Paris?" then "What about London?" — the router must understand the second query is also a weather request.

**Solution:**

```python
from pydantic import BaseModel, Field
from typing import Optional

class RoutingDecision(BaseModel):
    tool: str = Field(description="Tool to use")
    arguments: dict = Field(description="Arguments for the tool")
    reasoning: str = Field(description="Why this tool was selected")

def build_context_aware_prompt(
    user_message: str,
    recent_turns: list[dict],
    available_tools: list[dict],
) -> str:
    # Format recent conversation for context
    history = ""
    if recent_turns:
        history = "Recent conversation:\n"
        for turn in recent_turns[-5:]:
            history += f"  {turn['role']}: {turn['content']}\n"
            if turn.get('tool_used'):
                history += f"  [used tool: {turn['tool_used']}]\n"
        history += "\n"
    
    tool_descriptions = "\n".join([
        f"- {t['name']}: {t['description']}" for t in available_tools
    ])
    
    return f"""{history}Available tools:
{tool_descriptions}

Current user message: "{user_message}"

IMPORTANT: If the user's message is a follow-up (e.g., "what about London?" after a weather query), 
use the conversation context to determine the intended tool and fill in missing parameters.

Select the best tool and extract all parameters. Return JSON:
{{"tool": "tool_name", "arguments": {{}}, "reasoning": "..."}}"""


def route_with_context(
    user_message: str,
    memory: 'ConversationMemory',
    tools: list[dict],
) -> RoutingDecision:
    recent = [
        {"role": t.role, "content": t.content, "tool_used": t.tool_used}
        for t in memory.get_recent(5)
    ]
    
    prompt = build_context_aware_prompt(user_message, recent, tools)
    
    # Call LLM for routing
    raw = llm.generate(prompt, temperature=0)
    
    # Parse and validate
    decision = RoutingDecision.model_validate_json(raw)
    return decision

# Test scenario:
# Turn 1: "What's the weather in Paris?" → routes to weather(city="Paris")
# Turn 2: "What about London?" → context injection → routes to weather(city="London")
# Turn 3: "Convert that to Celsius" → context injection → routes to converter(value=14, from="F", to="C")
```

---

## Q4. Build a JSON Conversation Exporter with Analytics ⭐⭐

**Prompt:** Export a full conversation session as structured JSON including per-turn analytics, tool usage breakdown, cost estimates, and session metadata.

**Solution:**

```python
import json
from datetime import datetime
from dataclasses import dataclass

@dataclass
class SessionExporter:
    session_id: str
    started_at: str
    memory: 'ConversationMemory'
    model: str = "gpt-4o-mini"
    
    # Pricing per 1M tokens
    INPUT_COST = 0.15
    OUTPUT_COST = 0.60
    
    def export(self) -> dict:
        turns = []
        total_input_tokens = 0
        total_output_tokens = 0
        tool_usage = {}
        errors = 0
        
        for i, turn in enumerate(self.memory.turns):
            turn_data = {
                "turn_number": i + 1,
                "role": turn.role,
                "content": turn.content,
                "tool_used": turn.tool_used,
                "tokens": turn.token_estimate,
                "timestamp": turn.timestamp,
            }
            turns.append(turn_data)
            
            if turn.role == "user":
                total_input_tokens += turn.token_estimate
            else:
                total_output_tokens += turn.token_estimate
            
            if turn.tool_used:
                tool_usage[turn.tool_used] = tool_usage.get(turn.tool_used, 0) + 1
        
        input_cost = (total_input_tokens / 1_000_000) * self.INPUT_COST
        output_cost = (total_output_tokens / 1_000_000) * self.OUTPUT_COST
        
        return {
            "metadata": {
                "session_id": self.session_id,
                "model": self.model,
                "started_at": self.started_at,
                "exported_at": datetime.now().isoformat(),
                "total_turns": len(turns),
                "duration_turns": len([t for t in turns if t["role"] == "user"]),
            },
            "turns": turns,
            "analytics": {
                "total_input_tokens": total_input_tokens,
                "total_output_tokens": total_output_tokens,
                "total_tokens": total_input_tokens + total_output_tokens,
                "estimated_cost_usd": round(input_cost + output_cost, 6),
                "tool_usage": tool_usage,
                "most_used_tool": max(tool_usage, key=tool_usage.get) if tool_usage else None,
                "avg_tokens_per_turn": (total_input_tokens + total_output_tokens) // max(len(turns), 1),
            },
        }
    
    def export_to_file(self, filepath: str):
        with open(filepath, "w") as f:
            json.dump(self.export(), f, indent=2)
```

---

## Q5. Build a Tool Registry with Auto-Discovery ⭐⭐⭐

**Prompt:** Create a tool registry that supports decorator-based registration so adding a new tool is just adding one function — no editing the router.

**Solution:**

```python
from typing import Callable, Any
from pydantic import BaseModel
import inspect

class ToolRegistry:
    _instance = None
    _tools: dict[str, dict] = {}
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._tools = {}
        return cls._instance
    
    @classmethod
    def register(cls, name: str, description: str, schema: dict | None = None):
        """Decorator for registering tools."""
        def decorator(func: Callable) -> Callable:
            # Auto-generate schema from type hints if not provided
            if schema is None:
                sig = inspect.signature(func)
                auto_schema = {}
                for param_name, param in sig.parameters.items():
                    if param.annotation != inspect.Parameter.empty:
                        auto_schema[param_name] = {
                            "type": param.annotation.__name__,
                            "required": param.default == inspect.Parameter.empty,
                        }
            else:
                auto_schema = schema
            
            cls._tools[name] = {
                "func": func,
                "description": description,
                "schema": auto_schema,
                "name": name,
            }
            return func
        return decorator
    
    @classmethod
    def get_tool(cls, name: str) -> dict | None:
        return cls._tools.get(name)
    
    @classmethod
    def execute(cls, name: str, **kwargs) -> Any:
        tool = cls._tools.get(name)
        if not tool:
            raise ValueError(f"Unknown tool: {name}")
        return tool["func"](**kwargs)
    
    @classmethod
    def get_descriptions_for_llm(cls) -> str:
        lines = []
        for name, tool in cls._tools.items():
            params = ", ".join([f"{k}: {v['type']}" for k, v in tool["schema"].items()])
            lines.append(f"- {name}({params}): {tool['description']}")
        return "\n".join(lines)
    
    @classmethod
    def list_tools(cls) -> list[str]:
        return list(cls._tools.keys())


# Tool definitions — each in its own file
# tools/calculator.py
@ToolRegistry.register("calculator", "Evaluate mathematical expressions")
def calculator(expression: str) -> float:
    import ast
    return float(eval(compile(ast.parse(expression, mode='eval'), '<string>', 'eval')))

# tools/weather.py
@ToolRegistry.register("weather", "Get current weather for a city")
def weather(city: str) -> dict:
    import requests
    resp = requests.get(f"https://api.weather.com/current?city={city}")
    return resp.json()

# tools/wikipedia.py
@ToolRegistry.register("wikipedia", "Get a Wikipedia summary for a topic")
def wikipedia(topic: str) -> str:
    import requests
    resp = requests.get(f"https://en.wikipedia.org/api/rest_v1/page/summary/{topic}")
    return resp.json().get("extract", "No article found.")

# tools/converter.py
@ToolRegistry.register("unit_converter", "Convert between units of measurement")
def unit_converter(value: float, from_unit: str, to_unit: str) -> float:
    conversions = {
        ("fahrenheit", "celsius"): lambda v: (v - 32) * 5/9,
        ("celsius", "fahrenheit"): lambda v: v * 9/5 + 32,
        ("miles", "kilometers"): lambda v: v * 1.60934,
        ("kilometers", "miles"): lambda v: v / 1.60934,
        ("pounds", "kilograms"): lambda v: v * 0.453592,
        ("kilograms", "pounds"): lambda v: v / 0.453592,
    }
    key = (from_unit.lower(), to_unit.lower())
    if key not in conversions:
        raise ValueError(f"Unsupported conversion: {from_unit} → {to_unit}")
    return round(conversions[key](value), 4)


# Usage — router doesn't need to know about individual tools
print(ToolRegistry.get_descriptions_for_llm())
result = ToolRegistry.execute("calculator", expression="25 * 4")
```

---

## Q6. Build a Deployment Health Check System ⭐⭐

**Prompt:** Create a `/health` endpoint that checks all dependencies (LLM API, database, tools) and returns structured health status.

**Solution:**

```python
from fastapi import FastAPI
from datetime import datetime
import asyncio
import httpx

app = FastAPI()

async def check_llm_health() -> dict:
    try:
        start = datetime.now()
        async with httpx.AsyncClient(timeout=5) as client:
            resp = await client.post(
                "https://api.openai.com/v1/chat/completions",
                headers={"Authorization": f"Bearer {API_KEY}"},
                json={"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "ping"}], "max_tokens": 1},
            )
        latency = (datetime.now() - start).total_seconds() * 1000
        return {"status": "healthy", "latency_ms": round(latency), "model": "gpt-4o-mini"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

async def check_weather_api() -> dict:
    try:
        async with httpx.AsyncClient(timeout=5) as client:
            resp = await client.get("https://api.openweathermap.org/data/2.5/weather?q=London&appid=KEY")
        return {"status": "healthy" if resp.status_code == 200 else "degraded"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

async def check_database() -> dict:
    try:
        # Attempt a simple query
        result = db.execute("SELECT 1")
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

@app.get("/health")
async def health_check():
    checks = await asyncio.gather(
        check_llm_health(),
        check_weather_api(),
        check_database(),
        return_exceptions=True,
    )
    
    results = {
        "llm": checks[0] if not isinstance(checks[0], Exception) else {"status": "error"},
        "weather_api": checks[1] if not isinstance(checks[1], Exception) else {"status": "error"},
        "database": checks[2] if not isinstance(checks[2], Exception) else {"status": "error"},
    }
    
    overall = "healthy" if all(r.get("status") == "healthy" for r in results.values()) else "degraded"
    
    return {
        "status": overall,
        "timestamp": datetime.now().isoformat(),
        "checks": results,
    }
```

---

## Q7-Q16: Additional Coding Challenges (Condensed)

### Q7. Build a Streamlit Multi-Page App with Shared State ⭐⭐
Create a multi-page Streamlit app (Chat, Analytics, Settings) where conversation state persists across pages. Key: use `st.session_state` as shared state, `st.page_link()` for navigation.

### Q8. Implement Streaming Response Display in Streamlit ⭐⭐⭐
Display LLM tokens as they generate in the Streamlit chat interface. Key: use `st.write_stream()` or manually update a `st.empty()` placeholder in a loop consuming an async generator.

### Q9. Build a Conversation Analytics Dashboard ⭐⭐
From exported JSON conversations, build a dashboard showing: turns per session, tool usage distribution, average response time, cost per session, most common query types. Key: `pandas` + `st.bar_chart` / `plotly`.

### Q10. Implement a Dockerfile for an AI App ⭐⭐
Write a multi-stage Dockerfile for a Streamlit + FastAPI app. Key: separate builder (dependencies) and runtime stages, copy only `requirements.txt` first (layer caching), run as non-root user, health check, expose correct ports.

### Q11. Build a Rate Limiter for a Deployed AI App ⭐⭐⭐
Per-user rate limiting using `st.session_state` or Redis. Track requests per minute, show "slow down" warning when approaching limit, hard block at limit. Key: sliding window counter per session/user.

### Q12. Implement Conversation Branching (Fork a Conversation) ⭐⭐⭐⭐
Allow users to "branch" a conversation — go back to turn 3, ask a different question, and maintain both branches. Key: tree data structure for conversation history, not a linear list.

### Q13. Build a Tool Execution Timeout Wrapper ⭐⭐
Wrap tool calls in a timeout so slow APIs don't freeze the UI. Key: `asyncio.wait_for(tool_call(), timeout=10)` with fallback message.

### Q14. Build an Auto-Save System for Streamlit Conversations ⭐⭐
Auto-save conversation to a local SQLite database every 5 turns. Allow "resume previous conversation" on app reload. Key: background save trigger, session ID persistence, load on startup.

### Q15. Build a CI/CD Pipeline Config for an AI App ⭐⭐⭐
Write a GitHub Actions workflow that: runs tests, checks code formatting, validates Pydantic schemas, runs a small eval suite, and deploys to Streamlit Cloud on merge. Key: separate jobs for test/lint/eval/deploy.

### Q16. Build a Feature Flag System for AI Capabilities ⭐⭐
Toggle AI features (memory on/off, specific tools enabled/disabled, model selection) via a config file or env variables. Key: `FeatureFlags` class that reads from config, checked at each decision point.
