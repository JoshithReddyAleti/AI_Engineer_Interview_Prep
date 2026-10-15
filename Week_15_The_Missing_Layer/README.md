# 🏛️ Week 15 — The Missing Layer: Architecture + Cost + Reasoning + Enterprise Integration

> **Maps to:** [Episode_15_The_Missing_Layer](https://github.com/JoshithReddyAleti/Episode_15_The_Missing_Layer)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)
>
> **This is the closer.** Week 15 covers the four architectural gaps between "AI prototype that got 40K likes" and "enterprise feature that ships to a Fortune 500 customer."

---

## Why This Week Matters More Than Any Other

Weeks 1-14 taught you HOW LLMs work, how to RAG them, how to fine-tune them, how to evaluate them, how to deploy them, how to secure them, how to A/B test them.

Week 15 teaches you the four things that decide whether your AI feature actually *ships to enterprise customers*:

1. **Application architecture** — the meta-layer before the prompt
2. **Cost engineering** — unit economics that survive scale
3. **Reasoning & test-time compute** — when o1/o3/R1 pattern wins vs when it burns money
4. **Enterprise integration & compliance** — the SharePoint problem, SAML, SOC 2, HIPAA, GDPR, data residency

**Signal calibration (what interviewers detect):**

- **Junior:** "We used ChatGPT API and it worked."
- **Mid:** "We built a RAG system with LangChain."
- **Senior:** "We architected shared-tier for SMB and dedicated-tier for enterprise, with row-level security syncing SharePoint ACLs to our retrieval layer."
- **Staff+ / Frontier Lab:** "For our multi-tenant SaaS with 10K tenants, we chose shared model + logical isolation for the free tier ($0.003/user/day) and dedicated inference pods with tenant-specific KV cache namespacing for enterprise ($4.20/user/day). BAA-covered providers (Anthropic, OpenAI enterprise, Azure OpenAI) for regulated tenants. Row-level security enforced via pre-retrieval ACL filter with 4-layer audit — the SharePoint leak class is impossible by construction, not by hope."

Week 15 gets you to the staff+ / frontier-lab bar across all four discipline gaps.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 32 | Multi-tenancy architecture, cost model derivation, reasoning models (o1/o3/R1), best-of-N log-scaling, PRMs, SharePoint problem, SAML/OIDC/SCIM, SOC 2/HIPAA/GDPR mapping, data residency |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 17 | Semantic cache with tenant isolation, cost calculator, best-of-N with verifier, ACL-aware retrieval, audit event schema with hash chain, streaming SSE handler |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | Multi-tenant SaaS AI platform, cost-tiered LLM serving with break-even math, reasoning gateway routing, permission-aware RAG across SharePoint/Confluence/Salesforce, GDPR right-to-erasure workflow, EU AI Act compliance |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 17 | $180K/month cost explosion, reasoning model 30× more expensive than expected, cross-tenant data leak in production, SOC 2 audit finding, GDPR request paralyzed team for 2 weeks, EU AI Act blocked launch |
| [05_Terminology_Differentiation.md](05_Terminology_Differentiation.md) | Reference | Shared vs pooled vs dedicated tenancy, prompt caching vs semantic caching vs KV caching, reasoning vs chain-of-thought vs test-time compute, SAML vs OIDC vs SCIM, RBAC vs ABAC vs ReBAC, residency vs sovereignty, SOC 2 vs ISO 27001, GDPR vs CCPA vs LGPD |

## The Four Folder Coverage

### Folder 1 — LLM Application Architecture Patterns

| Sub-section | Interview Territory |
|---|---|
| Application shapes | Chat, agent, workflow, embedded, batch — when each |
| Multi-tenancy | Shared / pooled / dedicated / silo — cost & isolation trade-offs |
| Streaming architecture | SSE vs WebSocket vs polling, backpressure, reconnection, partial-response handling |
| Backend-for-Frontend | Why the LLM API cannot be the app API, prompt injection surface, latency shaping |
| Session & multi-device | Conversation state, cross-device continuity, memory scope, TTL policies |

### Folder 2 — Cost Engineering

Cost per unit (per user, per query, per session, per feature). Semantic caching hit rates. Prompt caching (Anthropic 2024). Model tiering (Haiku for classification, Sonnet for generation, Opus for reasoning). Self-hosting break-even math (~15-30M tokens/hour sustained). Batch vs streaming cost trade-offs. Cost attribution for chargeback. Budget alerting and hard cutoffs.

### Folder 3 — Reasoning & Test-Time Compute

**Reasoning models:** o1/o3 (OpenAI 2024), DeepSeek-R1 (DeepSeek-AI 2024), Claude thinking. Trained via RL on chain-of-thought with verifiable rewards.

**Test-time compute patterns:** best-of-N with log-scaling law (Cobbe 2021), self-consistency (Wang 2022), iterative refinement, tree search (ToT — Yao 2023), MCTS (Xie 2024), Process Reward Models (Uesato 2022, Lightman 2023 "Let's Verify Step by Step").

**Cost multiplier reality:** 10-100× baseline cost per query. When justified: legal (70% cost reduction end-to-end vs manual review), complex agent planning (53% cost win). When not: casual chat, simple classification, high-throughput.

**Verifier ceiling:** best-of-N accuracy is bounded above by verifier quality. Weak verifier = wasted compute. This is the capability ceiling most teams miss.

### Folder 4 — Enterprise Integration & Compliance

**Identity & access:** SAML 2.0 (OASIS 2005), OIDC (OpenID Foundation 2014), SCIM 2.0 (RFC 7643/7644), RBAC/ABAC/ReBAC hybrid, zero-trust for LLM.

**Row-level security & the SharePoint problem:** naive RAG leaks documents the user cannot see in the source system. Four-layer enforcement: source ACL sync → index-time metadata → query-time pre-filter → post-retrieval audit.

**Compliance frameworks:** SOC 2 CC controls, HIPAA §164.312 with provider BAA landscape (Anthropic, OpenAI enterprise, Azure OpenAI, AWS Bedrock all offer BAAs; consumer APIs do not), GDPR articles 15/17/20, PCI DSS tokenization, FedRAMP Moderate/High paths, EU AI Act risk tiers (unacceptable/high/limited/minimal), US state laws (Colorado AI Act, CCPA extensions).

**Data residency & sovereignty:** the 15-layer data location table (client, edge, gateway, prompt store, vector DB, cache, provider inference, provider training, logs, backups, DR, analytics, SIEM, audit, cold archive). Transfer instruments (adequacy, SCCs, BCRs). Residency vs sovereignty distinction. Provider routing tables (2026 baseline).

**Audit trails:** complete audit event schema, tamper-evident storage (hash chain, WORM, blockchain-anchored), retention matrices per regulation, GDPR right-to-erasure workflows across multi-system topology.

## Cost of the Missing Layer

- **Wrong tenancy model:** picked shared when isolation was required. Cross-tenant leak → breach notification → churn. $2M-$20M incident cost.
- **No cost engineering:** $180K/month LLM bill nobody predicted. Feature killed. Team blamed.
- **Reasoning models used blindly:** 30× cost increase for tasks where reasoning didn't help. Or under-used where it would have saved 70%.
- **No enterprise integration:** stuck at pilot with SMB customers. Cannot land Fortune 500 without SOC 2, SAML, BAA. Revenue ceiling.
- **Compliance gaps:** SOC 2 audit finding. HIPAA violation. GDPR fine (up to €20M or 4% global revenue). EU AI Act penalties (up to €35M).

Every gap in Week 15 has direct revenue and dollar consequences. The teams that fill these gaps land enterprise deals. The teams that don't stay in pilot forever.

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1-14 | LLM Fundamentals → The Discipline Layer | (see main README) | ✅ |
| **15** | **The Missing Layer: Architecture + Cost + Reasoning + Enterprise** ← you are here | [Episode_15_The_Missing_Layer](https://github.com/JoshithReddyAleti/Episode_15_The_Missing_Layer) | ✅ |

**Week 15 is the final and closing week of the AI Engineering Roadmap 2026.** From here, the series bridges to the **Agentic AI Roadmap 2027**.
