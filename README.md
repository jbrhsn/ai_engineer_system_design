# AI System Design for AI Engineers

A Markdown-only learning repository for **AI/ML system design interview preparation at the Applied AI Engineer level** — designing and defending production-grade agentic AI, RAG, and multi-agent architectures, with a security-first and evaluation-first lens.

This repo is built for an engineer with 4 years of hands-on production experience (LangGraph multi-agent systems, RAG pipelines, FastAPI/PostgreSQL) who needs to formalize that experience into interview-ready system design communication. It is not a from-scratch AI course — it assumes working knowledge of agentic AI, RAG, and LLM-based systems, and focuses on the *system design framing, trade-off vocabulary, and production concerns* that interviewers probe for.

**Time budget:** ~20–30 hours across 14 chapters.

---

## Learning Path

| Phase | Section | Est. hours | Focus area |
|---|---|---|---|
| 1 | [01 – Classical Systems Refresher](01-classical-systems-refresher/) | 3–6 | Distributed systems fundamentals + data/API layer, as a light refresher (not the focus) |
| 2 | [02 – LLM Serving & RAG Architecture](02-llm-serving-and-rag-architecture/) | 4.5–9 | Inference economics, embeddings/vector search, RAG pipeline design |
| 3 | [03 – Agentic & Multi-Agent Systems](03-agentic-and-multi-agent-systems/) | 4.5–9 | Agent design patterns, LangGraph orchestration, context engineering |
| 4 | [04 – Production AI Systems: Security, Eval, Scale](04-production-ai-systems-security-eval-scale/) | 4.5–9 | OWASP LLM Top 10, evaluation methodology, scaling/cost |
| 5 | [05 – Interview Practice: Case Studies & Drills](05-interview-practice-case-studies-and-drills/) | 4.5–9 | Delivery framework, worked case studies, mock question banks |
| — | [Capstone](capstone/) | — | Integrative production-platform design project |

---

## Repository Structure

```
ai_engineer_system_design/
├── AGENTS.md                                          ← authoring rules for this repo
├── templates/                                         ← reference templates (never edit directly)
├── 00-roadmap/                                        ← personal study roadmap (stub)
├── 01-classical-systems-refresher/
├── 02-llm-serving-and-rag-architecture/
├── 03-agentic-and-multi-agent-systems/
├── 04-production-ai-systems-security-eval-scale/
├── 05-interview-practice-case-studies-and-drills/
├── capstone/
└── progress-tracker.md
```

Each chapter folder contains topic notes (`01-*.md` – `03-*.md`, or `01`–`04` for the one dense chapter) plus an `interview-prep.md` file numbered immediately after the last topic note.

---

## Section Summaries

- **[01 – Classical Systems Refresher](01-classical-systems-refresher/)** — Scalability, load balancing, caching, CAP theorem, and the API/database layer, framed as the minimum classical systems vocabulary needed before discussing AI-native architecture.
- **[02 – LLM Serving & RAG Architecture](02-llm-serving-and-rag-architecture/)** — Inference/serving economics, embeddings and vector search, and RAG pipeline design patterns including zero-hallucination retrieval.
- **[03 – Agentic & Multi-Agent Systems](03-agentic-and-multi-agent-systems/)** — ReAct/plan-and-execute agent patterns, LangGraph multi-agent orchestration (supervisor, hierarchical), and context engineering/agent memory.
- **[04 – Production AI Systems: Security, Eval, Scale](04-production-ai-systems-security-eval-scale/)** — OWASP GenAI security risks and agentic threat models, RAG/agent evaluation methodology, and scaling/deployment/cost concerns.
- **[05 – Interview Practice: Case Studies & Drills](05-interview-practice-case-studies-and-drills/)** — The AI system design interview delivery framework, worked case studies drawn from production agentic system patterns, and rapid-fire mock question banks.

---

## File Type Guide

| File type | Pattern | Purpose | Created at scaffold? |
|---|---|---|---|
| Topic notes | `01-kebab-case.md` – `03-*.md` (or `04-*.md` for the dense chapter) | Standard chapter template: TL;DR, ELI5, Key Concepts, Implementation, Self-Check | Yes (blank stub) |
| Interview prep | `04-interview-prep.md` (or `05-*.md` for the dense chapter) | Core questions, scenario questions, STAR frames, red flags | Yes (blank stub) |
| Section index | `README.md` (inside each numbered section) | Overview, chapter table, study tips | Yes (blank stub) |
| Roadmap | `00-roadmap/learning-roadmap.md` | Personal pacing plan | Yes (blank stub) |
| Progress tracker | `progress-tracker.md` | Chapter completion tracking | Yes (blank stub) |
| Capstone brief | `capstone/project-brief.md` | Integrative design project spanning ≥2 sections | Yes (blank stub) |
| Templates | `templates/*.md` | Reference structure + authoring rubric | Yes (fully populated) |

---

## Certification / Exam Target

No single certification exists for this topic. This repo prepares for **AI/ML system design interviews** (Applied AI Engineer track), grounded in the OWASP GenAI Security Project (LLM Top 10:2025), Anthropic's Applied AI engineering guidance, LangChain/LangGraph documentation, Ragas evaluation documentation, and Hello Interview's ML system design delivery framework, adapted for agentic/RAG systems. See `AGENTS.md` for full source-of-truth details and authoring rules.
