# 💻 Week 2 — Technical / Coding Questions

> **Focus:** Practical Python for AI — API clients, retry logic, data pipelines, concurrent processing
>
> **How to use:** Set a 20-minute timer per question. Code it from scratch before reading the solution.

---

## Q1. Build a Production-Grade LLM API Client ⭐⭐

**Prompt:** Write an `LLMClient` class that wraps an LLM API with: retries, timeouts, rate limiting, response validation, and structured logging. This is the #1 Python coding question for AI roles.

**Solution:**

```python
import os
import time
import json
import logging
from dataclasses import dataclass, field
from typing import Any

import httpx
from pydantic import BaseModel, Field
from dotenv import load_dotenv

load_dotenv()
logger = logging.getLogger(__name__)

class ChatMessage(BaseModel):
    role: str
    content: str

class LLMResponse(BaseModel):
    content: str
    model: str
    input_tokens: int
    output_tokens: int
    latency_ms: int
    retries_used: int = 0

@dataclass
class LLMClient:
    """
    Production LLM client with retries, timeouts, validation, and logging.
    """
    model: str = "gpt-4o-mini"
    api_key: str = field(default_factory=lambda: os.getenv("OPENAI_API_KEY", ""))
    base_url: str = "https://api.openai.com/v1"
    timeout: float = 30.0
    max_retries: int = 3
    base_delay: float = 1.0
    temperature: float = 0.7
    max_tokens: int = 1024
    
    def __post_init__(self):
        if not self.api_key:
            raise ValueError("API key not set. Check OPENAI_API_KEY environment variable.")
        self._client = httpx.Client(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=self.timeout,
        )
        self._request_count = 0
        self._total_tokens = 0
    
    def chat(
        self,
        messages: list[dict[str, str]],
        temperature: float | None = None,
        max_tokens: int | None = None,
    ) -> LLMResponse:
        """Send a chat completion request with retry logic."""
        
        payload = {
            "model": self.model,
            "messages": messages,
            "temperature": temperature or self.temperature,
            "max_tokens": max_tokens or self.max_tokens,
        }
        
        last_error = None
        for attempt in range(self.max_retries + 1):
            start = time.time()
            try:
                response = self._client.post(
                    "/chat/completions",
                    json=payload,
                )
                latency_ms = int((time.time() - start) * 1000)
                
                # Handle HTTP errors
                if response.status_code == 429:
                    # Rate limited — extract retry-after if available
                    retry_after = int(response.headers.get("retry-after", self.base_delay * (2 ** attempt)))
                    logger.warning(f"Rate limited. Waiting {retry_after}s (attempt {attempt + 1})")
                    time.sleep(retry_after)
                    continue
                
                if response.status_code >= 500:
                    logger.warning(f"Server error {response.status_code} (attempt {attempt + 1})")
                    time.sleep(self.base_delay * (2 ** attempt))
                    continue
                
                response.raise_for_status()
                data = response.json()
                
                # Extract and validate response
                result = LLMResponse(
                    content=data["choices"][0]["message"]["content"],
                    model=data["model"],
                    input_tokens=data["usage"]["prompt_tokens"],
                    output_tokens=data["usage"]["completion_tokens"],
                    latency_ms=latency_ms,
                    retries_used=attempt,
                )
                
                # Track usage
                self._request_count += 1
                self._total_tokens += result.input_tokens + result.output_tokens
                
                logger.info(
                    f"LLM call success | model={result.model} | "
                    f"tokens={result.input_tokens}+{result.output_tokens} | "
                    f"latency={result.latency_ms}ms | retries={attempt}"
                )
                
                return result
                
            except httpx.TimeoutException:
                last_error = TimeoutError(f"Request timed out after {self.timeout}s")
                logger.warning(f"Timeout (attempt {attempt + 1}/{self.max_retries + 1})")
                time.sleep(self.base_delay * (2 ** attempt))
                
            except httpx.HTTPStatusError as e:
                last_error = e
                if e.response.status_code < 500:
                    raise  # Client errors (400, 401, 403) don't retry
                time.sleep(self.base_delay * (2 ** attempt))
        
        raise RuntimeError(
            f"All {self.max_retries + 1} attempts failed. Last error: {last_error}"
        )
    
    def get_usage_stats(self) -> dict:
        return {
            "total_requests": self._request_count,
            "total_tokens": self._total_tokens,
        }
    
    def close(self):
        self._client.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.close()
```

**Follow-up:** "Make it async." The interviewer wants to see if you can convert `httpx.Client` → `httpx.AsyncClient`, `def chat` → `async def chat`, and `time.sleep` → `asyncio.sleep`.

---

## Q2. Concurrent LLM Calls with Progress Tracking ⭐⭐⭐

**Prompt:** Write a function that sends N prompts to an LLM API concurrently (max K parallel), shows progress, handles partial failures, and returns results in order.

**Solution:**

```python
import asyncio
from dataclasses import dataclass
from typing import Any

@dataclass
class BatchResult:
    index: int
    prompt: str
    response: str | None
    error: str | None
    latency_ms: int

async def batch_llm_calls(
    prompts: list[str],
    max_concurrent: int = 5,
    model: str = "gpt-4o-mini",
    on_progress: callable = None,
) -> list[BatchResult]:
    """
    Process N prompts concurrently with max K parallel connections.
    Returns results in original order, even if they complete out of order.
    """
    semaphore = asyncio.Semaphore(max_concurrent)
    results: list[BatchResult] = [None] * len(prompts)
    completed = 0
    
    async def process_one(index: int, prompt: str):
        nonlocal completed
        async with semaphore:
            import time
            start = time.time()
            try:
                # Simulate async LLM call
                # In production: response = await async_client.chat(...)
                await asyncio.sleep(0.5)  # Simulated API latency
                response_text = f"Response to: {prompt[:30]}"
                
                results[index] = BatchResult(
                    index=index,
                    prompt=prompt,
                    response=response_text,
                    error=None,
                    latency_ms=int((time.time() - start) * 1000),
                )
            except Exception as e:
                results[index] = BatchResult(
                    index=index,
                    prompt=prompt,
                    response=None,
                    error=str(e),
                    latency_ms=int((time.time() - start) * 1000),
                )
            finally:
                completed += 1
                if on_progress:
                    on_progress(completed, len(prompts))
    
    # Launch all tasks
    tasks = [
        asyncio.create_task(process_one(i, p))
        for i, p in enumerate(prompts)
    ]
    
    await asyncio.gather(*tasks)
    
    # Summary
    successes = sum(1 for r in results if r.error is None)
    failures = sum(1 for r in results if r.error is not None)
    print(f"\nBatch complete: {successes} succeeded, {failures} failed")
    
    return results


# Usage
def show_progress(done, total):
    pct = done / total * 100
    print(f"\r[{'█' * int(pct/5):<20}] {done}/{total} ({pct:.0f}%)", end="", flush=True)

# asyncio.run(batch_llm_calls(["prompt1", "prompt2", ...], on_progress=show_progress))
```

---

## Q3. JSON Response Parser with Fallback Extraction ⭐⭐

**Prompt:** LLMs often return JSON wrapped in markdown code blocks, with trailing commas, or with commentary text around it. Build a robust JSON extractor that handles all these cases.

**Solution:**

```python
import json
import re
from typing import Any

def extract_json(raw_text: str) -> Any:
    """
    Robustly extract JSON from LLM output.
    
    Handles:
    1. Clean JSON
    2. JSON in ```json code blocks
    3. JSON in ``` code blocks (no language tag)
    4. JSON with surrounding commentary
    5. JSON with trailing commas
    6. Multiple JSON objects (returns first valid one)
    """
    
    # Strategy 1: Try direct parse first (fastest path)
    try:
        return json.loads(raw_text.strip())
    except json.JSONDecodeError:
        pass
    
    # Strategy 2: Extract from markdown code blocks
    code_block_patterns = [
        r'```json\s*\n?(.*?)\n?\s*```',   # ```json ... ```
        r'```\s*\n?(.*?)\n?\s*```',         # ``` ... ```
    ]
    for pattern in code_block_patterns:
        match = re.search(pattern, raw_text, re.DOTALL)
        if match:
            try:
                return json.loads(match.group(1).strip())
            except json.JSONDecodeError:
                # Try fixing common issues
                fixed = _fix_common_json_issues(match.group(1).strip())
                try:
                    return json.loads(fixed)
                except json.JSONDecodeError:
                    continue
    
    # Strategy 3: Find JSON-like structures in the text
    # Look for {...} or [...]
    json_patterns = [
        r'(\{[^{}]*(?:\{[^{}]*\}[^{}]*)*\})',   # Nested objects
        r'(\[[^\[\]]*(?:\[[^\[\]]*\][^\[\]]*)*\])',  # Nested arrays
    ]
    for pattern in json_patterns:
        matches = re.findall(pattern, raw_text, re.DOTALL)
        for match in matches:
            try:
                return json.loads(match)
            except json.JSONDecodeError:
                fixed = _fix_common_json_issues(match)
                try:
                    return json.loads(fixed)
                except json.JSONDecodeError:
                    continue
    
    # Strategy 4: Nothing worked
    raise ValueError(
        f"Could not extract valid JSON from response. "
        f"Raw text (first 200 chars): {raw_text[:200]}"
    )

def _fix_common_json_issues(text: str) -> str:
    """Fix common JSON issues from LLM output."""
    # Remove trailing commas before } or ]
    text = re.sub(r',\s*([}\]])', r'\1', text)
    
    # Replace single quotes with double quotes (careful with apostrophes)
    # Only do this if there are no double quotes already
    if '"' not in text and "'" in text:
        text = text.replace("'", '"')
    
    # Remove comments (// style)
    text = re.sub(r'//.*?$', '', text, flags=re.MULTILINE)
    
    # Fix unquoted keys (naive approach)
    text = re.sub(r'(\{|,)\s*(\w+)\s*:', r'\1 "\2":', text)
    
    return text


# Test cases
test_cases = [
    '{"name": "test", "value": 42}',                           # Clean JSON
    '```json\n{"name": "test"}\n```',                           # Code block
    'Here is the result: {"name": "test"} Hope this helps!',   # Surrounded by text
    '{"name": "test", "items": [1, 2, 3,]}',                   # Trailing comma
    "```\n{'name': 'test'}\n```",                               # Single quotes
]

for tc in test_cases:
    try:
        result = extract_json(tc)
        print(f"✓ Parsed: {result}")
    except ValueError as e:
        print(f"✗ Failed: {e}")
```

---

## Q4. Configuration Manager for AI Projects ⭐⭐

**Prompt:** Build a configuration management system that loads settings from (in priority order): environment variables > .env file > config.yaml > defaults. Validate all config values.

**Solution:**

```python
import os
from pathlib import Path
from typing import Any

import yaml
from pydantic import BaseModel, Field, field_validator
from dotenv import dotenv_values

class LLMConfig(BaseModel):
    """Validated configuration for LLM operations."""
    
    # API settings
    api_key: str = Field(description="LLM API key")
    model: str = Field(default="gpt-4o-mini")
    base_url: str = Field(default="https://api.openai.com/v1")
    
    # Generation settings
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    max_tokens: int = Field(default=1024, ge=1, le=128000)
    top_p: float = Field(default=0.95, ge=0.0, le=1.0)
    
    # Operational settings
    timeout_seconds: float = Field(default=30.0, ge=1.0)
    max_retries: int = Field(default=3, ge=0, le=10)
    rate_limit_rpm: int = Field(default=60, ge=1)
    
    # Logging
    log_level: str = Field(default="INFO")
    log_requests: bool = Field(default=False)
    
    @field_validator("log_level")
    @classmethod
    def validate_log_level(cls, v):
        valid = {"DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"}
        if v.upper() not in valid:
            raise ValueError(f"log_level must be one of {valid}")
        return v.upper()
    
    @field_validator("api_key")
    @classmethod
    def validate_api_key(cls, v):
        if not v or v == "sk-your-key-here":
            raise ValueError("Valid API key required. Set OPENAI_API_KEY env var.")
        return v


class ConfigManager:
    """
    Loads config from multiple sources with priority:
    1. Environment variables (highest)
    2. .env file
    3. config.yaml
    4. Defaults (lowest)
    """
    
    ENV_PREFIX = "LLM_"  # LLM_MODEL, LLM_TEMPERATURE, etc.
    
    def __init__(
        self,
        config_path: str | Path = "config.yaml",
        env_path: str | Path = ".env",
    ):
        self.config_path = Path(config_path)
        self.env_path = Path(env_path)
    
    def load(self) -> LLMConfig:
        """Load and merge configuration from all sources."""
        
        # Layer 1: Defaults (handled by Pydantic Field defaults)
        merged = {}
        
        # Layer 2: YAML config file
        if self.config_path.exists():
            with open(self.config_path) as f:
                yaml_config = yaml.safe_load(f) or {}
            merged.update(yaml_config)
        
        # Layer 3: .env file
        if self.env_path.exists():
            env_values = dotenv_values(self.env_path)
            mapped = self._map_env_to_config(env_values)
            merged.update(mapped)
        
        # Layer 4: Actual environment variables (highest priority)
        env_vars = {k: v for k, v in os.environ.items() if k.startswith(self.ENV_PREFIX)}
        mapped = self._map_env_to_config(env_vars)
        merged.update(mapped)
        
        # Also check for OPENAI_API_KEY directly
        if "api_key" not in merged and "OPENAI_API_KEY" in os.environ:
            merged["api_key"] = os.environ["OPENAI_API_KEY"]
        
        # Validate and return
        return LLMConfig(**merged)
    
    def _map_env_to_config(self, env_dict: dict) -> dict:
        """Map environment variable names to config field names."""
        result = {}
        for key, value in env_dict.items():
            # Strip prefix and lowercase: LLM_MAX_TOKENS → max_tokens
            config_key = key.replace(self.ENV_PREFIX, "").lower()
            
            # Also handle non-prefixed common keys
            if key == "OPENAI_API_KEY":
                config_key = "api_key"
            
            if config_key and value:
                # Type coercion for known types
                result[config_key] = self._coerce_type(config_key, value)
        
        return result
    
    def _coerce_type(self, key: str, value: str) -> Any:
        """Coerce string environment values to appropriate types."""
        bool_fields = {"log_requests"}
        int_fields = {"max_tokens", "max_retries", "rate_limit_rpm"}
        float_fields = {"temperature", "top_p", "timeout_seconds"}
        
        if key in bool_fields:
            return value.lower() in ("true", "1", "yes")
        elif key in int_fields:
            return int(value)
        elif key in float_fields:
            return float(value)
        return value


# Usage
config = ConfigManager().load()
print(f"Model: {config.model}, Temp: {config.temperature}")
```

---

## Q5. Build a Streaming File Processor for Document Ingestion ⭐⭐⭐

**Prompt:** Write a pipeline that reads files from a directory (PDF, TXT, MD), chunks the text, and yields chunks as a generator. Must handle files larger than available RAM.

**Solution:**

```python
from pathlib import Path
from dataclasses import dataclass
from typing import Generator
import re

@dataclass
class TextChunk:
    text: str
    source_file: str
    chunk_index: int
    char_offset: int
    token_estimate: int  # rough estimate: chars / 4

def chunk_text(
    text: str,
    chunk_size: int = 500,       # target tokens per chunk
    chunk_overlap: int = 50,     # overlap tokens
) -> Generator[str, None, None]:
    """
    Split text into overlapping chunks at sentence boundaries.
    """
    # Convert token targets to char estimates (1 token ≈ 4 chars)
    char_size = chunk_size * 4
    char_overlap = chunk_overlap * 4
    
    # Split into sentences
    sentences = re.split(r'(?<=[.!?])\s+', text)
    
    current_chunk = []
    current_length = 0
    
    for sentence in sentences:
        sentence_len = len(sentence)
        
        if current_length + sentence_len > char_size and current_chunk:
            # Yield current chunk
            yield " ".join(current_chunk)
            
            # Keep overlap
            overlap_text = " ".join(current_chunk)
            overlap_start = max(0, len(overlap_text) - char_overlap)
            overlap_sentences = overlap_text[overlap_start:].split(". ")
            
            current_chunk = [s for s in overlap_sentences if s.strip()]
            current_length = sum(len(s) for s in current_chunk)
        
        current_chunk.append(sentence)
        current_length += sentence_len
    
    # Yield remaining
    if current_chunk:
        yield " ".join(current_chunk)

def read_file_streaming(filepath: Path) -> Generator[str, None, None]:
    """Read a file and yield text content. Handles large files."""
    suffix = filepath.suffix.lower()
    
    if suffix in ('.txt', '.md'):
        # Stream text files in chunks to handle large files
        with open(filepath, 'r', encoding='utf-8', errors='replace') as f:
            buffer = []
            for line in f:
                buffer.append(line)
                if len(buffer) >= 100:  # Yield every 100 lines
                    yield "".join(buffer)
                    buffer = []
            if buffer:
                yield "".join(buffer)
    
    elif suffix == '.pdf':
        # Placeholder — in production use PyMuPDF or pdfplumber
        # Stream page by page to avoid loading entire PDF in memory
        try:
            import fitz  # PyMuPDF
            doc = fitz.open(str(filepath))
            for page in doc:
                yield page.get_text()
            doc.close()
        except ImportError:
            raise ImportError("Install PyMuPDF: pip install pymupdf")
    
    else:
        raise ValueError(f"Unsupported file type: {suffix}")

def process_directory(
    directory: str | Path,
    chunk_size: int = 500,
    chunk_overlap: int = 50,
    extensions: set[str] = {'.txt', '.md', '.pdf'},
) -> Generator[TextChunk, None, None]:
    """
    Stream-process all documents in a directory.
    Memory-efficient: only one chunk in memory at a time.
    """
    dir_path = Path(directory)
    
    for filepath in sorted(dir_path.rglob("*")):
        if filepath.suffix.lower() not in extensions:
            continue
        if filepath.is_dir():
            continue
        
        print(f"Processing: {filepath.name}")
        chunk_index = 0
        char_offset = 0
        
        try:
            # Stream file content → chunk → yield
            for text_block in read_file_streaming(filepath):
                for chunk_text_str in chunk_text(text_block, chunk_size, chunk_overlap):
                    if chunk_text_str.strip():
                        yield TextChunk(
                            text=chunk_text_str.strip(),
                            source_file=str(filepath),
                            chunk_index=chunk_index,
                            char_offset=char_offset,
                            token_estimate=len(chunk_text_str) // 4,
                        )
                        chunk_index += 1
                        char_offset += len(chunk_text_str)
        
        except Exception as e:
            print(f"Error processing {filepath}: {e}")
            continue

# Usage — constant memory regardless of corpus size
for chunk in process_directory("./documents"):
    print(f"[{chunk.source_file}] chunk {chunk.chunk_index}: {chunk.text[:60]}...")
    # In production: embed(chunk.text) → store_in_vector_db(embedding, chunk.metadata)
```

---

## Q6. Environment-Aware Application Factory ⭐⭐

**Prompt:** Build a factory function that creates a properly configured LLM application based on the environment (development, staging, production). Different environments should use different models, logging levels, and safety settings.

**Solution:**

```python
import os
import logging
from dataclasses import dataclass
from enum import Enum
from typing import Protocol

class Environment(Enum):
    DEV = "development"
    STAGING = "staging"
    PROD = "production"

@dataclass
class AppConfig:
    env: Environment
    model: str
    fallback_model: str
    temperature: float
    max_tokens: int
    log_level: str
    enable_caching: bool
    enable_rate_limiting: bool
    enable_content_filter: bool
    max_retries: int
    timeout: float

# Environment-specific defaults
ENV_CONFIGS = {
    Environment.DEV: AppConfig(
        env=Environment.DEV,
        model="gpt-4o-mini",            # Cheap model for dev
        fallback_model="gpt-4o-mini",
        temperature=0.7,
        max_tokens=512,
        log_level="DEBUG",
        enable_caching=False,            # Fresh responses for debugging
        enable_rate_limiting=False,
        enable_content_filter=False,     # Less restrictive for testing
        max_retries=1,
        timeout=60.0,                    # Longer timeout for debugging
    ),
    Environment.STAGING: AppConfig(
        env=Environment.STAGING,
        model="gpt-4o-mini",
        fallback_model="gpt-4o-mini",
        temperature=0.3,
        max_tokens=1024,
        log_level="INFO",
        enable_caching=True,
        enable_rate_limiting=True,
        enable_content_filter=True,
        max_retries=2,
        timeout=30.0,
    ),
    Environment.PROD: AppConfig(
        env=Environment.PROD,
        model="gpt-4o",                 # Best model for production
        fallback_model="gpt-4o-mini",   # Cheap fallback
        temperature=0.2,                # More deterministic
        max_tokens=2048,
        log_level="WARNING",
        enable_caching=True,
        enable_rate_limiting=True,
        enable_content_filter=True,
        max_retries=3,
        timeout=30.0,
    ),
}

def create_app(env: str | None = None) -> AppConfig:
    """
    Factory: create app configuration based on environment.
    
    Priority: explicit arg > ENV_NAME env var > default to development
    """
    if env is None:
        env = os.getenv("ENV_NAME", "development")
    
    try:
        environment = Environment(env)
    except ValueError:
        logging.warning(f"Unknown environment '{env}', defaulting to development")
        environment = Environment.DEV
    
    config = ENV_CONFIGS[environment]
    
    # Allow env var overrides for specific settings
    if model_override := os.getenv("LLM_MODEL"):
        config.model = model_override
    
    logging.basicConfig(level=getattr(logging, config.log_level))
    logging.info(f"App initialized: env={config.env.value}, model={config.model}")
    
    return config


# Usage
config = create_app()  # Reads from ENV_NAME environment variable
print(f"Environment: {config.env.value}")
print(f"Model: {config.model}")
print(f"Caching: {config.enable_caching}")
```

---

## Q7. Deep Copy of an Undirected Graph — DFS vs BFS ⭐⭐⭐

**Prompt:** How would you create a deep copy of an undirected graph? Why DFS over BFS?

**What the interviewer is really testing:** Classic data structure question that maps directly to AI concepts (knowledge graphs, dependency graphs, agent workflows).

**Solution:**

```python
class GraphNode:
    def __init__(self, val: int, neighbors: list['GraphNode'] = None):
        self.val = val
        self.neighbors = neighbors or []

def clone_graph_dfs(node: GraphNode | None) -> GraphNode | None:
    """
    Deep copy using DFS (recursive).
    
    Why DFS over BFS for this problem:
    1. Natural recursive structure matches graph traversal
    2. Simpler code — no explicit queue needed
    3. Memory: DFS uses O(h) stack space where h = max depth
       BFS uses O(w) queue space where w = max width
       For deep, narrow graphs (common in AI chains), DFS is better
    4. DFS naturally handles the "clone as you go" pattern
    """
    if not node:
        return None
    
    cloned = {}  # Maps original node → cloned node
    
    def dfs(original: GraphNode) -> GraphNode:
        if original in cloned:
            return cloned[original]
        
        # Create clone
        copy = GraphNode(original.val)
        cloned[original] = copy  # Register before recursing (handles cycles)
        
        # Clone neighbors
        for neighbor in original.neighbors:
            copy.neighbors.append(dfs(neighbor))
        
        return copy
    
    return dfs(node)


def clone_graph_bfs(node: GraphNode | None) -> GraphNode | None:
    """
    Deep copy using BFS (iterative).
    
    Better when:
    - Graph is very deep (DFS might stack overflow)
    - You need level-order processing
    - You want iterative control (easier to add logging/progress)
    """
    if not node:
        return None
    
    from collections import deque
    
    cloned = {node: GraphNode(node.val)}
    queue = deque([node])
    
    while queue:
        current = queue.popleft()
        
        for neighbor in current.neighbors:
            if neighbor not in cloned:
                cloned[neighbor] = GraphNode(neighbor.val)
                queue.append(neighbor)
            cloned[current].neighbors.append(cloned[neighbor])
    
    return cloned[node]

# AI Engineering connection:
# This exact pattern appears in:
# - Cloning agent workflow DAGs
# - Deep-copying conversation trees (for branching conversations)
# - Duplicating RAG pipeline graphs for A/B testing
# - Serializing/deserializing knowledge graphs
```

---

## Q8. FastAPI Lifespan Events for AI Applications ⭐⭐⭐

**Prompt:** How do you manage shared resources (LLM clients, DB connections, model weights) in FastAPI using lifespan events? What happens during graceful shutdown?

**Solution:**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
import httpx

# Shared resources — expensive to create, reused across requests
class AppState:
    llm_client: httpx.AsyncClient | None = None
    embedding_model: object | None = None
    vector_db: object | None = None

state = AppState()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Lifespan manager — runs once on startup and once on shutdown.
    
    Startup: Pre-load expensive resources
    - LLM client with connection pooling
    - Embedding model weights into GPU memory
    - Database connection pools
    
    Shutdown: Graceful cleanup
    - Close HTTP connections (prevents resource leaks)
    - Flush pending writes
    - Release GPU memory
    """
    # === STARTUP ===
    print("🚀 Loading resources...")
    
    # 1. HTTP client with connection pooling (reuse TCP connections)
    state.llm_client = httpx.AsyncClient(
        base_url="https://api.openai.com/v1",
        headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}"},
        timeout=30.0,
        limits=httpx.Limits(
            max_connections=20,          # Connection pool size
            max_keepalive_connections=10, # Keep-alive pool
        ),
    )
    
    # 2. Embedding model (loaded once, used for every request)
    # state.embedding_model = SentenceTransformer("all-MiniLM-L6-v2")
    
    # 3. Vector DB connection
    # state.vector_db = await connect_to_pinecone()
    
    print("✅ Resources loaded")
    
    yield  # Application runs here — handles requests
    
    # === SHUTDOWN (graceful) ===
    print("🛑 Shutting down gracefully...")
    
    # Close HTTP client (flushes pending requests, releases connections)
    if state.llm_client:
        await state.llm_client.aclose()
    
    # Close DB connections
    # if state.vector_db:
    #     await state.vector_db.close()
    
    # Free GPU memory
    # del state.embedding_model
    # torch.cuda.empty_cache()
    
    print("✅ Cleanup complete")

app = FastAPI(lifespan=lifespan)

@app.post("/chat")
async def chat(request: Request):
    """
    Each request reuses the shared LLM client.
    No client creation overhead per request.
    """
    body = await request.json()
    
    response = await state.llm_client.post(
        "/chat/completions",
        json={"model": "gpt-4o-mini", "messages": body["messages"]},
    )
    
    return response.json()

# What happens during graceful shutdown:
# 1. Server stops accepting NEW connections
# 2. In-flight requests are allowed to complete (within timeout)
# 3. lifespan __aexit__ runs (our cleanup code)
# 4. Process exits
#
# Kubernetes sends SIGTERM → app has 30s (default) to finish
# If not done in 30s → SIGKILL (forced kill, no cleanup)
# That's why cleanup should be fast and non-blocking
```

---

## Q9. Build a Request/Response Logger for AI Debugging ⭐⭐

**Prompt:** Build a logging decorator that captures LLM request/response pairs for debugging, with configurable log levels and PII redaction.

**Solution:**

```python
import functools
import json
import logging
import re
import time
from typing import Callable, Any

logger = logging.getLogger("llm_audit")

# PII patterns to redact
PII_PATTERNS = [
    (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL_REDACTED]'),
    (r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE_REDACTED]'),
    (r'\b\d{3}-\d{2}-\d{4}\b', '[SSN_REDACTED]'),
    (r'\b(?:\d{4}[-\s]?){3}\d{4}\b', '[CARD_REDACTED]'),
]

def redact_pii(text: str) -> str:
    """Remove PII patterns from text before logging."""
    for pattern, replacement in PII_PATTERNS:
        text = re.sub(pattern, replacement, text)
    return text

def log_llm_call(
    log_level: str = "INFO",
    redact: bool = True,
    log_full_response: bool = False,
    max_content_length: int = 200,
):
    """
    Decorator that logs LLM call details.
    
    Usage:
        @log_llm_call(log_level="DEBUG", redact=True)
        def call_llm(messages, model="gpt-4o"):
            ...
    """
    level = getattr(logging, log_level.upper())
    
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            # Capture request details
            start_time = time.time()
            request_id = f"req_{int(start_time * 1000) % 1000000}"
            
            # Extract messages from args or kwargs
            messages = kwargs.get("messages") or (args[0] if args else [])
            model = kwargs.get("model", "unknown")
            
            # Log request
            last_message = messages[-1]["content"] if messages else ""
            if redact:
                last_message = redact_pii(last_message)
            
            logger.log(level, json.dumps({
                "event": "llm_request",
                "request_id": request_id,
                "model": model,
                "message_count": len(messages),
                "last_message_preview": last_message[:max_content_length],
            }))
            
            try:
                result = func(*args, **kwargs)
                elapsed_ms = int((time.time() - start_time) * 1000)
                
                # Log response
                response_preview = str(result)[:max_content_length] if not log_full_response else str(result)
                if redact:
                    response_preview = redact_pii(response_preview)
                
                logger.log(level, json.dumps({
                    "event": "llm_response",
                    "request_id": request_id,
                    "model": model,
                    "latency_ms": elapsed_ms,
                    "status": "success",
                    "response_preview": response_preview,
                }))
                
                return result
                
            except Exception as e:
                elapsed_ms = int((time.time() - start_time) * 1000)
                logger.error(json.dumps({
                    "event": "llm_error",
                    "request_id": request_id,
                    "model": model,
                    "latency_ms": elapsed_ms,
                    "error_type": type(e).__name__,
                    "error_message": str(e),
                }))
                raise
        
        return wrapper
    return decorator


# Usage
@log_llm_call(log_level="INFO", redact=True)
def call_llm(messages: list[dict], model: str = "gpt-4o"):
    # Actual LLM call here
    return {"content": "Response text", "tokens": 42}

call_llm(
    messages=[{"role": "user", "content": "My email is test@example.com, help me"}],
    model="gpt-4o",
)
```

---

## Q10. Docker Image Optimization for AI Applications ⭐⭐⭐

**Prompt:** How would you optimize a Docker image for a Python AI application to reduce build time, image size, and deployment speed?

**Solution:**

```dockerfile
# === Stage 1: Builder (install dependencies) ===
FROM python:3.11-slim as builder

WORKDIR /build

# Install build dependencies separately (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# === Stage 2: Runtime (minimal image) ===
FROM python:3.11-slim

WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /install /usr/local

# Copy application code LAST (changes most frequently → cache-friendly)
COPY app/ ./app/
COPY config/ ./config/

# Non-root user for security
RUN useradd --create-home appuser
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health')" || exit 1

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Key optimization techniques:**

```python
# 1. LAYER ORDERING — least-changing layers first
# requirements.txt changes rarely → cached
# app/ code changes often → rebuilt each time
# Wrong: COPY . . then pip install (busts cache on every code change)
# Right: COPY requirements.txt, pip install, COPY app/ (only app/ layer rebuilds)

# 2. MULTI-STAGE BUILD — build deps don't ship
# Builder stage: has gcc, build tools (~1GB)
# Runtime stage: only Python + packages (~300MB)
# Savings: 50-70% smaller image

# 3. .dockerignore — don't copy junk into the image
# .git, __pycache__, .env, venv/, tests/, docs/, *.md
# Speeds up COPY and reduces image size

# 4. REQUIREMENTS PINNING — deterministic builds
# requirements.txt with exact versions: openai==1.35.0
# Not: openai>=1.0 (could get different version on each build)

# 5. SLIM BASE IMAGE — python:3.11-slim vs python:3.11
# slim: ~150MB, full: ~900MB
# alpine: ~50MB but breaks many Python packages (musl vs glibc)

# 6. REDUCE NETWORK TRAFFIC IN K8S
# - Use a private container registry in the same region
# - Pre-pull base images to nodes
# - Use image layer caching (containerd snapshotter)
# - Compress layers with zstd

# 7. MULTI-REGION K8S CHALLENGES
# - Image pull latency from remote registries
# - Model weight files (GBs) must be accessible in every region
# - Solution: replicate registry per region + shared model storage (S3/GCS)
```
