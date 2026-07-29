# 04 — Production AI Systems: Security, Evaluation & Scale

**Estimated time:** 27.5 hrs | **Prerequisites:** Section 03 (Agentic & Multi-Agent Systems) and Section 02 (LLM Serving & RAG Architecture) — this section takes the agents and retrieval pipelines those sections built and hardens them for production.

## Overview

This is the section where a working agentic or RAG system becomes a *shippable, defensible, affordable* one. It moves through the three concerns that separate a demo from production, in order: **security & governance** (bounding what a manipulated agent can do, then wrapping it in an accountability layer), **evaluation, observability & guardrails** (proving the system works, seeing inside every run, and enforcing runtime safety), and **scaling, cost & deployment** (surviving load, shipping changes safely, and controlling the token bill). The lens throughout is security-first and evaluation-first: assume prompt injection succeeds and engineer the blast radius, never report a single quality number, and treat cost as a first-class metric — the exact production maturity signals interviewers use to tell shippers from prototypers.

## Learning Outcomes

By completing this section you will be able to:
- Name the OWASP Top 10 for LLM Applications 2025 risks by ID, threat-model an agentic system by sizing the blast radius of every tool action, and defend architectural least-privilege (tool-layer RBAC) as the primary control for the unpreventable — because "the system prompt says not to" is not a security control.
- Wrap technical controls in a governance layer — NIST AI RMF, EU AI Act risk tiers, ISO/IEC 42001, plus concrete artifacts (model cards, use registry, versioned model/prompt pins, approval gates, audit logs) — that makes an AI system accountable and auditable.
- Evaluate a RAG/agent system by decomposing quality into retrieval vs generation (and, for agents, trajectory vs outcome) metrics, grounding them in a golden dataset, and calibrating an LLM-as-judge against human labels before gating any deploy on it.
- Instrument production runs with per-step tracing (spans, OpenTelemetry GenAI conventions, LangSmith) and enforce input/output guardrails with a deliberate fail-open vs fail-closed decision.
- Scale an agentic system by queuing, caching, backing off, and falling back against provider rate limits; ship prompt/model/tool changes as eval-gated, versioned, reversible progressive rollouts; and model and shrink per-request cost across tokens × price × calls.

## Chapters

| # | Chapter | Est. time | Files |
|---|---|---|---|
| 1 | Security & Governance for Agentic AI — the OWASP LLM Top 10 (2025) and agentic threat modeling (Excessive Agency, blast radius), prompt-injection defense-in-depth with tool-layer RBAC/least-privilege, and the governance wrapper (NIST AI RMF, EU AI Act, ISO/IEC 42001, model cards, approval gates, audit logs). | 9 hrs | notes + interview-prep |
| 2 | Evaluation, Observability & Guardrails — retrieval-vs-generation and agent trajectory/outcome metrics on golden datasets, LLM-as-judge scoring modes and bias calibration wired into offline/online eval pipelines, and production observability (span-level tracing, OpenTelemetry GenAI, LangSmith) with input/output guardrails and fail-open/closed trade-offs. | 9.5 hrs | notes + interview-prep |
| 3 | Scaling, Cost & Deployment for AI Systems — scaling under load against provider rate limits (async/concurrency, backoff, queue+worker, semantic/prompt caching), eval-gated versioned reversible CI/CD (shadow → canary → full with auto-rollback), and token economics (cost = tokens × price × calls) with per-tenant budgets and Denial-of-Wallet defense. | 9 hrs | notes + interview-prep |

## How This Section Fits

This section hardens what Sections 02 and 03 built. The RAG pipeline from Section 02 is now something you must *evaluate* (retrieval vs generation), *guardrail* (groundedness, PII), and *cost-model* (tokens × calls). The agents, tools, and durable execution from Section 03 reappear here as attack surface to bound (Excessive Agency, tool RBAC), as trajectories to trace and score (tool-call and goal accuracy), and as stateless orchestration behind an external checkpointer that scales horizontally. Forward, it directly stocks **Section 05 — Interview Practice**: the security containment argument, the retrieval-vs-generation eval split, the LLM-as-judge calibration caveat, the multi-agent token multiplier as a cost decision, and the eval-gated deploy story are the cross-cutting trade-off material that turns a plausible architecture into a defensible one in a case study.

## Study Tips

- **Memorize the quotable vocabulary — it is the fastest credibility signal.** Be able to rattle off the OWASP LLM Top 10 IDs (especially LLM01 Prompt Injection, LLM06 Excessive Agency, LLM10 Unbounded Consumption) and the Ragas-style metric names (context precision/recall, faithfulness, answer relevancy, tool-call accuracy, agent goal accuracy). Interviewers use these as shibboleths for whether you have actually operated these systems.
- **The three highest-yield topics are security containment, the eval split, and token economics.** For security, always land on architectural least-privilege / tool-layer RBAC over "filter the input" — you contain blast radius because injection isn't reliably preventable. For eval, never report one number: decompose retrieval vs generation so a failing score tells you *which stage* to fix, and remember an LLM judge is an approximation you must calibrate against humans. For cost, be able to write `cost = (input + output tokens) × price × calls` on a whiteboard and name the levers (route, trim, cache, bound loops).
- **Connect every control back to the system it hardens.** The blast radius you bound is the tool set from Section 03; the groundedness you guardrail is the retrieval from Section 02; the stateless orchestration you scale is the checkpointer-backed durable execution from Section 03. Answers that trace the control to the component read as production experience, not memorization.
- **Verify fast-moving provider and tooling details against current docs.** Rate-limit dimensions (RPM/TPM/ITPM), prompt-caching semantics, OpenTelemetry GenAI `gen_ai.*` conventions, LangSmith APIs, and Ragas metric definitions all evolve — confirm any specific field or number against the current official documentation before stating it as fact in an answer.
- Do the interview-prep file in each chapter last, as a self-test; if you can defend the security containment strategy, the retrieval/generation eval decomposition, and the per-request cost model without re-reading the notes, you are ready for Section 05.
