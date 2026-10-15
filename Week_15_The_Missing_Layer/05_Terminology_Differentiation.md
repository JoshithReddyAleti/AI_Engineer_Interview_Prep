# 📖 Week 15 — Terminology Differentiation Reference

> **Focus:** Every set of similar-sounding Missing Layer terms across application architecture, cost engineering, reasoning models, and enterprise integration & compliance.
>
> **Why it matters:** these distinctions map to real architectural decisions. Confusing shared with pooled costs 10× isolation properties. Confusing residency with sovereignty loses EU customers. Confusing reasoning with chain-of-thought misroutes queries to 20× cost.

---

# Section 1: Multi-Tenancy Models

## Shared vs Pooled vs Dedicated vs Silo

### Shared Everything

**Definition:** all tenants share infrastructure; logical isolation only.

**Cost:** cheapest (~$0.003-$0.05/user/day).

**Isolation:** application code enforces tenant_id filters. Trust boundary is code correctness.

**Use for:** free tier, prosumer, non-sensitive workloads.

---

### Pooled with Logical Isolation

**Definition:** shared compute + enforced namespace isolation (cache keys, index collections, audit trails).

**Cost:** low (~$0.005-$0.10/user/day).

**Isolation:** logical + enforced at every layer (not just filter but keyspace).

**Use for:** standard B2B SaaS.

---

### Dedicated Inference / Pooled Data Plane

**Definition:** per-tenant model endpoints; shared but sharded data plane.

**Cost:** higher (~$0.50-$5/user/day).

**Isolation:** compute + data separation. Noisy neighbor eliminated.

**Use for:** enterprise, regulated verticals, per-tenant fine-tuning.

---

### Silo (Full Isolation)

**Definition:** per-tenant everything: VPC, model, storage, cache, audit.

**Cost:** very high (~$2-$50/user/day or flat $10K-$100K/mo/tenant).

**Isolation:** complete.

**Use for:** healthcare (HIPAA + BAA per instance), defense, financial services, tenants requiring in-region deployment.

---

### 🎯 The Distinction That Matters

| Model | Isolation | Cost | Compliance Fit |
|---|---|---|---|
| **Shared** | Logical | Cheapest | Free tier |
| **Pooled** | Logical + enforced | Low | SOC 2 SaaS |
| **Dedicated** | Compute + data | High | HIPAA, enterprise |
| **Silo** | Complete | Very high | Sovereign, air-gap |

---

# Section 2: Caching Types

## Prompt Cache vs Semantic Cache vs KV Cache vs Response Cache

### Prompt Cache (Anthropic 2024, OpenAI 2024)

**What it caches:** KV state of common prompt prefix at provider side.

**Savings:** 90% cost reduction on cached tokens.

**Scope:** per-provider-account, TTL 5 min typical.

**Use for:** fixed system prompts, few-shot examples reused across queries.

---

### Semantic Cache

**What it caches:** full query→response pairs.

**Match:** cosine similarity of query embedding to cached queries.

**Savings:** 100% cost on hit (no LLM call).

**Scope:** application-managed, must namespace per tenant.

**Use for:** repeated queries with similar answers (support, FAQ).

---

### KV Cache (Inference-Level)

**What it caches:** attention key/value tensors during generation.

**Scope:** within a single generation. Enables next-token generation without recomputing.

**Use for:** all LLM inference — this is table-stakes at the model serving layer.

---

### Response Cache (Full Match)

**What it caches:** exact-match query→response.

**Match:** deterministic string hash.

**Savings:** 100% on hit.

**Use for:** truly repeated queries (rare in chat, common in batch).

---

### 🎯 The Distinction That Matters

| Cache | Match | Managed | Savings |
|---|---|---|---|
| **Prompt cache** | Prefix hash | Provider-side | 90% cached tokens |
| **Semantic cache** | Vector similarity | App-side | 100% on hit |
| **KV cache** | Position | Inference-level | Enables generation |
| **Response cache** | Exact hash | App-side | 100% on hit |

**These stack.** Prompt cache reduces per-query cost; semantic cache avoids the call entirely; both together produce 60-70% total cost reduction on typical workloads.

---

# Section 3: Reasoning Concepts

## Reasoning Models vs Chain of Thought vs Test-Time Compute

### Chain of Thought (Wei 2022)

**Definition:** prompting technique. Base LLM generates reasoning steps before answer.

**Cost:** ~1x baseline (slightly more output tokens).

**Availability:** any LLM.

---

### Reasoning Models (o1/o3/R1/Claude thinking, 2024-2025)

**Definition:** LLMs trained via RL on verifiable rewards to reason internally before answering.

**Cost:** 10-100× baseline per query (thinking tokens + output tokens).

**Availability:** specific models (o1, o3, DeepSeek-R1, Claude with extended thinking).

---

### Test-Time Compute

**Definition:** broader family of techniques trading inference compute for quality: best-of-N, self-consistency, tree search, iterative refinement.

**Cost:** N× baseline where N is samples/branches.

**Availability:** applied at application layer over any LLM.

---

### 🎯 The Distinction That Matters

| Concept | What It Is | Cost | When |
|---|---|---|---|
| **CoT** | Prompting technique | ~1x | Always cheap to try |
| **Reasoning model** | RL-trained model | 10-100× | Hard reasoning tasks |
| **Test-time compute** | Family of techniques | N× | Verifiable outputs |

**Interview trap:** conflating these. Reasoning models are a subset of test-time compute done at model level. CoT is prompt-level. You can combine: reasoning model + best-of-N + PRM verifier is the state of the art on math benchmarks.

---

# Section 4: Identity Protocols

## SAML vs OIDC vs SCIM

### SAML 2.0 (OASIS 2005)

**What it does:** SSO. Identity Provider asserts user identity via signed XML.

**Age:** legacy but still dominant enterprise.

**Endpoint style:** browser POST binding, redirect binding.

**Use for:** enterprise SSO integration.

---

### OIDC (OpenID Foundation 2014)

**What it does:** SSO built on OAuth 2.0. Returns ID Token (JWT).

**Age:** modern.

**Endpoint style:** REST + JSON.

**Use for:** modern SSO, mobile apps, developer-friendly integration.

---

### SCIM 2.0 (RFC 7643/7644, 2015)

**What it does:** user/group provisioning. IdP pushes user lifecycle events to SaaS.

**Endpoint style:** REST /Users, /Groups.

**Use for:** automated user provisioning, deprovisioning on employee departure.

---

### 🎯 The Distinction That Matters

| Protocol | Purpose | Direction |
|---|---|---|
| **SAML** | Authentication (SSO) | IdP → SaaS on login |
| **OIDC** | Authentication (SSO) | IdP → SaaS on login |
| **SCIM** | Provisioning | IdP → SaaS on lifecycle events |

**SAML and OIDC are alternatives.** SCIM is complementary. Enterprise SaaS supports all three: SAML or OIDC for login, SCIM for user sync.

---

# Section 5: Access Control Models

## RBAC vs ABAC vs ReBAC

### RBAC (Role-Based Access Control)

**Model:** users have roles; roles have permissions.

**Example:** 'Sales' role → read customer records.

**Strength:** simple, well-understood.

**Weakness:** coarse. Hard to model fine-grained sharing.

---

### ABAC (Attribute-Based Access Control)

**Model:** access rules use attributes (user, resource, context).

**Example:** 'user in region=EU can access resources with region=EU during business hours.'

**Strength:** flexible, context-aware.

**Weakness:** complex to reason about, hard to audit.

---

### ReBAC (Relationship-Based Access Control)

**Model:** access based on relationships (Google Zanzibar).

**Example:** 'user is member of folder that is shared with document owner.'

**Strength:** fine-grained sharing (SharePoint, Google Drive model).

**Weakness:** requires graph engine.

---

### 🎯 The Distinction That Matters

| Model | Granularity | Complexity | Best For |
|---|---|---|---|
| **RBAC** | Coarse | Low | Enterprise roles |
| **ABAC** | Context-aware | Medium | Compliance rules |
| **ReBAC** | Fine-grained sharing | High | Document collaboration |

**Enterprise reality:** hybrid. RBAC for coarse (department roles), ABAC for context (data classification, geography), ReBAC for document-level.

---

# Section 6: Compliance Frameworks

## SOC 2 vs ISO 27001 vs HIPAA vs GDPR

### SOC 2 (AICPA)

**Scope:** service organization controls for security, availability, confidentiality, processing integrity, privacy.

**Nature:** American, audit-based, Type 1 (point-in-time) or Type 2 (period).

**Use for:** table stakes for enterprise SaaS in US.

---

### ISO 27001

**Scope:** information security management system.

**Nature:** international, certification-based.

**Use for:** international customer trust, EU market.

---

### HIPAA (US Healthcare)

**Scope:** protected health information handling.

**Requirements:** BAA with any downstream processor, technical safeguards (§164.312).

**Use for:** healthcare workloads.

---

### GDPR (EU)

**Scope:** any EU person's personal data.

**Requirements:** legal basis, individual rights (Articles 15/17/20/22), 72-hour breach notification.

**Penalties:** up to €20M or 4% global revenue.

---

### 🎯 The Distinction That Matters

| Framework | Scope | Certification | Applicability |
|---|---|---|---|
| **SOC 2** | Service orgs | Audit report | Enterprise SaaS US |
| **ISO 27001** | Info sec mgmt | Certification | Global |
| **HIPAA** | PHI | BAA + safeguards | Healthcare |
| **GDPR** | EU personal data | Regulation | Any EU data touch |

Also relevant: PCI DSS (payment cards), FedRAMP (US federal), EU AI Act (AI-specific), CCPA/CTDPA/VCDPA (US state laws).

---

# Section 7: Data Location Concepts

## Residency vs Sovereignty vs Localization

### Data Residency

**Definition:** data physically stored in specified jurisdiction.

**Requirement:** infrastructure geography.

---

### Data Sovereignty

**Definition:** data subject to jurisdiction's laws, protected from other jurisdictions' access.

**Requirement:** residency + legal + operational + contractual controls (e.g., US CLOUD Act protection).

---

### Data Localization

**Definition:** legal requirement that data be stored in-country (e.g., Russia, China).

**Requirement:** infrastructure geography + often prohibition on export.

---

### 🎯 The Distinction That Matters

| Concept | What It Requires |
|---|---|
| **Residency** | Where data is stored |
| **Sovereignty** | Whose laws apply + who can access |
| **Localization** | Legally mandated in-country |

**Enterprise EU customers increasingly demand sovereignty, not just residency.** Post-Schrems II, US-based providers must show data cannot be reached under US CLOUD Act.

---

# Section 8: Cross-Border Transfer Instruments

## Adequacy vs SCCs vs BCRs vs Derogations

### Adequacy Decisions

**Definition:** EU declares a country's data protection adequate.

**Examples:** UK, Japan, Argentina, US (via Data Privacy Framework).

**Effect:** no additional safeguards required.

---

### Standard Contractual Clauses (SCCs)

**Definition:** EU-approved contract templates for transfers to non-adequate jurisdictions.

**Post-Schrems II:** requires supplementary measures (encryption, pseudonymization).

---

### Binding Corporate Rules (BCRs)

**Definition:** internal rules within a multinational's group.

**Approval:** supervisory authority.

**Use for:** intragroup transfers only.

---

### Derogations

**Definition:** limited exceptions (consent, contract necessity, public interest).

**Scope:** limited use, not for systematic transfers.

---

### 🎯 The Distinction That Matters

| Instrument | Approval | Use Case |
|---|---|---|
| **Adequacy** | EU Commission | Certain jurisdictions |
| **SCCs** | Template | Most cross-border business |
| **BCRs** | Supervisory authority | Intragroup only |
| **Derogations** | Case-by-case | Exceptional |

---

# Section 9: Streaming Protocols

## SSE vs WebSocket vs Long Polling vs gRPC Streaming

### Server-Sent Events (SSE)

**Direction:** unidirectional (server → client).

**Complexity:** simple.

**Firewall friendliness:** high (HTTP).

**Use for:** LLM token streaming, notifications.

---

### WebSocket

**Direction:** bidirectional persistent connection.

**Complexity:** medium.

**Firewall friendliness:** moderate (some blocked).

**Use for:** agent workflows requiring interruption, real-time collaboration.

---

### Long Polling

**Direction:** effectively client-initiated pull.

**Complexity:** medium.

**Firewall friendliness:** high.

**Use for:** fallback when SSE/WS blocked.

---

### gRPC Streaming

**Direction:** unidirectional or bidirectional.

**Complexity:** higher (Protobuf, HTTP/2).

**Firewall friendliness:** low in browsers.

**Use for:** internal service-to-service streaming.

---

### 🎯 The Distinction That Matters

| Protocol | Direction | Simplicity | Best For |
|---|---|---|---|
| **SSE** | S→C | Simplest | LLM token streaming |
| **WebSocket** | Bidir | Medium | Agents with interruption |
| **Long polling** | C→S | Medium | Firewall-restricted |
| **gRPC** | Either | Complex | Internal services |

**For LLM token streaming: SSE is the standard.** WebSocket only when bidirectional is required.

---

# Section 10: Cost Optimization Levers

## Prompt Cache vs Semantic Cache vs Model Tiering vs Self-Hosting

### Prompt Cache

**Savings:** 30-50% on typical RAG.

**Effort:** low (provider-side, enable via API).

**Constraints:** requires common prompt prefixes.

---

### Semantic Cache

**Savings:** 20-50% depending on hit rate.

**Effort:** moderate (app-side infrastructure).

**Constraints:** requires similarity threshold tuning; per-tenant namespacing.

---

### Model Tiering

**Savings:** 40-60% via router.

**Effort:** moderate (classifier training or heuristics).

**Constraints:** quality validation per tier.

---

### Self-Hosting

**Savings:** 30× per output token at scale.

**Effort:** high (ops, monitoring, capacity planning).

**Constraints:** requires sustained volume (15-30M output tokens/hour).

---

### 🎯 The Distinction That Matters

| Lever | Savings | Effort | Constraint |
|---|---|---|---|
| **Prompt cache** | 30-50% | Low | Common prefixes |
| **Semantic cache** | 20-50% | Medium | Similar queries |
| **Model tiering** | 40-60% | Medium | Router quality |
| **Self-hosting** | 30× | High | Sustained volume |

**These stack.** Full-stack cost engineering: prompt cache + semantic cache + model tiering can achieve 60-70% cost reduction on standard RAG workloads. Self-hosting on top adds another 30× at sufficient scale.

---

## 🎯 The Meta-Lesson

The Missing Layer has terminology that trips up 90% of candidates because it spans four disciplines. Shared isn't 'a cheap tenancy' — it's a specific isolation model with specific trust boundaries. Reasoning models aren't 'better' — they're a spending decision with specific ROI conditions. Residency isn't 'the same as sovereignty' — one is where, one is who can reach.

Loose terminology → loose thinking → wrong architectural choice → security incident, cost blowup, compliance violation, or lost enterprise deal.

Precise terminology → precise thinking → pooled tenancy for SMB + silo for enterprise + reasoning router for high-value queries + sovereignty option for EU + 9-layer defense stack + CUPED for the A/B test that measures it all → durable systems that ship to Fortune 500.

The vocabulary IS the seniority signal. And Week 15's vocabulary IS the enterprise architect signal.
