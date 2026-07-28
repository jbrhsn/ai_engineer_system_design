# Rapid-Fire Mock Question Bank & Self-Assessment — Interview Prep

**Section:** 05 · Interview Practice → 03 · Rapid-Fire Mock Question Bank & Self-Assessment | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

> **How to read this file.** This is the *capstone-readiness* companion for the three mock banks in this chapter — `01-mock-question-bank-rag-and-retrieval-systems.md`, `02-mock-question-bank-multi-agent-and-orchestration.md`, and `03-mock-question-bank-security-eval-and-scaling.md`. Where those banks drill one domain at a time, this file collects the **cross-cutting** questions that force you to connect retrieval, orchestration, and production concerns in a single answer — the way a real system-design loop does. Use it to self-assess whether you can defend an end-to-end production AI system, not just recite one topic.

---

## Core Conceptual Questions

These test whether you can connect the three domains cold. Each question deliberately spans RAG **and** agents **and** security/eval/scaling — the interviewer is checking that you don't silo your knowledge.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| How do you evaluate a RAG system end-to-end? | Split into retriever and generator; retriever = context precision + context recall (ranking-aware, completeness); generator = faithfulness (claims entailed by context) + answer relevancy (addresses intent). Faithfulness measures grounding, **not** real-world truth. Start reference-free so you can run on production traffic, then add a gold set for regression gating. | "I'd check if the answers look good" or naming only faithfulness — treating a single metric as sufficient, or conflating faithfulness with factual correctness. |
| How do you defend an agent that has tool access? | Layered defense-in-depth: indirect prompt injection (LLM01) triggers Excessive Agency (LLM06). Kill excessive **functionality** (read-only tools), excessive **permissions** (least-privilege scopes, per-user context), excessive **autonomy** (HITL before irreversible actions). Enforce authorization in the downstream system (complete mediation) — never let the LLM decide if an action is allowed. | "A strong system prompt stops injection" — the app layer can never fully patch it; and treating the LLM's own reasoning as the security boundary. |
| When do you choose multi-agent over a single agent with tools? | Default to single-agent-with-tools. Add agents only for the three named drivers: context management (too many tools/domain for one window), distributed development (separate teams own capabilities), parallelization (genuinely independent subtasks). Multi-agent costs ~15× chat tokens vs ~4× for single agent, plus coordination complexity. | "Multi-agent is more powerful / state of the art" — reaching for sophistication without a concrete pressure that justifies the token and complexity cost. |
| How do you control token cost at scale without regressing quality? | Treat token spend as a first-class SLO. Levers: cap max agent steps/tool calls; route cheap subtasks to a small model, reserve the frontier model for synthesis; semantic-cache retrieval and repeated sub-answers; trim context to only what context precision says you need; per-user/per-task quotas + rate limits (also the LLM10 Denial-of-Wallet mitigation). Gate every cut with an eval suite. | "Just use a smaller/cheaper model" — cutting cost blind, with no eval gate to catch the silent drop in agent-goal-accuracy. |
| Faithfulness vs. answer relevancy — what's the difference and why watch both? | Faithfulness = are the answer's claims supported by the *retrieved context* (grounding/hallucination). Answer relevancy = does the answer address the *user's question* (intent), and it does **not** check factual accuracy. A bot can be perfectly faithful yet evasive/unhelpful, or highly relevant yet fabricated — the classic tension means you optimize neither alone. | "Faithfulness = correctness" — assuming a faithful answer is a true or helpful answer; ignoring that faithfulness is silent on both retrieval quality and intent. |
| Direct vs. indirect prompt injection — which is worse in a RAG/agent system, and why? | Direct = the user's own prompt manipulates behaviour ("ignore previous instructions"). Indirect = the model ingests external content (retrieved doc, webpage, email) carrying hidden instructions. Indirect is more dangerous here because the payload arrives through a *trusted-looking data channel the user never sees* — exactly the RAG/tool ingestion path. | "Injection is solved by input filtering" — a single filter is insufficient by design (obfuscation, payload splitting, encoding); missing that the retrieval channel itself is the attack surface. |
| Why doesn't a reranker fix low answer quality if retrieval recall is poor? | A reranker only *reorders what retrieval returned*. At recall@k = 0.71 the gold doc is absent ~29% of the time, so no reranking can surface it. Recall is the ceiling; fix stage 1 first (hybrid BM25+dense fusion, chunking, raise k to ~0.95 recall), then spend precision effort. Ties retrieval quality to the downstream generation/eval story. | "Add a bigger reranker" — spending latency reordering a candidate set that structurally lacks the answer. |
| When is a deterministic workflow the right call instead of an autonomous agent? | Workflows orchestrate LLMs/tools through predefined code paths — predictable, cheaper, best when the task decomposes into fixed subtasks. Agents let the LLM direct its own process — needed for open-ended problems with unpredictable step count/order. Agents trade latency, cost, and compounding-error risk for flexibility. Use the simplest thing that works. | "Agents are the modern way, workflows are legacy" — equating autonomy with quality and ignoring the cost/error-compounding trade-off. |

---

## Applied / Scenario Questions

These combine domains the way a production incident does. Each strong-answer framework includes an explicit **latency vs. accuracy vs. cost vs. safety** trade-off bullet.

**Q1:** Your production RAG agent is hallucinating on ~15% of answers *and* it's running over its monthly token budget. Diagnose and fix — you can't just throw a bigger model at it.

**Strong answer framework:**
- **Instrument before you touch anything.** Pull traces: per-request token spend, step count, faithfulness and context precision/recall on the failing samples. Separate the two problems — hallucination is an eval/retrieval problem, budget overrun is a scaling/cost problem — but note they often share a root cause (an unbounded agent loop that both burns tokens and drags in irrelevant context).
- **Diagnose the hallucination.** If context recall is low, the fix is *retrieval* (hybrid search, better chunking, raise k), not the generator — feed it grounded material so it can be faithful *and* relevant. If recall is fine but faithfulness dropped, inspect the generation change: new model version, higher temperature, or a prompt now encouraging "helpful detail" beyond context. Add a grounded prompt at temperature 0, citation enforcement, and a cheap faithfulness gate that routes low-confidence answers to abstention.
- **Diagnose the budget.** Cap max steps/tool calls per task (bounds worst-case fan-out); route cheap subtasks (classification, extraction) to a small model and reserve the frontier model for synthesis; add a semantic cache for repeated retrievals; trim context to only what context precision says is needed.
- **Trade-off (latency/accuracy/cost/safety):** Every cost lever (fewer steps, smaller models, aggressive context trimming) risks lowering agent-goal-accuracy, and every safety lever (abstention gate, temperature 0) trades *coverage* for *groundedness*. Resolve it by gating all cuts behind the eval suite so you reduce cost and hallucination together without silently regressing task success — the safe default when unsure is "I don't know," not a confident guess.

---

**Q2:** Design an evaluation + guardrail strategy for a new customer-facing support agent that can look up account data and draft (but not send) emails.

**Strong answer framework:**
- **Eval, split by pipeline stage.** Retrieval: context precision + context recall. Generation: faithfulness + answer relevancy, plus topic adherence to catch off-domain answers. Run reference-free on production traffic for the continuous signal; maintain a labelled gold set as the pre-merge regression gate.
- **Offline + online, not either/or.** Offline gold-set eval in CI blocks releases and catches regressions; online monitoring (reference-free faithfulness/relevancy, latency, cost, refusal rate) catches drift and adversarial inputs the offline set never saw. Feed new online failure cases back into the offline set.
- **Guardrails as defense-in-depth.** Input side: filter PII and injection patterns, scope off-topic queries. Retrieval channel: treat retrieved content as untrusted (indirect-injection surface), segregate and mark it. Tool side: account lookup runs read-only under the user's security context (least privilege, complete mediation in the DB); email drafting is autonomous but **sending requires human approval** (HITL before the irreversible action). Output side: validate format, scan for leaked secrets, run a faithfulness gate before display.
- **Trade-off (latency/accuracy/cost/safety):** Each guardrail layer adds latency and engineering surface, and HITL adds human cost — so gate HITL only on high-risk/irreversible actions (send email) while read-only lookups stay autonomous. You accept a slightly less "magical" hands-off assistant in exchange for a bounded blast radius, and you accept some false abstentions to keep fabrication near zero.

---

## System Design / Architecture Questions

**Q:** Design a production-grade multi-agent customer-support system with retrieval, evaluation, guardrails, and cost controls. Assume it answers policy/product questions, looks up account state, and can perform a small set of account actions (e.g. issue a refund).

**Approach:**

1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity, cost ceiling).** What's the p95 latency SLO and traffic concurrency? What is the hallucination tolerance — is this advisory or does it quote binding policy (which demands near-zero fabrication and abstention)? What data sensitivity / tenancy isolation is required? Is there a hard monthly token budget? These answers decide topology, guardrail strictness, and where HITL sits.

2. **Propose high-level architecture (agent topology, retrieval layer, guardrails, eval loop).**
   - *Topology:* start with a **supervisor** that routes intents — a Q&A path (RAG over the policy/product KB), an account-lookup path (read-only DB tool), and an action path (refund tool). Only split into subagents where a driver justifies it (e.g. a separate team owns billing). Bound every agent loop with a max-step/recursion limit and a token budget.
   - *Retrieval layer:* hybrid search (BM25 + dense) fused with RRF for recall, a cross-encoder reranker over the top-k, small-to-big chunking so the KB returns sharp, entry-aligned context. A retrieval score gate abstains when top-chunk similarity is below threshold.
   - *Guardrails:* input filtering (PII, injection); retrieved content treated as untrusted (indirect-injection surface); least-privilege tool scopes (read-only account lookup under the user's security context, complete mediation in the DB); HITL approval immediately before the refund (irreversible) action; output validation + faithfulness/citation gate before display.
   - *Eval + observability:* Ragas metrics (context precision/recall, faithfulness, answer relevancy) as an offline gold-set gate in CI, plus online reference-free monitoring with distributed tracing across every agent hop, tool call, latency, and per-request token spend. Route new failures back into the gold set.
   - *Cost controls:* semantic cache on retrieval and repeated sub-answers; model routing (small model for intent classification/extraction, frontier for synthesis); per-user/per-task quotas + rate limits (Denial-of-Wallet / LLM10 defense); input-size validation and timeouts.

3. **Justify choices and name trade-offs explicitly (cost, latency, complexity, security).** Supervisor over a swarm because centralized routing is easier to trace and cheaper than ~15× multi-agent token cost — you only pay for isolation where a real driver exists. Hybrid + rerank buys precision at ~50–120 ms latency, capped by never reranking beyond a bounded candidate set. The abstention/HITL layers trade coverage and latency for safety on binding-policy answers and irreversible actions — a deliberate choice given data sensitivity. The eval gate + tracing add engineering surface but are the only way to cut cost (routing, step caps) without silently regressing goal-accuracy. Net: the design optimizes for *defensible* answers within budget, not maximal autonomy.

---

## Vocabulary That Signals Expertise

Use these naturally to show you think across all three domains — don't force them:

- **Faithfulness** — invoke when discussing hallucination; signals you know grounding-in-context is measurable and distinct from truth.
- **Context precision / context recall** — use to separate *ranking* (precision, position-aware) from *completeness* (recall); shows you diagnose retrieval before blaming the generator.
- **LLM-as-judge** — name when explaining how faithfulness/relevancy scale past human review; pair with "calibrated against a human-labelled gold set" to show you know its biases (verbosity, position, self-preference).
- **Indirect prompt injection** — use when discussing RAG/agent threats; signals you understand the *channel* distinction and that retrieval is an attack surface.
- **Excessive agency** — cite the three root causes (functionality, permissions, autonomy); signals OWASP LLM06 fluency and that you scope tools, not just prompts.
- **Defense in depth** — use to explain why no single filter stops injection; signals you layer independent controls that each lower success probability.
- **Semantic cache** — name as a cost/latency lever for repeated retrievals or sub-answers.
- **Continuous batching** — invoke when discussing serving throughput under concurrency; signals you know inference-layer scaling, not just app-layer.
- **Denial-of-wallet** — use for the cost-attack angle of LLM10 Unbounded Consumption; signals you treat token spend as a security concern, not just a bill.
- **Guardrail** — use precisely as a *runtime input/output control*; signal you know it reduces blast radius but doesn't eliminate injection.
- **Observability / tracing** — name distributed tracing across agent hops and tool calls; signals you can debug and cost-attribute a live multi-agent system.

---

## Vocabulary That Signals Weakness

Avoid these — each reveals a shallow or outdated mental model:

- **"RAG eliminates hallucination"** — red flag: RAG *reduces* but never *eliminates* it; the model can still ignore or misread retrieved text, which is why grounding, verification, and abstention exist.
- **"We'll add eval later"** — red flag: without eval you can't gate releases, catch regressions, or safely cut cost; eval is the instrument you steer by, not a nice-to-have.
- **"Prompt injection is solved by a system prompt"** — red flag: OWASP LLM01 is explicit there is no fool-proof app-layer prevention; the system prompt is one layer of defense-in-depth, not the boundary.
- **"Just use a bigger model"** — red flag: ignores cost/latency, doesn't fix retrieval recall or injection, and shows no diagnostic discipline — the answer to almost nothing is "bigger model."
- **"Multi-agent is state of the art"** — red flag: reaching for the ~15×-token architecture with no concrete driver (context, distributed dev, parallelization) signals sophistication-chasing over engineering judgment.
- **"Faithfulness is high so the answer is correct"** — red flag: conflates grounding-in-context with real-world truth; a faithful answer over a wrong/stale source is still wrong.
- **"A reranker will fix our bad answers"** — red flag: ignores that a reranker can't surface a doc retrieval never returned; recall is the ceiling.
- **"We don't need rate limits, we're not that big"** — red flag: no cost awareness; Denial-of-Wallet doesn't care about your size, it targets your pricing model.

---

## STAR Answer Frame

**Situation:** A production customer-support RAG agent was quietly hallucinating on binding policy answers and its monthly inference bill had grown ~40% quarter-over-quarter with no traffic increase — leadership wanted both fixed without degrading the customer experience.

**Task:** I owned hardening the agent's groundedness and bringing spend back under the budget ceiling, while proving with data that neither fix regressed answer quality.

**Action:** I first added distributed tracing (per-hop tokens, step counts) and a Ragas eval harness (context precision/recall + faithfulness + answer relevancy) run offline on a gold set and reference-free on live traffic. Tracing showed an unbounded agent loop was fanning out redundant retrievals — the root of *both* problems. I capped max steps, routed intent classification to a small model while reserving the frontier model for synthesis, and added a semantic cache. For groundedness, context recall was fine but faithfulness had dropped after a prompt change that encouraged "extra detail," so I pinned temperature 0, added citation enforcement, and a faithfulness gate routing low-confidence answers to abstention. Every change shipped behind the offline eval gate.

**Result:** Faithfulness recovered from 0.71 to 0.94, hallucinated policy answers dropped from ~15% to under 2%, and monthly token spend fell ~35% — with answer-relevancy holding flat, proving the cost cuts didn't quietly degrade quality.

---

## Red Flags Interviewers Watch For

Specific to production AI systems — an interviewer downgrades a candidate who:

- **Has no eval strategy** — proposes an architecture but can't say how they'd *measure* whether it works, or names "vibes"/manual spot-checks instead of retriever/generator metrics and a gold-set regression gate.
- **Ignores security / injection** — designs an agent with tool access and never mentions indirect prompt injection, least privilege, complete mediation, or HITL on irreversible actions — treating the LLM's reasoning as the security boundary.
- **Shows no cost awareness** — talks accuracy and latency but never token economics, step bounds, model routing, caching, or Denial-of-Wallet — as if inference were free.
- **Has no observability plan** — can't say how they'd debug a failing multi-agent run in production (no tracing across hops, no per-request cost attribution, no way to feed failures back into eval).
- **Optimizes one metric in isolation** — pushes faithfulness to 1.0 while ignoring relevancy/recall, or chases recall without precision — missing that these live in tension.
- **Reaches for complexity without a driver** — jumps to multi-agent or an agentic CRAG loop with no measured failure justifying the added tokens, latency, and failure surface.

---

### How to Run a Timed Mock Loop with This Chapter's Banks

Use the three banks in this chapter as a graded, timed self-assessment before a real loop:

1. **Warm-up (recall, ~30s each).** Run the Rapid-Fire round of `01-mock-question-bank-rag-and-retrieval-systems.md`, then `02-mock-question-bank-multi-agent-and-orchestration.md`, then `03-mock-question-bank-security-eval-and-scaling.md` — answer *aloud before* revealing each `<details>` block. No peeking.
2. **Depth (applied + analysis, ~90s each).** Do the Applied and Analysis/Trade-off rounds from all three banks. For every answer, force yourself to name the explicit trade-off (latency vs. accuracy vs. cost vs. safety) — that is what the scenario questions in *this* file drill.
3. **Cross-cut (this file).** Then answer the Core Conceptual, Scenario, and System Design questions above cold — they deliberately span all three banks at once.
4. **Score and route.** Mark each row on the scorecard at the bottom of each bank: **✓** explain cold · **△** shaky · **✗** can't yet. Re-drill only the △/✗ rows against the review targets each bank links.
5. **Repeat under time pressure.** On the second pass, set a timer and speak answers as if an interviewer is probing follow-ups — the goal is a crisp opening answer plus one layer of depth, in the target time per question type.
