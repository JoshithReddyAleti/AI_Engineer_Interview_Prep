# 💻 Week 7 — Technical / Coding Questions

> **Focus:** Memory type implementations, persistence layers, state machines, checkpointing, compression, memory-aware RAG, cross-tenant isolation
>
> **How to use:** Build each before reading the solution. These are the coding challenges companies actually give in AI engineer live-code rounds.

---

## Q1. Build a Complete MemoryManager That Combines Window + Entity + Semantic Memory ⭐⭐⭐⭐

**Prompt:** Design and implement a `MemoryManager` class that orchestrates all three memory types with a unified API. Handle write-back, retrieval, token budgeting, and per-user isolation.

**Solution:**

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
import uuid

@dataclass
class Turn:
    role: str
    content: str
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    turn_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    tokens: int = 0

@dataclass  
class Entity:
    key: str
    value: str
    confidence: float
    source_turn_id: str
    updated_at: str

class MemoryManager:
    """
    Unified memory management combining window, entity, and semantic memory.
    Enforces per-user isolation and token budgeting.
    """
    
    def __init__(
        self,
        user_id: str,
        embed_fn,
        llm_fn,
        entity_store,
        vector_store,
        window_size: int = 10,
        token_budget: int = 4000,
    ):
        self.user_id = user_id
        self.embed_fn = embed_fn
        self.llm_fn = llm_fn
        self.entity_store = entity_store
        self.vector_store = vector_store
        self.window_size = window_size
        self.token_budget = token_budget
        
        # Window memory: recent turns in RAM
        self.window: list[Turn] = []
    
    def add_turn(self, role: str, content: str):
        """Add a turn to memory. Triggers write-back to all memory types."""
        turn = Turn(role=role, content=content, tokens=len(content) // 4)
        
        # 1. Window memory (in-memory, bounded)
        self.window.append(turn)
        if len(self.window) > self.window_size:
            self.window = self.window[-self.window_size:]
        
        # 2. Entity extraction (async in production, sync here for clarity)
        if role == "user" and len(content) > 20:  # Skip trivial turns
            self._extract_and_store_entities(turn)
        
        # 3. Semantic memory (embed and store)
        if len(content) > 30:  # Skip trivial turns
            self._store_semantic(turn)
    
    def get_context(self, query: str) -> dict:
        """
        Retrieve relevant memories for a query.
        Returns structured context ready for prompt assembly.
        """
        remaining_budget = self.token_budget
        context = {"window": [], "entities": [], "semantic": []}
        
        # Priority 1: Recent window (always include)
        window_tokens = sum(t.tokens for t in self.window)
        if window_tokens <= remaining_budget:
            context["window"] = self.window
            remaining_budget -= window_tokens
        else:
            # Trim oldest turns until budget fits
            trimmed = self.window[:]
            while sum(t.tokens for t in trimmed) > remaining_budget and trimmed:
                trimmed.pop(0)
            context["window"] = trimmed
            remaining_budget = 0
        
        # Priority 2: Entities (compact, high signal)
        entity_budget = min(remaining_budget, 500)
        entities = self.entity_store.get_all(user_id=self.user_id)
        for entity in entities:
            entity_tokens = len(f"{entity.key}: {entity.value}") // 4
            if entity_tokens <= entity_budget:
                context["entities"].append(entity)
                entity_budget -= entity_tokens
                remaining_budget -= entity_tokens
        
        # Priority 3: Semantic memory (only if budget remains)
        if remaining_budget > 200:
            query_emb = self.embed_fn(query)
            # CRITICAL: filter by user_id for isolation
            similar = self.vector_store.search(
                query_emb,
                filter={"user_id": self.user_id},
                top_k=5,
            )
            for item in similar:
                item_tokens = item["tokens"]
                if item_tokens <= remaining_budget:
                    context["semantic"].append(item)
                    remaining_budget -= item_tokens
        
        return context
    
    def _extract_and_store_entities(self, turn: Turn):
        """Extract entity facts from a turn and update entity store."""
        prompt = f"""Extract personal facts from this user statement.
Return JSON list of {{"key": str, "value": str, "confidence": float}}.
Only include facts that would be useful to remember long-term.
Return empty list if no facts.

Statement: {turn.content}"""
        
        result = self.llm_fn(prompt)
        try:
            import json
            facts = json.loads(result)
        except json.JSONDecodeError:
            return
        
        for fact in facts:
            existing = self.entity_store.get(user_id=self.user_id, key=fact["key"])
            if existing:
                # Conflict resolution: prefer higher confidence or newer
                if fact["confidence"] > existing.confidence:
                    self.entity_store.update(
                        user_id=self.user_id,
                        key=fact["key"],
                        value=fact["value"],
                        confidence=fact["confidence"],
                        source_turn_id=turn.turn_id,
                    )
            else:
                self.entity_store.create(
                    user_id=self.user_id,
                    key=fact["key"],
                    value=fact["value"],
                    confidence=fact["confidence"],
                    source_turn_id=turn.turn_id,
                )
    
    def _store_semantic(self, turn: Turn):
        """Embed turn and add to vector store with user isolation."""
        embedding = self.embed_fn(turn.content)
        self.vector_store.upsert(
            id=turn.turn_id,
            embedding=embedding,
            metadata={
                "user_id": self.user_id,   # CRITICAL for isolation
                "content": turn.content,
                "role": turn.role,
                "timestamp": turn.timestamp,
                "tokens": turn.tokens,
            },
        )
    
    def forget(self, before_timestamp: Optional[str] = None):
        """Forget memories older than timestamp (GDPR-compliant deletion)."""
        # Delete from vector store
        self.vector_store.delete(
            filter={"user_id": self.user_id, "timestamp_lt": before_timestamp}
        )
        # Delete from entity store
        self.entity_store.delete_by_user(
            user_id=self.user_id, before=before_timestamp
        )
        # Clear window
        if before_timestamp:
            self.window = [t for t in self.window if t.timestamp >= before_timestamp]
        else:
            self.window = []
    
    def export_all(self) -> dict:
        """Export all memory for a user (GDPR data access)."""
        return {
            "user_id": self.user_id,
            "exported_at": datetime.utcnow().isoformat(),
            "window": [t.__dict__ for t in self.window],
            "entities": [e.__dict__ for e in self.entity_store.get_all(user_id=self.user_id)],
            "semantic": self.vector_store.get_all(filter={"user_id": self.user_id}),
        }
```

**Key concepts demonstrated:**
- Multi-source memory fusion
- Priority-based token budgeting
- Per-user isolation via metadata filtering
- Write-back after every turn
- GDPR-compliant deletion and export
- Conflict resolution in entity memory

---

## Q2. Implement Summary Memory with Threshold-Based Compression ⭐⭐⭐

**Prompt:** Build memory that keeps recent turns verbatim but compresses older turns into an evolving summary. Compression triggers at a token threshold.

**Solution:**

```python
class SummaryMemory:
    """
    Hybrid memory: recent turns verbatim + evolving summary of older turns.
    Compression triggered when total tokens exceed threshold.
    """
    
    def __init__(
        self,
        llm_fn,
        max_recent_turns: int = 6,
        compression_threshold: int = 3000,
        keep_recent_tokens: int = 1500,
    ):
        self.llm_fn = llm_fn
        self.max_recent_turns = max_recent_turns
        self.compression_threshold = compression_threshold
        self.keep_recent_tokens = keep_recent_tokens
        
        self.summary: str = ""
        self.recent: list[Turn] = []
    
    def add_turn(self, role: str, content: str):
        turn = Turn(role=role, content=content, tokens=len(content) // 4)
        self.recent.append(turn)
        
        if self._should_compress():
            self._compress()
    
    def _should_compress(self) -> bool:
        """Check if compression should trigger."""
        total_tokens = self._recent_tokens() + len(self.summary) // 4
        return total_tokens > self.compression_threshold
    
    def _recent_tokens(self) -> int:
        return sum(t.tokens for t in self.recent)
    
    def _compress(self):
        """Compress old turns into summary, keep only recent."""
        # Identify turns to compress (all but the recent ones)
        to_compress = []
        keep_from = 0
        
        # Keep turns from newest, up to keep_recent_tokens
        cumulative_tokens = 0
        for i in range(len(self.recent) - 1, -1, -1):
            cumulative_tokens += self.recent[i].tokens
            if cumulative_tokens >= self.keep_recent_tokens:
                keep_from = i + 1
                break
        
        if keep_from == 0:
            return  # Not enough to compress meaningfully
        
        to_compress = self.recent[:keep_from]
        self.recent = self.recent[keep_from:]
        
        # Build compression prompt
        old_conversation = "\n".join(
            [f"{t.role}: {t.content}" for t in to_compress]
        )
        
        if self.summary:
            prompt = f"""Update this conversation summary to include the new turns.
Keep it concise. Preserve important facts, decisions, and context.

Existing summary:
{self.summary}

New turns to incorporate:
{old_conversation}

Updated summary:"""
        else:
            prompt = f"""Summarize this conversation. Keep it concise.
Preserve important facts, decisions, entities, and context.

Conversation:
{old_conversation}

Summary:"""
        
        self.summary = self.llm_fn(prompt).strip()
    
    def get_context(self) -> str:
        """Format for LLM context."""
        parts = []
        if self.summary:
            parts.append(f"[Previous conversation summary]\n{self.summary}\n")
        parts.append("[Recent turns]")
        for turn in self.recent:
            parts.append(f"{turn.role}: {turn.content}")
        return "\n".join(parts)
    
    def stats(self) -> dict:
        return {
            "recent_turns": len(self.recent),
            "recent_tokens": self._recent_tokens(),
            "summary_tokens": len(self.summary) // 4,
            "total_tokens": self._recent_tokens() + len(self.summary) // 4,
        }
```

**Design decisions explained:**
- Compression threshold prevents constant recompression
- Keep-recent-tokens ensures immediate context isn't lost
- Summary is UPDATED (not regenerated) — preserves earlier compression work
- Explicit stats method for observability

---

## Q3. Build a Persistent Entity Memory with SQLite ⭐⭐⭐

**Prompt:** Implement entity memory backed by SQLite. Support: get, upsert, delete, list, with proper indexing and per-user isolation.

**Solution:**

```python
import sqlite3
from contextlib import contextmanager
from datetime import datetime
from typing import Optional

class SQLiteEntityStore:
    """
    Persistent entity memory using SQLite.
    Per-user isolation enforced at query level.
    """
    
    def __init__(self, db_path: str = "entities.db"):
        self.db_path = db_path
        self._init_schema()
    
    @contextmanager
    def _connect(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
            conn.commit()
        finally:
            conn.close()
    
    def _init_schema(self):
        with self._connect() as conn:
            conn.executescript("""
                CREATE TABLE IF NOT EXISTS entities (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT NOT NULL,
                    key TEXT NOT NULL,
                    value TEXT NOT NULL,
                    confidence REAL NOT NULL DEFAULT 1.0,
                    source_turn_id TEXT,
                    created_at TEXT NOT NULL,
                    updated_at TEXT NOT NULL,
                    valid_until TEXT,
                    UNIQUE(user_id, key)
                );
                CREATE INDEX IF NOT EXISTS idx_user_id ON entities(user_id);
                CREATE INDEX IF NOT EXISTS idx_user_key ON entities(user_id, key);
                CREATE INDEX IF NOT EXISTS idx_updated ON entities(updated_at);
                
                CREATE TABLE IF NOT EXISTS entity_history (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    user_id TEXT NOT NULL,
                    key TEXT NOT NULL,
                    old_value TEXT,
                    new_value TEXT,
                    changed_at TEXT NOT NULL,
                    change_reason TEXT
                );
                CREATE INDEX IF NOT EXISTS idx_history_user ON entity_history(user_id);
            """)
    
    def upsert(
        self,
        user_id: str,
        key: str,
        value: str,
        confidence: float = 1.0,
        source_turn_id: Optional[str] = None,
    ):
        now = datetime.utcnow().isoformat()
        with self._connect() as conn:
            # Check for existing to record history
            existing = conn.execute(
                "SELECT value FROM entities WHERE user_id = ? AND key = ?",
                (user_id, key),
            ).fetchone()
            
            if existing and existing["value"] != value:
                # Log history for auditability
                conn.execute(
                    """INSERT INTO entity_history 
                    (user_id, key, old_value, new_value, changed_at, change_reason)
                    VALUES (?, ?, ?, ?, ?, ?)""",
                    (user_id, key, existing["value"], value, now, "user_update"),
                )
            
            conn.execute(
                """INSERT INTO entities (user_id, key, value, confidence, source_turn_id, created_at, updated_at)
                VALUES (?, ?, ?, ?, ?, ?, ?)
                ON CONFLICT(user_id, key) DO UPDATE SET 
                    value = excluded.value,
                    confidence = excluded.confidence,
                    source_turn_id = excluded.source_turn_id,
                    updated_at = excluded.updated_at""",
                (user_id, key, value, confidence, source_turn_id, now, now),
            )
    
    def get(self, user_id: str, key: str) -> Optional[dict]:
        with self._connect() as conn:
            row = conn.execute(
                "SELECT * FROM entities WHERE user_id = ? AND key = ?",
                (user_id, key),
            ).fetchone()
            return dict(row) if row else None
    
    def list(self, user_id: str, limit: int = 100) -> list[dict]:
        with self._connect() as conn:
            rows = conn.execute(
                """SELECT * FROM entities WHERE user_id = ? 
                ORDER BY updated_at DESC LIMIT ?""",
                (user_id, limit),
            ).fetchall()
            return [dict(r) for r in rows]
    
    def delete(self, user_id: str, key: Optional[str] = None):
        """Delete a specific entity or all entities for a user."""
        with self._connect() as conn:
            if key:
                conn.execute(
                    "DELETE FROM entities WHERE user_id = ? AND key = ?",
                    (user_id, key),
                )
            else:
                # GDPR: delete all entities for a user
                conn.execute("DELETE FROM entities WHERE user_id = ?", (user_id,))
                # Also delete history
                conn.execute("DELETE FROM entity_history WHERE user_id = ?", (user_id,))
    
    def get_history(self, user_id: str, key: Optional[str] = None) -> list[dict]:
        """Retrieve change history — critical for compliance audits."""
        with self._connect() as conn:
            if key:
                rows = conn.execute(
                    """SELECT * FROM entity_history 
                    WHERE user_id = ? AND key = ? ORDER BY changed_at DESC""",
                    (user_id, key),
                ).fetchall()
            else:
                rows = conn.execute(
                    """SELECT * FROM entity_history 
                    WHERE user_id = ? ORDER BY changed_at DESC LIMIT 100""",
                    (user_id,),
                ).fetchall()
            return [dict(r) for r in rows]
```

**Enterprise features included:**
- Per-user isolation via UNIQUE constraint and indexes
- Change history table for audit trails
- Cascading deletion for GDPR
- Optimistic upsert pattern with conflict handling

---

## Q4. Build a Vector Memory Store with Per-User Isolation ⭐⭐⭐⭐

**Prompt:** Implement semantic memory using FAISS. Must support: adding memories, searching with per-user filter, deletion, and export.

**Solution:**

```python
import numpy as np
import json
import os
from typing import Optional

class VectorMemoryStore:
    """
    Semantic memory with per-user isolation.
    Uses FAISS for efficient similarity search.
    """
    
    def __init__(self, embedding_dim: int = 1536, storage_dir: str = "./memory_data"):
        import faiss
        self.embedding_dim = embedding_dim
        self.storage_dir = storage_dir
        os.makedirs(storage_dir, exist_ok=True)
        
        # Global index — filtering happens at query time via metadata
        self.index = faiss.IndexIDMap(faiss.IndexFlatIP(embedding_dim))
        
        # Metadata store (in production: separate DB)
        self.metadata: dict[int, dict] = {}
        self.id_counter = 0
        
        self._load()
    
    def upsert(
        self,
        user_id: str,
        content: str,
        embedding: np.ndarray,
        metadata: Optional[dict] = None,
    ) -> int:
        """Add or update a memory. Returns the internal ID."""
        # Normalize for cosine similarity via inner product
        embedding = embedding / (np.linalg.norm(embedding) + 1e-8)
        
        self.id_counter += 1
        memory_id = self.id_counter
        
        self.index.add_with_ids(
            embedding.reshape(1, -1).astype(np.float32),
            np.array([memory_id]),
        )
        
        self.metadata[memory_id] = {
            "user_id": user_id,  # CRITICAL for isolation
            "content": content,
            "timestamp": (metadata or {}).get("timestamp"),
            **(metadata or {}),
        }
        
        self._persist()
        return memory_id
    
    def search(
        self,
        user_id: str,
        query_embedding: np.ndarray,
        top_k: int = 5,
        min_score: float = 0.3,
    ) -> list[dict]:
        """
        Search memories for a specific user.
        CRITICAL: retrieval is filtered by user_id.
        """
        if self.index.ntotal == 0:
            return []
        
        # Normalize query
        query_embedding = query_embedding / (np.linalg.norm(query_embedding) + 1e-8)
        query_embedding = query_embedding.reshape(1, -1).astype(np.float32)
        
        # Over-fetch since we'll filter by user_id
        fetch_k = min(top_k * 10, self.index.ntotal)
        scores, ids = self.index.search(query_embedding, fetch_k)
        
        results = []
        for score, memory_id in zip(scores[0], ids[0]):
            if memory_id == -1:
                continue
            
            meta = self.metadata.get(int(memory_id))
            if not meta:
                continue
            
            # CRITICAL FILTER — never leak across users
            if meta["user_id"] != user_id:
                continue
            
            if score < min_score:
                continue
            
            results.append({
                "memory_id": int(memory_id),
                "content": meta["content"],
                "score": float(score),
                "metadata": meta,
            })
            
            if len(results) >= top_k:
                break
        
        return results
    
    def delete_by_user(self, user_id: str):
        """GDPR-compliant: delete all memories for a user."""
        # Find all IDs for this user
        ids_to_delete = [
            mid for mid, meta in self.metadata.items()
            if meta["user_id"] == user_id
        ]
        
        if not ids_to_delete:
            return
        
        # Remove from FAISS index
        self.index.remove_ids(np.array(ids_to_delete))
        
        # Remove from metadata
        for mid in ids_to_delete:
            del self.metadata[mid]
        
        self._persist()
    
    def delete_by_id(self, memory_id: int, user_id: str):
        """Delete a specific memory. Verifies ownership."""
        meta = self.metadata.get(memory_id)
        if not meta:
            return False
        if meta["user_id"] != user_id:
            # SECURITY: never delete memories owned by other users
            raise PermissionError(f"Memory {memory_id} not owned by user {user_id}")
        
        self.index.remove_ids(np.array([memory_id]))
        del self.metadata[memory_id]
        self._persist()
        return True
    
    def export_user_memories(self, user_id: str) -> list[dict]:
        """GDPR-compliant data export."""
        return [
            {"memory_id": mid, **meta}
            for mid, meta in self.metadata.items()
            if meta["user_id"] == user_id
        ]
    
    def _persist(self):
        import faiss
        faiss.write_index(self.index, os.path.join(self.storage_dir, "index.faiss"))
        with open(os.path.join(self.storage_dir, "metadata.json"), "w") as f:
            json.dump({"metadata": self.metadata, "counter": self.id_counter}, f)
    
    def _load(self):
        import faiss
        index_path = os.path.join(self.storage_dir, "index.faiss")
        metadata_path = os.path.join(self.storage_dir, "metadata.json")
        
        if os.path.exists(index_path):
            self.index = faiss.read_index(index_path)
        
        if os.path.exists(metadata_path):
            with open(metadata_path) as f:
                data = json.load(f)
                # JSON keys are strings; convert back to int
                self.metadata = {int(k): v for k, v in data["metadata"].items()}
                self.id_counter = data["counter"]
```

**Enterprise features:**
- Hard filtering by user_id in EVERY search
- Ownership verification in delete
- Persistence to disk
- GDPR export and delete
- Over-fetching to compensate for filtering

---

## Q5. Implement a State Machine for a Multi-Step AI Workflow ⭐⭐⭐

**Prompt:** Build a state machine that tracks an AI workflow (e.g., customer onboarding: gather_info → verify → confirm → complete). Handle transitions, validation, and persistence.

**Solution:**

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Callable
import json

class State(Enum):
    START = "start"
    GATHERING_INFO = "gathering_info"
    VERIFYING = "verifying"
    AWAITING_CONFIRMATION = "awaiting_confirmation"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class WorkflowContext:
    user_id: str
    current_state: State = State.START
    data: dict = field(default_factory=dict)
    history: list = field(default_factory=list)
    error: Optional[str] = None

class WorkflowStateMachine:
    """
    State machine for AI workflow. Explicit transitions, validation, persistence.
    """
    
    # Define allowed transitions
    TRANSITIONS = {
        State.START: [State.GATHERING_INFO],
        State.GATHERING_INFO: [State.VERIFYING, State.FAILED],
        State.VERIFYING: [State.AWAITING_CONFIRMATION, State.GATHERING_INFO, State.FAILED],
        State.AWAITING_CONFIRMATION: [State.COMPLETED, State.GATHERING_INFO, State.FAILED],
        State.COMPLETED: [],  # Terminal
        State.FAILED: [State.START],  # Can restart
    }
    
    def __init__(self, context: WorkflowContext, persistence=None):
        self.context = context
        self.persistence = persistence  # Optional persistent store
        
        # Register state handlers
        self.handlers: dict[State, Callable] = {
            State.START: self._handle_start,
            State.GATHERING_INFO: self._handle_gathering,
            State.VERIFYING: self._handle_verifying,
            State.AWAITING_CONFIRMATION: self._handle_confirmation,
            State.COMPLETED: self._handle_completed,
        }
    
    def transition(self, new_state: State, reason: str = "") -> bool:
        """Attempt a state transition. Returns True if successful."""
        current = self.context.current_state
        allowed = self.TRANSITIONS.get(current, [])
        
        if new_state not in allowed:
            self.context.error = f"Illegal transition: {current.value} → {new_state.value}"
            return False
        
        # Record transition in history
        self.context.history.append({
            "from": current.value,
            "to": new_state.value,
            "reason": reason,
            "timestamp": datetime.utcnow().isoformat(),
        })
        
        self.context.current_state = new_state
        
        # Persist state change
        if self.persistence:
            self.persistence.save(self.context)
        
        return True
    
    def handle_input(self, user_input: str) -> dict:
        """Process user input for current state."""
        handler = self.handlers.get(self.context.current_state)
        if not handler:
            return {"error": "No handler for state", "state": self.context.current_state.value}
        
        return handler(user_input)
    
    def _handle_start(self, user_input: str) -> dict:
        self.transition(State.GATHERING_INFO, "user initiated")
        return {"message": "Let's begin. What's your name?"}
    
    def _handle_gathering(self, user_input: str) -> dict:
        # In real system: LLM would extract fields from user_input
        if "name" not in self.context.data:
            self.context.data["name"] = user_input.strip()
            return {"message": f"Thanks {user_input}. What's your email?"}
        elif "email" not in self.context.data:
            if "@" not in user_input:
                return {"message": "That doesn't look like an email. Please try again."}
            self.context.data["email"] = user_input.strip()
            self.transition(State.VERIFYING, "info gathered")
            return {"message": "Verifying your information..."}
        return {"message": "..."}
    
    def _handle_verifying(self, user_input: str) -> dict:
        # In real system: check external systems
        if self._verify(self.context.data):
            self.transition(State.AWAITING_CONFIRMATION, "verification passed")
            return {"message": f"Verified. Confirm: {self.context.data}? (yes/no)"}
        else:
            self.transition(State.GATHERING_INFO, "verification failed, redo")
            return {"message": "Couldn't verify. Please re-enter your name."}
    
    def _handle_confirmation(self, user_input: str) -> dict:
        if user_input.strip().lower() in ("yes", "y", "confirm"):
            self.transition(State.COMPLETED, "user confirmed")
            return {"message": "Complete!", "data": self.context.data}
        else:
            self.transition(State.GATHERING_INFO, "user requested changes")
            return {"message": "Let's redo. What's your name?"}
    
    def _handle_completed(self, user_input: str) -> dict:
        return {"message": "This workflow is complete.", "data": self.context.data}
    
    def _verify(self, data: dict) -> bool:
        # Placeholder — real verification logic
        return "@" in data.get("email", "")
    
    def get_state(self) -> dict:
        return {
            "current": self.context.current_state.value,
            "data": self.context.data,
            "history": self.context.history,
        }
```

**Design principles:**
- Explicit transition table — invalid transitions blocked
- History tracking for debugging and audit
- Persistence hook (checkpointing)
- Handlers separated per state — easy to test

---

## Q6-Q16: Additional Coding Challenges (Condensed)

### Q6. Build a Redis-Based Session State Store ⭐⭐
Session data (current workflow position, temporary variables) stored in Redis with TTL. Support: get, set, delete, extend_ttl. Use hash structures for structured session data.

### Q7. Implement Graph Memory with NetworkX ⭐⭐⭐⭐
Extract (subject, relation, object) triples from conversation. Store as directed graph. Support: add_relationship, find_related, query_by_relation, path_between. Include per-user subgraphs.

### Q8. Build a Memory Compression Pipeline ⭐⭐⭐
Given N conversation turns, compress older ones into summary while keeping recent verbatim. Support: incremental compression (add new turns without recompressing all), multiple compression levels (recent → medium → deep summary).

### Q9. Implement Episodic Memory with Timeline Queries ⭐⭐
Store timestamped events. Support: `get_events(user_id, from, to)`, `get_recent(user_id, N)`, `find_events(user_id, matching)`. Includes indexing by timestamp for fast range queries.

### Q10. Build Checkpoint Persistence for LangGraph ⭐⭐⭐⭐
Custom checkpointer that stores LangGraph state to PostgreSQL. Support: save_checkpoint, load_checkpoint, list_checkpoints (by thread_id), delete_old. Include concurrency handling.

### Q11. Implement Memory Retrieval Fusion ⭐⭐⭐
Query multiple memory types (window, entity, semantic, episodic). Merge results. Rank by combined score. Respect token budget. Return unified structured context.

### Q12. Build a GDPR-Compliant Deletion Pipeline ⭐⭐⭐⭐
When user requests deletion, cascade through: entity store, vector store, cache, backups, analytics logs. Track deletion progress. Return completion certificate with timestamp.

### Q13. Implement Memory-Aware RAG ⭐⭐⭐
Combine RAG (document retrieval) with memory (conversation history). Query considers BOTH. When memory contradicts retrieved documents, resolve intelligently. Show which sources were used.

### Q14. Build a Cross-Session Assistant with Persistent Memory ⭐⭐⭐
User returns 3 days later. Assistant remembers: name, previous topics, preferences, unresolved questions. Include: warm greeting referencing prior conversation, context restoration, forgetting of stale details.

### Q15. Implement Memory Access Auditing ⭐⭐⭐
Log every memory read/write with: timestamp, user_id, operation, memory_id, result. Detect anomalies: cross-user access attempts, unusual access patterns, mass read operations. Alert on suspicious activity.

### Q16. Build a Memory Quality Evaluator ⭐⭐⭐⭐
For a set of ground-truth (query, expected_memory) pairs, evaluate: retrieval accuracy (was expected memory retrieved?), ranking (was it in top-3?), latency (retrieval speed), quality (are retrieved memories actually useful?). Output: JSON report with metrics + failed cases.
