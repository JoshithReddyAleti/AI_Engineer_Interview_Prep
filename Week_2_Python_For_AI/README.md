# 🐍 Week 2 — Python for AI Interview Prep

> **Maps to:** [Python_For_AI_What_Actually_Matters](https://github.com/JoshithReddyAleti/Python_For_AI_What_Actually_Matters)
>
> **Newsletter:** [AI Engineering Roadmap 2026](https://www.linkedin.com/newsletters/ai-engineering-roadmap-2026-7467249724752908288/)

This isn't general Python trivia. These are the specific Python patterns that AI engineering interviews drill on — HTTP clients, async, data serialization, environment management, and the error handling philosophy that separates production code from notebook code.

## Files in This Folder

| File | Questions | Focus |
|---|---|---|
| [01_Deep_Conceptual_Questions.md](01_Deep_Conceptual_Questions.md) | 12 | Async vs sync, GIL, data modeling, environment management, error handling philosophy |
| [02_Technical_Coding_Questions.md](02_Technical_Coding_Questions.md) | 10 | API wrappers, retry logic, data pipelines, config management, concurrent calls |
| [03_System_Design_Questions.md](03_System_Design_Questions.md) | 6 | Python service architecture, package design, deployment, testing non-deterministic systems |
| [04_Behavioral_Scenario_Questions.md](04_Behavioral_Scenario_Questions.md) | 8 | Debugging production Python, dependency management, code review conflicts, tech debt |

## Key Topics Tested

- HTTP requests and API client design patterns
- JSON/dict manipulation — the data format of LLM I/O
- Environment variables and secrets management (never hardcode keys)
- Error handling, retry logic, and graceful degradation
- Async/await for concurrent LLM API calls
- Dataclasses vs Pydantic for data modeling
- File I/O patterns for AI pipelines (streaming, batching, logging)
- Testing strategies for systems with non-deterministic outputs
- Python packaging and dependency management for AI projects
