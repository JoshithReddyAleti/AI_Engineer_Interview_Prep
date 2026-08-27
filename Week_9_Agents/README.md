# 🤖 Week 9 — Agents: When AI Systems Make Decisions

> **Maps to:** [Episode_9_Agents_When_AI_Systems_Make_Decisions](https://github.com/JoshithReddyAleti/Episode_9_Agents_When_AI_Systems_Make_Decisions)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

**Agents are where AI stops being a chatbot and starts being a system.** They're also where interviewers separate senior candidates from principals.

A chain executes a plan. An agent DECIDES the plan. That single shift changes everything — cost model, debugging surface, security posture, evaluation methodology, failure modes, and enterprise risk. Every question in this week probes whether you understand the delta.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 27 | Foundations, all 7 reasoning patterns, tool use, planning, multi-agent coordination, autonomy spectrum, safety, framework comparison |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 16 | Build ReAct from scratch, tool selection, multi-agent coordinator, HIL agent, circuit breakers, cost/loop guards, LangGraph state machines |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 16 | Enterprise agent platform, browser agent security, coding agent architecture, multi-agent orchestration, production observability, kill switches |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 16 | Agent cost explosions, infinite loops, unauthorized actions, multi-agent deadlocks, framework migrations, "agent went rogue" incidents |

## The 14 Sections Covered (Full Episode 9)

| # | Section | Key Enterprise Questions |
|---|---|---|
| 1 | Foundations | When agent vs chain vs workflow? |
| 2 | Reasoning Patterns | ReAct vs Plan-and-Execute vs Reflexion — when? |
| 3 | Tool Use | How to design tools agents won't misuse? |
| 4 | Planning | Task decomposition + plan revision strategies |
| 5 | Single-Agent Architectures | 6 canonical patterns and their trade-offs |
| 6 | Multi-Agent Systems | Coordinator vs peer-to-peer vs hierarchical |
| 7 | Agent State & Memory | Working, episodic, persistent (vs Week 7) |
| 8 | Control Flow | Termination, loops, HIL, approval gates |
| 9 | Autonomous Agents | Goal-setting, self-correction, boundaries |
| 10 | Production Agents | Observability, cost caps, circuit breakers, rate limits |
| 11 | Safety & Alignment | Action boundaries, permissions, kill switches, rollback |
| 12 | Frameworks | LangGraph, CrewAI, AutoGen, Swarm, LlamaIndex, SmolAgents |
| 13 | Real-World Agents | Research, coding, support, analysis, browser, workflow |
| 14 | Utils | Common patterns across the stack |

## Key Enterprise Themes

**Cost Management:**
- Runaway loops that cost thousands
- Multi-agent chatter multiplying LLM calls
- Cost caps, iteration limits, budget breakers
- When agents are cheaper vs more expensive than alternatives

**Safety & Alignment:**
- Action boundaries (what the agent CANNOT do)
- Permission systems (least-privilege for tools)
- Approval gates for destructive actions
- Kill switches and emergency stops
- Audit trails for regulated industries

**Debugging & Observability:**
- Full trace of agent decisions
- Reasoning-vs-action divergence detection
- Tool call sequences
- Multi-agent conversation logs
- Root-causing agent failures in production

**Evaluation:**
- Trajectory analysis (not just output correctness)
- Tool selection accuracy
- Task completion rate
- Cost efficiency
- Human-in-the-loop calibration

**Real-World Deployment:**
- Browser agents (biggest security surface)
- Coding agents (production access implications)
- Customer support agents (liability)
- Research agents (data leakage)
- Workflow agents (automation risk)

**Framework Selection:**
- When LangGraph (state machines, HIL)
- When CrewAI (role-based multi-agent)
- When AutoGen (conversational multi-agent)
- When OpenAI Swarm (minimal orchestration)
- When to build custom (framework limitations)

## Series Navigation

| Week | Topic | Repo | Status |
|---|---|---|---|
| 1 | LLM Fundamentals | [Understanding_LLMs_From_The_Inside_Out](https://github.com/JoshithReddyAleti/Understanding_LLMs_From_The_Inside_Out) | ✅ |
| 2 | Python for AI | [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters) | ✅ |
| 3 | Tool Calling, APIs & Validation | [Building_AI_Project-Blueprint_for_Begin](https://github.com/JoshithReddyAleti/Building_AI_Project-Blueprint_for_Begin) | ✅ |
| 4 | End-to-End AI Projects | [Your_First_End_To_End_AI_Project](https://github.com/JoshithReddyAleti/Episode_4_Your_First_End_To_End_AI_Project) | ✅ |
| 5 | RAG & Augmented Generation | [Mastering_RAG_and_Augmented_Generation](https://github.com/JoshithReddyAleti/Episode_5_Mastering_RAG_and_Augmented_Generation) | ✅ |
| 6 | Frameworks & Fine-Tuning | [AI_Frameworks_and_Fine_Tuning_Complete_Guide](https://github.com/JoshithReddyAleti/Episode_6_AI_Frameworks_and_Fine_Tuning_Complete_Guide) | ✅ |
| 7 | Memory & State in AI Systems | [Memory_and_State_in_AI_Systems](https://github.com/JoshithReddyAleti/Episode_7_Memory_and_State_in_AI_Systems) | ✅ |
| 8 | Evaluation, Validation & Governance | [AI_Evaluation_Validation_and_Governance](https://github.com/JoshithReddyAleti/Episode_8_AI_Evaluation_Validation_and_Governance) | ✅ |
| **9** | **Agents: When AI Systems Make Decisions** ← you are here | [Agents_When_AI_Systems_Make_Decisions](https://github.com/JoshithReddyAleti/Episode_9_Agents_When_AI_Systems_Make_Decisions) | ✅ |
