# 💻 Week 3 — Technical / Coding Questions

> **Focus:** Tool calling implementation, agent loops, validation pipelines, structured output parsing
>
> **How to use:** These are hands-on coding challenges. Build them before reading solutions.

---

## Q1. Build a Complete Tool-Calling Agent Loop ⭐⭐⭐

**Prompt:** Implement the full ReAct agent loop: receive user input → LLM decides tool → execute tool → validate result → feed back → repeat until done. Include loop detection and max iteration limits.

**Solution:**

```python
import json
from dataclasses import dataclass, field
from typing import Any, Callable
from pydantic import BaseModel

class ToolCall(BaseModel):
    tool_name: str
    arguments: dict[str, Any]

class ToolResult(BaseModel):
    success: bool
    data: Any = None
    error: str | None = None

@dataclass
class AgentConfig:
    max_iterations: int = 5
    max_retries_per_tool: int = 2
    max_duplicate_calls: int = 2

class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Callable] = {}
        self._schemas: dict[str, dict] = {}
    
    def register(self, name: str, func: Callable, schema: dict):
        self._tools[name] = func
        self._schemas[name] = schema
    
    def execute(self, tool_call: ToolCall) -> ToolResult:
        if tool_call.tool_name not in self._tools:
            return ToolResult(success=False, error=f"Unknown tool: {tool_call.tool_name}")
        try:
            result = self._tools[tool_call.tool_name](**tool_call.arguments)
            return ToolResult(success=True, data=result)
        except Exception as e:
            return ToolResult(success=False, error=str(e))
    
    def get_descriptions(self) -> list[dict]:
        return [{"name": name, "schema": schema} for name, schema in self._schemas.items()]

class AgentLoop:
    """
    Production-grade ReAct agent loop with:
    - Max iteration limits
    - Duplicate call detection
    - Per-tool retry logic
    - Structured logging of every step
    """
    
    def __init__(self, tools: ToolRegistry, config: AgentConfig = None):
        self.tools = tools
        self.config = config or AgentConfig()
        self.call_history: list[str] = []
        self.execution_log: list[dict] = []
    
    def _detect_loop(self, tool_call: ToolCall) -> bool:
        """Check if we're calling the same tool with same args repeatedly."""
        signature = f"{tool_call.tool_name}:{json.dumps(tool_call.arguments, sort_keys=True)}"
        recent = self.call_history[-5:]
        duplicate_count = sum(1 for c in recent if c == signature)
        self.call_history.append(signature)
        return duplicate_count >= self.config.max_duplicate_calls
    
    def _simulate_llm_decision(self, messages: list[dict]) -> dict:
        """
        Simulate LLM deciding next action.
        In production: actual API call with tool definitions.
        Returns either a tool call or a final response.
        """
        # Placeholder — in production this is your LLM API call
        last_message = messages[-1]["content"]
        
        if "calculate" in last_message.lower():
            return {
                "type": "tool_call",
                "tool_name": "calculator",
                "arguments": {"expression": "2 + 2"},
            }
        return {"type": "response", "content": "Here's my answer based on the information gathered."}
    
    def run(self, user_input: str) -> dict:
        """
        Execute the full agent loop.
        
        Returns:
            {
                "response": str,
                "iterations_used": int,
                "tools_called": list,
                "execution_log": list,
            }
        """
        messages = [
            {"role": "system", "content": f"You have tools: {json.dumps(self.tools.get_descriptions())}"},
            {"role": "user", "content": user_input},
        ]
        
        tools_called = []
        
        for iteration in range(self.config.max_iterations):
            # Step 1: LLM decides next action
            decision = self._simulate_llm_decision(messages)
            
            self.execution_log.append({
                "iteration": iteration + 1,
                "decision_type": decision["type"],
            })
            
            # Step 2: If LLM wants to respond directly, we're done
            if decision["type"] == "response":
                return {
                    "response": decision["content"],
                    "iterations_used": iteration + 1,
                    "tools_called": tools_called,
                    "execution_log": self.execution_log,
                }
            
            # Step 3: Parse and validate tool call
            try:
                tool_call = ToolCall(
                    tool_name=decision["tool_name"],
                    arguments=decision["arguments"],
                )
            except Exception as e:
                messages.append({
                    "role": "assistant",
                    "content": f"I tried to call a tool but the format was wrong: {e}"
                })
                continue
            
            # Step 4: Loop detection
            if self._detect_loop(tool_call):
                messages.append({
                    "role": "system",
                    "content": "LOOP DETECTED. You are calling the same tool repeatedly. "
                               "Provide a final answer with the information you have."
                })
                self.execution_log.append({"warning": "loop_detected"})
                continue
            
            # Step 5: Execute tool
            result = self.tools.execute(tool_call)
            tools_called.append(tool_call.tool_name)
            
            self.execution_log.append({
                "tool": tool_call.tool_name,
                "arguments": tool_call.arguments,
                "success": result.success,
                "result_preview": str(result.data)[:100] if result.data else result.error,
            })
            
            # Step 6: Feed result back to LLM
            messages.append({
                "role": "tool",
                "content": json.dumps({
                    "tool": tool_call.tool_name,
                    "result": result.data if result.success else None,
                    "error": result.error,
                }),
            })
        
        # Hit max iterations
        return {
            "response": "I reached the maximum number of steps. Here's what I found so far.",
            "iterations_used": self.config.max_iterations,
            "tools_called": tools_called,
            "execution_log": self.execution_log,
            "warning": "max_iterations_reached",
        }


# Register tools
registry = ToolRegistry()
registry.register(
    "calculator",
    lambda expression: eval(expression),  # In production: use a safe math parser
    {"expression": {"type": "string", "description": "Math expression to evaluate"}},
)
registry.register(
    "weather",
    lambda city: {"temp": 72, "condition": "sunny", "city": city},
    {"city": {"type": "string", "description": "City name"}},
)

# Run agent
agent = AgentLoop(registry)
result = agent.run("What is 25 * 4?")
print(f"Response: {result['response']}")
print(f"Iterations: {result['iterations_used']}")
print(f"Tools used: {result['tools_called']}")
```

---

## Q2. Build a Pydantic Validation Pipeline for LLM Outputs ⭐⭐⭐

**Prompt:** Create a validation pipeline that takes raw LLM output, attempts to parse it into a schema, retries with error feedback if it fails, and falls back gracefully.

**Solution:**

```python
from pydantic import BaseModel, Field, ValidationError
from typing import TypeVar, Type
import json

T = TypeVar("T", bound=BaseModel)

class ValidationPipeline:
    """
    Multi-attempt LLM output validation with error feedback.
    
    Attempt 1: Parse raw output
    Attempt 2: Extract JSON from text, then parse
    Attempt 3: Re-prompt LLM with validation errors
    Attempt 4: Return fallback
    """
    
    def __init__(self, max_retries: int = 2):
        self.max_retries = max_retries
        self.validation_log: list[dict] = []
    
    def validate(
        self,
        raw_output: str,
        schema: Type[T],
        reprompt_fn: callable = None,
    ) -> T | None:
        """
        Attempt to validate LLM output against a Pydantic schema.
        """
        
        # Attempt 1: Direct parse
        try:
            result = schema.model_validate_json(raw_output.strip())
            self.validation_log.append({"attempt": 1, "status": "success"})
            return result
        except (ValidationError, json.JSONDecodeError) as e:
            self.validation_log.append({"attempt": 1, "status": "failed", "error": str(e)})
        
        # Attempt 2: Extract JSON from markdown/text
        from week2_json_extractor import extract_json  # Our Q3 from Week 2
        try:
            extracted = extract_json(raw_output)
            result = schema.model_validate(extracted)
            self.validation_log.append({"attempt": 2, "status": "success"})
            return result
        except (ValueError, ValidationError) as e:
            self.validation_log.append({"attempt": 2, "status": "failed", "error": str(e)})
        
        # Attempt 3+: Re-prompt with error feedback
        if reprompt_fn:
            for retry in range(self.max_retries):
                error_feedback = self._format_error_feedback(raw_output, schema)
                
                new_output = reprompt_fn(error_feedback)
                
                try:
                    result = schema.model_validate_json(new_output.strip())
                    self.validation_log.append({
                        "attempt": 3 + retry, "status": "success"
                    })
                    return result
                except (ValidationError, json.JSONDecodeError) as e:
                    raw_output = new_output  # Update for next iteration
                    self.validation_log.append({
                        "attempt": 3 + retry, "status": "failed", "error": str(e)
                    })
        
        # All attempts failed
        return None
    
    def _format_error_feedback(self, raw_output: str, schema: Type[BaseModel]) -> str:
        """Generate a clear error message for the LLM to fix its output."""
        schema_json = json.dumps(schema.model_json_schema(), indent=2)
        
        return (
            f"Your previous response was not valid JSON matching the required schema.\n\n"
            f"Your response was:\n{raw_output[:500]}\n\n"
            f"Required schema:\n{schema_json}\n\n"
            f"Please respond with ONLY valid JSON matching this schema. "
            f"No markdown, no explanation, no code blocks."
        )


# Example usage
class ToolDecision(BaseModel):
    tool_name: str = Field(description="Name of the tool to call")
    arguments: dict = Field(description="Tool arguments")
    reasoning: str = Field(description="Why this tool was selected", min_length=10)
    confidence: float = Field(ge=0, le=1)

pipeline = ValidationPipeline(max_retries=2)

# Test with various LLM outputs
test_outputs = [
    '{"tool_name": "calculator", "arguments": {"expr": "2+2"}, "reasoning": "User asked for math", "confidence": 0.95}',
    '```json\n{"tool_name": "weather", "arguments": {"city": "NYC"}, "reasoning": "User wants weather info", "confidence": 0.8}\n```',
    'I think we should use the calculator tool',  # This will fail
]

for output in test_outputs:
    result = pipeline.validate(output, ToolDecision)
    if result:
        print(f"✓ {result.tool_name} (confidence: {result.confidence})")
    else:
        print(f"✗ Failed to validate: {output[:50]}...")
```

---

## Q3. Implement a Tool Permission System ⭐⭐⭐

**Prompt:** Build a permission layer that controls which tools an agent can call based on user role, context, and risk level.

**Solution:**

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any

class RiskLevel(Enum):
    LOW = "low"           # Read-only operations
    MEDIUM = "medium"     # Write operations (create, update)
    HIGH = "high"         # Delete, financial, or irreversible operations
    CRITICAL = "critical" # Admin-level operations

class UserRole(Enum):
    ANONYMOUS = "anonymous"
    USER = "user"
    ADMIN = "admin"
    SYSTEM = "system"

@dataclass
class ToolPermission:
    tool_name: str
    risk_level: RiskLevel
    allowed_roles: set[UserRole]
    requires_confirmation: bool = False  # Human-in-the-loop for high-risk
    rate_limit_per_minute: int = 60
    max_calls_per_session: int = 100

class PermissionManager:
    def __init__(self):
        self._permissions: dict[str, ToolPermission] = {}
        self._call_counts: dict[str, dict[str, int]] = {}  # session_id → {tool → count}
    
    def register(self, permission: ToolPermission):
        self._permissions[permission.tool_name] = permission
    
    def check(
        self,
        tool_name: str,
        user_role: UserRole,
        session_id: str,
        arguments: dict[str, Any] = None,
    ) -> dict:
        """
        Check if a tool call is permitted.
        Returns: {"allowed": bool, "reason": str, "requires_confirmation": bool}
        """
        # Check tool exists
        if tool_name not in self._permissions:
            return {"allowed": False, "reason": f"Tool '{tool_name}' is not registered"}
        
        perm = self._permissions[tool_name]
        
        # Check role
        if user_role not in perm.allowed_roles:
            return {
                "allowed": False,
                "reason": f"Role '{user_role.value}' cannot use '{tool_name}'. "
                          f"Required: {[r.value for r in perm.allowed_roles]}",
            }
        
        # Check session rate limit
        session_counts = self._call_counts.setdefault(session_id, {})
        current_count = session_counts.get(tool_name, 0)
        
        if current_count >= perm.max_calls_per_session:
            return {
                "allowed": False,
                "reason": f"Session limit reached for '{tool_name}' "
                          f"({current_count}/{perm.max_calls_per_session})",
            }
        
        # Check for dangerous argument patterns
        if arguments:
            danger_check = self._check_argument_safety(tool_name, arguments)
            if not danger_check["safe"]:
                return {"allowed": False, "reason": danger_check["reason"]}
        
        # Track call
        session_counts[tool_name] = current_count + 1
        
        return {
            "allowed": True,
            "reason": "Permitted",
            "requires_confirmation": perm.requires_confirmation,
        }
    
    def _check_argument_safety(self, tool_name: str, arguments: dict) -> dict:
        """Check for dangerous patterns in tool arguments."""
        # Check for SQL injection patterns
        for key, value in arguments.items():
            if isinstance(value, str):
                dangerous_patterns = ["DROP", "DELETE", "--", ";", "UNION SELECT"]
                for pattern in dangerous_patterns:
                    if pattern.lower() in value.lower():
                        return {"safe": False, "reason": f"Dangerous pattern in argument '{key}'"}
        
        return {"safe": True}


# Setup permissions
pm = PermissionManager()

pm.register(ToolPermission(
    tool_name="web_search",
    risk_level=RiskLevel.LOW,
    allowed_roles={UserRole.USER, UserRole.ADMIN, UserRole.SYSTEM},
))

pm.register(ToolPermission(
    tool_name="send_email",
    risk_level=RiskLevel.HIGH,
    allowed_roles={UserRole.ADMIN},
    requires_confirmation=True,
    max_calls_per_session=5,
))

pm.register(ToolPermission(
    tool_name="delete_record",
    risk_level=RiskLevel.CRITICAL,
    allowed_roles={UserRole.ADMIN},
    requires_confirmation=True,
    max_calls_per_session=1,
))

# Test
print(pm.check("web_search", UserRole.USER, "session_1"))
# → {"allowed": True, ...}

print(pm.check("send_email", UserRole.USER, "session_1"))
# → {"allowed": False, "reason": "Role 'user' cannot use 'send_email'..."}

print(pm.check("delete_record", UserRole.ADMIN, "session_1"))
# → {"allowed": True, "requires_confirmation": True}
```

---

## Q4. Build a Tool Output Normalizer ⭐⭐

**Prompt:** Different tools return data in different formats. Build a normalizer that converts any tool output into a standardized format the LLM can consume consistently.

**Solution:**

```python
from dataclasses import dataclass
from typing import Any
from datetime import datetime
import json

@dataclass
class NormalizedToolOutput:
    """Standardized format for all tool outputs."""
    tool_name: str
    success: bool
    data_type: str             # "text", "number", "table", "list", "error"
    summary: str               # Human-readable one-line summary
    details: Any               # Full structured data
    confidence: float = 1.0    # How reliable is this data?
    timestamp: str = ""
    source: str = ""           # Attribution
    
    def to_llm_context(self) -> str:
        """Format for injection into LLM context."""
        if not self.success:
            return f"[Tool '{self.tool_name}' failed: {self.summary}]"
        
        return (
            f"[{self.tool_name} result]\n"
            f"Summary: {self.summary}\n"
            f"Data: {json.dumps(self.details) if not isinstance(self.details, str) else self.details}\n"
            f"Source: {self.source}\n"
            f"Confidence: {self.confidence}"
        )

class ToolOutputNormalizer:
    """Normalize diverse tool outputs into a consistent format."""
    
    def normalize(self, tool_name: str, raw_output: Any) -> NormalizedToolOutput:
        """Route to tool-specific normalizer."""
        normalizers = {
            "calculator": self._normalize_calculator,
            "weather": self._normalize_weather,
            "web_search": self._normalize_search,
            "database": self._normalize_database,
        }
        
        normalizer = normalizers.get(tool_name, self._normalize_generic)
        
        try:
            return normalizer(tool_name, raw_output)
        except Exception as e:
            return NormalizedToolOutput(
                tool_name=tool_name,
                success=False,
                data_type="error",
                summary=f"Failed to normalize output: {str(e)}",
                details={"raw": str(raw_output)[:500]},
                timestamp=datetime.now().isoformat(),
            )
    
    def _normalize_calculator(self, name: str, output: Any) -> NormalizedToolOutput:
        result = float(output) if not isinstance(output, (int, float)) else output
        return NormalizedToolOutput(
            tool_name=name,
            success=True,
            data_type="number",
            summary=f"Calculation result: {result}",
            details={"result": result},
            confidence=1.0,
            timestamp=datetime.now().isoformat(),
            source="local_calculator",
        )
    
    def _normalize_weather(self, name: str, output: dict) -> NormalizedToolOutput:
        temp = output.get("temperature", output.get("temp"))
        city = output.get("city", output.get("location", "Unknown"))
        condition = output.get("condition", output.get("weather", "Unknown"))
        
        return NormalizedToolOutput(
            tool_name=name,
            success=True,
            data_type="text",
            summary=f"Weather in {city}: {temp}°, {condition}",
            details={"temperature": temp, "city": city, "condition": condition},
            confidence=0.9,
            timestamp=datetime.now().isoformat(),
            source=output.get("source", "weather_api"),
        )
    
    def _normalize_search(self, name: str, output: Any) -> NormalizedToolOutput:
        if isinstance(output, list):
            results = output[:5]  # Cap at 5 results
            summary = f"Found {len(output)} results"
        elif isinstance(output, dict):
            results = output.get("results", [output])
            summary = output.get("summary", f"Search returned {len(results)} results")
        else:
            results = [{"text": str(output)}]
            summary = "Search returned 1 result"
        
        return NormalizedToolOutput(
            tool_name=name,
            success=True,
            data_type="list",
            summary=summary,
            details=results,
            confidence=0.7,
            timestamp=datetime.now().isoformat(),
            source="web_search",
        )
    
    def _normalize_generic(self, name: str, output: Any) -> NormalizedToolOutput:
        return NormalizedToolOutput(
            tool_name=name,
            success=True,
            data_type="text",
            summary=str(output)[:200],
            details=output,
            timestamp=datetime.now().isoformat(),
        )

# Usage
normalizer = ToolOutputNormalizer()

# Different tool outputs, same normalized format
calc_result = normalizer.normalize("calculator", 42)
weather_result = normalizer.normalize("weather", {"temp": 72, "city": "NYC", "weather": "sunny"})
search_result = normalizer.normalize("web_search", [
    {"title": "Result 1", "url": "..."}, 
    {"title": "Result 2", "url": "..."},
])

# All produce consistent LLM context
for r in [calc_result, weather_result, search_result]:
    print(r.to_llm_context())
    print("---")
```

---

## Q5. Implement Retry Logic with LLM Error Feedback ⭐⭐⭐

**Prompt:** When an LLM produces invalid output, you need to retry with the error message included. Build a retry decorator that feeds validation errors back to the model.

**Solution:**

```python
import functools
from typing import TypeVar, Callable, Any, Type
from pydantic import BaseModel, ValidationError
import json
import time

T = TypeVar("T", bound=BaseModel)

def retry_with_feedback(
    schema: Type[T],
    max_retries: int = 3,
    base_delay: float = 0.5,
):
    """
    Decorator that retries LLM calls with validation error feedback.
    
    The key insight: when the LLM produces invalid output, don't just retry blindly.
    Tell the model WHAT was wrong and let it fix it.
    """
    def decorator(func: Callable[..., str]) -> Callable[..., T | None]:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> T | None:
            last_error = None
            original_messages = kwargs.get("messages", args[0] if args else [])
            messages = list(original_messages)  # Copy to avoid mutation
            
            for attempt in range(max_retries + 1):
                try:
                    # Call the LLM
                    raw_output = func(messages=messages, **{k: v for k, v in kwargs.items() if k != "messages"})
                    
                    # Try to parse
                    # Handle potential markdown wrapping
                    clean = raw_output.strip()
                    if clean.startswith("```"):
                        clean = clean.split("\n", 1)[-1].rsplit("```", 1)[0].strip()
                    
                    result = schema.model_validate_json(clean)
                    
                    if attempt > 0:
                        print(f"  ✓ Succeeded on attempt {attempt + 1}")
                    
                    return result
                    
                except (ValidationError, json.JSONDecodeError) as e:
                    last_error = e
                    
                    if attempt < max_retries:
                        # Build feedback message
                        error_details = str(e)
                        
                        feedback = {
                            "role": "user",
                            "content": (
                                f"Your response was not valid. Error:\n{error_details}\n\n"
                                f"Required schema: {json.dumps(schema.model_json_schema())}\n\n"
                                f"Please respond with ONLY valid JSON. No markdown code blocks, "
                                f"no explanation, no preamble."
                            ),
                        }
                        
                        # Add the failed response and feedback
                        messages.append({"role": "assistant", "content": raw_output})
                        messages.append(feedback)
                        
                        time.sleep(base_delay * (2 ** attempt))
                        print(f"  ↻ Retry {attempt + 1}/{max_retries} (error: {str(e)[:80]})")
            
            print(f"  ✗ All {max_retries + 1} attempts failed. Last error: {last_error}")
            return None
        
        return wrapper
    return decorator


# Usage
class ExtractedEntities(BaseModel):
    people: list[str]
    locations: list[str]
    organizations: list[str]
    dates: list[str]

@retry_with_feedback(schema=ExtractedEntities, max_retries=2)
def extract_entities(messages: list[dict], model: str = "gpt-4o-mini") -> str:
    """Call LLM to extract entities. Returns raw string."""
    # In production: actual API call
    # Simulating a response
    return '{"people": ["John"], "locations": ["NYC"], "organizations": ["Acme"], "dates": ["2024"]}'

result = extract_entities(
    messages=[{"role": "user", "content": "Extract entities from: John visited NYC's Acme office in 2024."}],
)
if result:
    print(f"People: {result.people}, Locations: {result.locations}")
```

---

## Q6-10: Additional Coding Challenges (Brief)

### Q6. Safe Math Expression Evaluator ⭐⭐
Build a calculator tool that safely evaluates math expressions WITHOUT using `eval()`.

Key: Use `ast.literal_eval` or a custom parser. `eval()` is a security vulnerability — `eval("__import__('os').system('rm -rf /')")` would execute arbitrary code.

### Q7. API Circuit Breaker ⭐⭐⭐
Implement a circuit breaker that monitors external API health and stops calling failing APIs.

Key: Track failure rate over a sliding window. CLOSED (normal) → OPEN (failing, return fallback) → HALF-OPEN (test recovery).

### Q8. Tool Call Argument Sanitizer ⭐⭐⭐
Build a function that sanitizes tool arguments against injection attacks (SQL injection, command injection, path traversal).

Key: Allowlist valid characters, reject patterns like `; rm -rf`, `../`, `DROP TABLE`, validate types strictly.

### Q9. Multi-Tool Parallel Executor ⭐⭐⭐
When the LLM requests multiple independent tools (weather + calculator), execute them in parallel instead of sequentially.

Key: `asyncio.gather()` for independent tool calls. Sequential for dependent calls (search → then process results).

### Q10. Streaming Agent with Intermediate Results ⭐⭐⭐⭐
Build an agent that streams its reasoning to the UI in real-time: "Thinking... → Searching... → Found 3 results → Analyzing... → Here's your answer."

Key: Use `asyncio.Queue` or SSE to emit status updates at each agent step. The UI shows progress, not just the final answer.
