# 🏗️ Week 7 — System Design Questions

> **Focus:** Multi-tenant memory at scale, governance & compliance, monitoring, personal assistant architecture, distributed state, memory security
>
> **How to use:** 30-45 min whiteboard rounds. These are the enterprise questions where memory becomes a compliance, security, and trust problem — not just a technical one.

---

## Q1. Design a Memory System for a Personal AI Assistant Serving 10 Million Users ⭐⭐⭐⭐

**Prompt:** "Design memory infrastructure for a personal AI assistant. 10M users. Each has multi-session conversations. Must remember context, preferences, and relationships across months. Must respect GDPR."

**Architecture:**

```
┌────────────── User Query Flow ─────────────────────────────────┐
│                                                                 │
│  User (authenticated) → API Gateway → Assistant Service         │
│                                            │                    │
│                                            ▼                    │
│                             ┌────────────────────────┐          │
│                             │  Memory Retrieval Layer │          │
│                             │  (per-user isolation)   │          │
│                             └───────┬────────────────┘          │
│                                     │                            │
│              ┌──────────────────────┼───────────────────┐        │
│              ▼                      ▼                   ▼        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ Hot Cache        │  │ Warm Storage      │  │ Cold Storage │  │
│  │ (Redis)          │  │ (PostgreSQL)      │  │ (S3/Glacier) │  │
│  │                  │  │                   │  │              │  │
│  │ Last 30 days     │  │ Last 12 months    │  │ Older archive│  │
│  │ Active sessions  │  │ Entity + episodic │  │ (audit only) │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                 │
│                             ┌────────────────────────┐          │
│                             │  Vector Store           │          │
│                             │  (Pinecone/Weaviate)    │          │
│                             │  Per-user namespace     │          │
│                             │  Semantic memory        │          │
│                             └────────────────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

**Storage tier design:**

**Hot tier (Redis):**
- Active sessions (< 30 min inactivity)
- Recent turns (last 20)
- Entity cache (frequently accessed facts)
- TTL: 30 days sliding
- Estimated size: 100 KB × 10M = 1 TB Redis
- Optimize for: read latency < 5ms

**Warm tier (PostgreSQL, sharded):**
- All entity memories
- Episodic memory (event log)
- Compressed summaries
- Sharded by user_id (partition on hash)
- Estimated size: 5 MB × 10M = 50 TB total, sharded across 20+ instances
- Optimize for: query flexibility, indexes on user_id + timestamps

**Cold tier (S3 + Glacier):**
- Memories older than 12 months
- Compressed archive format
- Optimize for: cheap storage, occasional access for GDPR requests

**Vector store (Pinecone):**
- Semantic memory
- Per-user namespace (never cross-user searches)
- Auto-scaling
- Cost: ~$0.10 per user per month at 10M scale = $1M/month
- **This is often the biggest memory cost line item**

**Cross-cutting concerns:**

**Sharding strategy:** User-based sharding. `shard_id = hash(user_id) % 20`. All memory operations for a user hit the same shard. Enables horizontal scaling.

**Consistency model:** Eventually consistent between tiers. Hot cache may lag warm storage by seconds. Reads: try hot first, fall back to warm. Writes: hot immediately, warm asynchronously.

**Failure handling:** If Redis fails, degrade to warm-only retrieval (slower but functional). If PostgreSQL shard fails, only affected users are impacted.

**Cost estimate at scale:**
- Redis (1 TB): ~$5K/month
- PostgreSQL (50 TB, sharded): ~$25K/month
- Pinecone vector store: ~$100K/month (biggest line item)
- S3 cold storage: ~$1K/month
- **Total: ~$130K/month for 10M users = $0.013/user/month**

**Interview signal:** Discussing the vector store as the primary cost driver shows awareness of production economics.

---

## Q2. Design a Multi-Tenant Memory Architecture for an Enterprise SaaS Product ⭐⭐⭐⭐

**Prompt:** "Design memory for an enterprise SaaS where 1000+ companies each have their own users. Company A's memories must NEVER be visible to Company B. Support tenant-specific memory policies."

**Isolation architecture:**

```
Tenant hierarchy:
  Organization (Company)
    └── Users (Members of company)
         └── Memories (User's data)

Isolation rules:
  - Users see ONLY their own memories
  - Admins in a company see aggregated stats, not individual memories
  - No user in Company A can EVER see Company B's data
  - Deletion of a company deletes ALL derived data
```

**Physical isolation strategies (ranked by security level):**

**Level 1 — Logical isolation (namespace filtering):**
- Single database. Every table has `tenant_id`, `user_id`.
- Query middleware auto-injects tenant filter.
- Cheapest. Weakest isolation.
- Suitable for: SMB customers.

**Level 2 — Schema isolation (per-tenant schemas):**
- One database, one schema per tenant.
- Queries route to correct schema based on tenant.
- Better isolation. More operational overhead.
- Suitable for: mid-market.

**Level 3 — Database isolation (per-tenant DB):**
- Separate database per tenant.
- Physical data separation.
- Highest cost. Best isolation.
- Suitable for: enterprise, regulated industries.

**Level 4 — Region isolation:**
- Tenant data physically located in specific region (EU data in EU).
- For regulatory compliance (GDPR data residency).
- Extra: at rest and in transit encryption per region.

**Tenant-specific memory policies:**

Each tenant configures:
- **Retention:** How long to keep memories (7 days? 12 months? Forever?)
- **Extraction:** What types of facts to extract into entity memory
- **Sensitive fields:** Fields to redact or exclude (PII policies)
- **Shared memory:** Do users within a tenant share any memory? (e.g., "our company's product catalog")
- **Right-to-delete SLA:** Some tenants require <24h deletion

**Configuration schema:**
```
{
  "tenant_id": "acme_corp",
  "isolation_level": "database",
  "region": "eu-west",
  "retention_days": 365,
  "extraction_config": {
    "extract_entities": ["role", "team", "projects"],
    "redact_patterns": ["ssn", "credit_card"]
  },
  "deletion_sla_hours": 24,
  "shared_memory_enabled": true,
  "audit_logging": "detailed"
}
```

**Critical safeguards:**

1. **No query without tenant_id.** Middleware enforces this.
2. **Automated cross-tenant tests.** Run in CI: attempt to retrieve tenant A's data as tenant B. Must fail.
3. **Audit log per tenant.** Detailed access logs, retained per tenant's policy.
4. **Regular penetration testing.** External security firm attempts cross-tenant access.

**Interview signal:** Naming the four isolation levels AND their trade-offs shows enterprise architecture experience.

---

## Q3. Design a Memory Governance System (GDPR, CCPA, HIPAA) ⭐⭐⭐⭐

**Prompt:** "Design memory governance for a healthcare AI product. Must comply with HIPAA, GDPR, and be auditable by regulators."

**Governance requirements:**

**1. Data classification.**
Every memory field has a classification:
- PHI (Protected Health Information) → strictest handling
- PII (Personally Identifiable Information) → strong handling
- Personal preferences → standard handling
- Anonymous facts → minimal handling

Different classifications have different: retention, encryption, access controls.

**2. Consent management.**
- Explicit consent before memory storage
- Granular consent (allow entity memory, deny semantic memory)
- Consent audit trail
- Consent revocation mechanism

**3. Access control.**
- Users see their own data
- Providers see their patients' relevant data (with patient consent)
- Admins see aggregated stats, never raw memories
- Auditors have read-only access with logging
- Every access is logged

**4. Encryption.**
- At rest: AES-256 per-tenant keys
- In transit: TLS 1.3
- Field-level encryption for highly sensitive fields
- Key rotation quarterly

**5. Deletion (Right to be Forgotten).**
- User requests deletion → deletion pipeline runs
- Delete from: entity DB, vector store, caches, backups, analytics, downstream systems
- Confirmation with timestamp
- SLA: complete within 30 days (GDPR), 45 days (CCPA)
- Retain deletion audit trail (proof of deletion)

**6. Data export (Right to Access).**
- User requests export → structured JSON of all memories
- Include: entities, semantic memories, episodic events, summaries
- Delivered via secure download link with expiration
- SLA: within 30 days

**7. Audit trail.**
- Every memory read/write logged
- Log includes: user_id, actor (user? system?), operation, memory_id, timestamp, source (which conversation)
- Logs retained for 7 years (HIPAA) 
- Logs are themselves subject to access control

**8. Data residency.**
- EU users' memories stored in EU
- US users' memories stored in US
- Cross-region access requires explicit consent
- Data never crosses regulatory boundaries without justification

**Architecture:**

```
┌──── Consent Service ────┐
│ Records consent grants   │
│ Blocks non-consented ops │
└──────────┬───────────────┘
           │ Every operation checked
           ▼
┌──── Memory Operations ──┐
│ CRUD on memories         │────► Audit Log Service
└──────────┬───────────────┘      (immutable, encrypted)
           │
           ▼
┌──── Classification Layer ┐
│ Tags fields by sensitivity│
│ Applies encryption rules  │
└──────────┬───────────────┘
           │
           ▼
┌──── Storage (encrypted) ─┐
│ Region-specific           │
│ Tenant-isolated           │
└──────────────────────────┘
```

**Regulatory readiness:**
- **Documentation:** Every policy documented, versioned
- **DPO (Data Protection Officer):** Designated role, tooling
- **DPIA (Data Protection Impact Assessment):** Documented for each memory feature
- **Breach response plan:** Procedures for a memory data breach
- **Regular audits:** External audits annually

**Interview signal:** Discussing DPIA and breach response shows regulatory maturity. Most candidates only know "we encrypt."

---

## Q4. Design a Monitoring & Observability System for Production Memory ⭐⭐⭐⭐

**Prompt:** "Design monitoring for a production memory system. Detect issues early. Ensure trust and quality."

**Monitoring dimensions:**

**1. Health metrics (system operational).**
- Retrieval latency (p50, p95, p99 per memory type)
- Write latency
- Error rate per memory type
- Storage size (per user, per tenant, total)
- Storage growth rate

**2. Quality metrics (memory correctness).**
- Retrieval accuracy (sampled evaluations)
- Fact accuracy (audit-verified)
- User correction rate ("no, that's wrong")
- Contradiction detection rate
- Staleness distribution (age of memories being used)

**3. Cost metrics (financial health).**
- Cost per user per month
- Cost per memory type
- Vector store cost trend
- LLM cost (for compression, extraction)
- Cost efficiency: cost per useful retrieval

**4. Security metrics (breach detection).**
- Cross-tenant access attempts (must be 0)
- Failed auth on memory operations
- Anomalous access patterns
- Unusual bulk operations
- Retention policy violations (memories older than allowed)

**5. Compliance metrics (regulatory).**
- Deletion requests received / completed
- Time-to-deletion (SLA compliance)
- Data export requests
- Consent revocations
- Audit log completeness

**6. User experience metrics (trust).**
- Memory disclosure interactions (user opens "what do you know about me")
- Memory delete-all events
- User complaints about memory
- Session length before/after memory usage

**Alerts to configure:**

| Metric | Threshold | Action |
|---|---|---|
| Cross-tenant access | > 0 | Immediate page, security incident |
| Retrieval latency p95 | > 500ms | Investigate: index degradation? |
| User's memory grows | > 10x normal in 1 day | Check for abuse or bug |
| Fact accuracy on audit | < 90% | Investigate extraction quality |
| Deletion SLA | > 24h | Compliance risk, escalate |
| Vector store cost | > 20% MoM growth | Cost review |
| Contradiction rate | > 5% | Memory quality degrading |
| Failed authentication on memory ops | > baseline | Potential attack |

**The unified dashboard:**

Top row — real-time system health (latency, error rate, storage size).
Middle row — quality signals (accuracy, contradictions, staleness).
Bottom row — cost and compliance status.

**Continuous evaluation pipeline:**

Daily:
- Sample 1000 memory retrievals
- Verify via LLM-as-judge
- Track precision, recall, faithfulness
- Alert on quality regression

Weekly:
- Sample 100 memories for fact accuracy audit
- Human-in-the-loop verification for high-value tenants
- Report to leadership

**Interview signal:** Distinguishing between operational health and quality metrics shows sophisticated observability. Most candidates only monitor latency.

---

## Q5. Design a Distributed State System for Multi-Server AI Agents ⭐⭐⭐⭐

**Prompt:** "Your agent workflow runs across multiple servers (load balancer routes to any instance). How do you maintain state for a long-running conversation?"

**The problem:**
- User A's first message → Server 1
- User A's second message → Server 3
- Without shared state, Server 3 has no knowledge of the previous turn.

**Solution architectures:**

**Architecture 1 — Sticky sessions (simplest):**
Load balancer routes based on user_id. Same user always hits same server.
- Simple. State stays local.
- Weaknesses: uneven load, single point of failure per user, hard to scale.

**Architecture 2 — Externalized state (recommended):**
State lives in Redis/PostgreSQL. Any server can serve any request.
- Requires: fetch state at start of each request, persist state at end.
- Adds ~5-20ms latency per request.
- Scales horizontally.

**Architecture 3 — Event sourcing:**
State is derived from an event log. Every action produces an event. State reconstructed by replaying events.
- Full audit trail. Immutable.
- Slower reads (must replay). Higher storage.

**For AI agents specifically:**

**LangGraph with PostgresSaver:**
```python
from langgraph.checkpoint.postgres import PostgresSaver

# Every state transition automatically checkpointed
checkpointer = PostgresSaver.from_conn_string("postgresql://...")
graph = workflow.compile(checkpointer=checkpointer)

# Server 1 handles turn 1
config = {"configurable": {"thread_id": "user_a_session_1"}}
result_1 = graph.invoke({"query": "..."}, config=config)

# Server 3 handles turn 2 — loads state from Postgres
result_2 = graph.invoke({"query": "..."}, config=config)
# ↑ Automatically loads previous state via thread_id
```

**Concurrency handling:**

**Problem:** User sends 2 messages quickly. Server 1 processes msg 1, Server 5 processes msg 2. Both try to update state simultaneously.

**Solutions:**

**Optimistic locking:**
- Each state has a version number
- Update includes expected current version
- If version mismatch (someone else updated), retry
- Best for: low contention scenarios

**Pessimistic locking:**
- Acquire lock before reading state
- Hold lock during update
- Release lock after
- Best for: high contention, want strict ordering

**Message queue:**
- All user messages enter a queue
- Single worker per user (based on user_id hash)
- Guarantees serial processing per user
- Best for: guaranteed ordering, but adds latency

**The production pattern:**

Externalized state (PostgreSQL/Redis) + optimistic locking + retry on conflict. Falls back to queue-based processing if conflicts are frequent.

**Interview signal:** Discussing concurrency (not just persistence) shows distributed systems maturity.

---

## Q6-Q15: Additional System Design Questions (Condensed)

### Q6. Design a Memory Feedback Loop for Continuous Improvement ⭐⭐⭐
Users provide feedback (thumbs up/down) on responses. When memory-informed responses get downvoted, analyze: was memory retrieval wrong? Was extraction wrong? Weekly automated analysis. Improvements: rebalance retrieval weights, retrain extraction, adjust confidence thresholds.

### Q7. Design a Memory Sharing System for Team Collaboration ⭐⭐⭐⭐
Team members share memories about a project. User A's insight benefits User B on the same project. Design: per-project shared memory + per-user private memory. Access control per project. Attribution ("Alice mentioned this"). Privacy: user can opt-out of sharing.

### Q8. Design an Auto-Expiring Memory Policy Engine ⭐⭐⭐
Different memory types have different lifecycles. Some facts stay forever (name), others expire (current project). Policy engine: rule-based (`{key: "current_project", ttl_days: 90}`), LLM-assisted (LLM classifies fact type → applies policy), user-controlled (user marks memories as "keep forever" or "temporary").

### Q9. Design a Memory Migration System (SQLite → PostgreSQL) ⭐⭐⭐⭐
Growing SaaS. Started on SQLite. Now needs PostgreSQL for scale. Design zero-downtime migration: dual-write phase, backfill historical data, shadow-read validation, gradual cutover per tenant, rollback plan.

### Q10. Design a Memory Compression Service ⭐⭐⭐
As memory grows, compress older memories to reduce cost. Service: identify compression candidates (age, low access), batch compress via LLM, verify quality post-compression, replace original with compressed, keep audit trail. Runs as background job.

### Q11. Design a Personal AI With Memory (like ChatGPT Memory) ⭐⭐⭐⭐
Persistent per-user memory across all conversations. Memory extraction on every turn. User-facing memory management UI. Explicit fact confirmation. Category filtering. Import/export. Search across memories. The whole product design, not just backend.

### Q12. Design an Audit-Ready Memory System for Financial AI ⭐⭐⭐⭐
Every memory access must be auditable. Regulatory requirement: prove who saw what and when. Immutable audit log. Cryptographic tamper evidence. External auditor read-only access. 7-year retention. GDPR-compliant deletion of user data (not audit logs).

### Q13. Design a Memory Sync for Multi-Device AI Assistant ⭐⭐⭐
User uses assistant on phone AND laptop. Memory should sync. Design: cloud-first source of truth, device-side cache, conflict resolution (last-write-wins with device precedence), offline support with sync on reconnect.

### Q14. Design a Memory Quality Assurance Pipeline ⭐⭐⭐
CI/CD for memory quality. Test suite: golden queries + expected retrievals + expected responses. Every deploy runs suite. Regression detection. Human-in-the-loop for ambiguous cases. Quarterly audit reports.

### Q15. Design a Rate-Limited Memory API for Third Parties ⭐⭐⭐
Expose memory API to third-party plugins. Rate limits per API key. Scoped access (some plugins see entities only, some see semantic). Per-user consent for third-party memory access. Audit log of third-party access.
