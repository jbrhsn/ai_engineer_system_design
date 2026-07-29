# Agent Design Patterns — Interview Prep

**Section:** Agentic and Multi-Agent Systems | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is the ReAct pattern and what does the agent loop actually do on each turn? | ReAct interleaves reasoning ("Thought") with acting ("Action" = a tool call) and observing ("Observation" = tool result), then repeats. Each loop iteration: the model receives conversation state + tool results, decides to either call another tool or emit a final answer; tool outputs are appended to context as new observations. Modern implementations use **native tool/function calling** (structured `tool_calls` in the model response) rather than parsing free-text "Action:" strings. | Describing ReAct as just "the model thinks step by step" (that's chain-of-thought) and omitting the act→observe→re-reason cycle; or claiming the model "executes" tools itself — it only *requests* a call, your runtime executes it and feeds the result back. |
| How do native tool/function calling and tool schemas work, and why prefer them over regex-parsed ReAct? | You register tools with a JSON-schema description (name, params, types, descriptions); the model returns a structured `tool_calls` object your code dispatches. In LangChain this is `llm.bind_tools([...])`; LangGraph routes execution through a `ToolNode`. Schemas make args machine-parseable, reduce parsing failures, and let the model select args by type. | Saying "you just tell the model the tool exists in the prompt" — that's brittle text parsing; also forgetting that vague/overlapping tool descriptions are the #1 cause of wrong-tool selection. |
| Why must an agent loop be bounded, and how do you bound it? | An LLM can loop indefinitely (retrying a failing tool, oscillating between two tools, never emitting a final answer) → runaway cost/latency. Bound with a max-iteration/recursion cap, wall-clock timeout, and a termination condition (final-answer detection). Also detect no-progress loops (repeated identical tool calls). | Treating an unbounded `while` loop as acceptable "because the model will stop eventually," or relying only on token limits instead of an explicit iteration cap and timeout. |
| Contrast Plan-and-Execute with a single ReAct loop. When is planning worth the overhead? | ReAct decides one step at a time (reactive, flexible, more model calls interleaved with tools). Plan-and-Execute first has a **planner** emit a multi-step plan, an **executor** run steps, and a **replanner** revise when steps fail or new info arrives. Planning helps for long-horizon, multi-step tasks where up-front decomposition reduces wasted actions; ReAct is better for short, exploratory tasks. Trade-off: planning adds an extra model call and can commit to a stale plan. | Claiming Plan-and-Execute is "always better because it plans ahead" — it adds latency and can lock into a bad plan without a replanner; or conflating the planner and executor into one call. |
| What is the Reflection / self-critique pattern (and Reflexion), and what does it cost? | A generator produces an answer; a **critic** (often the same model with a critique prompt) evaluates it against criteria and returns feedback; the generator revises. Reflexion adds persisting that feedback across attempts as memory. Improves quality/accuracy on hard tasks but roughly multiplies latency and token cost per reflection round; needs a stop condition. Tree-of-Thoughts, by contrast, explores multiple branches and prunes — broader search, far higher cost. | Saying "just have the model check its own work and it'll be right" — self-critique without concrete criteria or a bounded number of rounds inflates cost with diminishing returns and can even reinforce errors. |
| How do you decide between raw tool-calling loop, LangChain agent, and LangGraph? | Raw loop: maximum control, minimal deps, you own state/looping — good for simple, well-scoped agents. LangChain prebuilt agents: fast to stand up, less boilerplate, less control over the graph. LangGraph: explicit `StateGraph`, durable persistence via a `checkpointer`, native human-in-the-loop (interrupts) and multi-agent topologies — pick it when you need control, persistence, HITL, or multi-agent. Selection criteria: control, persistence, HITL, multi-agent needs, lock-in tolerance. | "Just use LangGraph for everything" (or the inverse) — cargo-culting a framework without mapping requirements (persistence? HITL? multi-agent?) to capabilities; ignoring lock-in and operational complexity. |

---

## Applied / Scenario Questions

### Q1 — A production ReAct support agent occasionally spikes to 40s latency and burns tokens; some sessions never return.

**Strong answer framework:**
- **Diagnose the loop first:** unbounded or too-high iteration cap lets the model retry a flaky tool or oscillate between two tools. Add an explicit max-iteration cap and a wall-clock timeout, plus no-progress detection (identical repeated `tool_calls`).
- **Tool-error handling:** a tool that throws or returns an ambiguous error causes the model to retry blindly. Return structured, actionable error observations (e.g. "422: missing `order_id`") so the model can correct rather than loop; on repeated failure, break the loop and return a graceful fallback.
- **Trade-off framing:** a tighter iteration cap and timeout trade a small accuracy loss on genuinely hard queries for bounded latency/cost and safety (no runaway sessions). Quantify the budget: e.g. cap at 6 iterations / 15s, return partial answer + escalation path beyond that.
- **Tighten tool schemas:** vague descriptions cause wrong-tool selection and extra loops — sharpen names/descriptions and remove overlapping tools to reduce iterations at the source (accuracy *and* cost win).
- **Observability:** log per-iteration tool calls and latency so you can see where the loop stalls; add eval on a replay set before/after tuning the cap.

### Q2 — Answer quality on a multi-step research/report agent is inconsistent; stakeholders want higher accuracy without an unbounded cost blow-up.

**Strong answer framework:**
- **Match pattern to task shape:** long-horizon report generation favors Plan-and-Execute (planner decomposes → executor runs steps → replanner handles failed/steps-that-changed) over a flat ReAct loop that wastes actions rediscovering structure.
- **Add bounded Reflection for the quality lift:** a generator/critic round on the draft against explicit criteria (completeness, citation coverage) raises accuracy — but cap it at 1–2 rounds because cost/latency scale linearly per round with diminishing returns.
- **Trade-off framing:** planning + reflection improves accuracy at the cost of extra model calls (latency + $). Reserve reflection for the final synthesis step, not every intermediate step, to keep cost bounded. Contrast Tree-of-Thoughts (broad branch search) — higher accuracy ceiling but cost usually not justified for a report agent.
- **Safety:** ground claims in retrieved sources and have the critic check citation support to reduce hallucination, not just prose quality.
- **Framework fit:** on LangGraph, model planner/executor/critic as explicit nodes in a `StateGraph` with a `checkpointer` so long runs are durable and resumable, and steps are individually observable/evaluable.

---

## System Design / Architecture Questions

### Q — Design a tool-using agent that answers customer questions over internal APIs (orders, inventory, refunds), with human approval required before any refund is issued.

**Approach:**

1. **Clarify requirements.** Expected QPS and concurrency; latency budget (interactive → target < ~10–15s); which actions are read-only vs. state-changing; data sensitivity (PII in orders); hallucination tolerance (refunds must never be issued on a hallucinated order); do we need conversation persistence/resume; is human-in-the-loop mandatory (yes — refunds).
2. **High-level architecture.**
   - Core: a **ReAct-style tool-calling loop** — read-only tools (`get_order`, `check_inventory`) bound via structured schemas; the model requests calls, the runtime (a `ToolNode` in LangGraph terms) executes and returns observations.
   - **Bound the loop:** max-iteration cap + wall-clock timeout + no-progress detection + final-answer termination.
   - **HITL gate:** the `issue_refund` tool is a state-changing action routed through a human-approval interrupt — the graph pauses, surfaces the proposed refund to an agent, and only resumes on approval.
   - **Persistence:** a `checkpointer` (e.g. Postgres-backed) stores graph state so an interrupted/approval-pending session is durable and resumable — matches a FastAPI/PostgreSQL stack cleanly.
   - **Guardrails:** structured tool errors fed back as observations; input/output validation; scope refund tool to require a verified `order_id` returned by `get_order` (never a model-invented one).
3. **Justify choices and name trade-offs.**
   - **Framework:** LangGraph over a raw loop or a prebuilt LangChain agent because we need durable persistence *and* native human-in-the-loop interrupts *and* explicit control over the state graph — exactly its differentiators. Trade-off: more upfront modeling + framework lock-in vs. control/persistence/HITL we'd otherwise hand-roll.
   - **ReAct over Plan-and-Execute:** queries are short and reactive (look up, decide, maybe act); planning overhead isn't justified. Trade-off: less up-front decomposition, but far lower latency for the common case.
   - **Loop bound:** trades a rare accuracy loss on pathological queries for guaranteed bounded cost/latency and no runaway sessions (safety).
   - **HITL on refunds:** trades throughput/latency on the refund path for safety and auditability — the correct trade for an irreversible, money-moving action.
   - **Evaluation:** replay-set eval on tool-selection accuracy and end-to-end task success before rollout; log per-iteration traces for regression detection.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **ReAct (reason–act–observe loop)** — when describing the fundamental single-agent control loop and why observations re-enter context.
- **Native tool / function calling & tool schema** — when explaining structured `tool_calls` and JSON-schema tool registration (`bind_tools`) vs. brittle text parsing.
- **Loop bounding (max-iteration cap, wall-clock timeout, no-progress detection)** — when discussing cost/latency/safety control of any agent loop.
- **Structured tool-error observation** — when describing recoverable error handling that lets the model self-correct instead of retry-looping.
- **Plan-and-Execute (planner / executor / replanner)** — when the task is long-horizon and benefits from up-front decomposition plus revision on failure.
- **Reflection / self-critique, generator-critic, Reflexion** — when arguing for a bounded quality-improvement round and, for Reflexion, persisted feedback as memory.
- **Tree-of-Thoughts** — when contrasting broad branch search (higher ceiling, higher cost) against reflection's linear-cost revision.
- **StateGraph / ToolNode / checkpointer / human-in-the-loop interrupt** — when justifying LangGraph for control, durable persistence, and HITL.
- **Selection criteria: control / persistence / HITL / multi-agent / lock-in** — when framing framework choice as a requirements-to-capabilities mapping.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"Just add more agents / more reflection and it'll be smarter."** — Red flag: ignores that each added agent/round multiplies cost and latency and adds coordination failure modes; no cost/accuracy trade-off reasoning.
- **"ReAct is the only/right way to build agents."** — Red flag: shows no awareness that Plan-and-Execute or a plain bounded loop fits different task shapes; pattern dogma over requirements.
- **"The agent runs until it's done" / an unbounded loop is fine.** — Red flag: signals no grasp of runaway cost/latency and the need for explicit caps, timeouts, and termination conditions.
- **"The model executes the tools."** — Red flag: the model only *requests* tool calls; your runtime executes them and returns observations. Misunderstands the loop.
- **"Just tell the model about the tool in the prompt."** — Red flag: describes brittle text parsing instead of structured schemas / native function calling.
- **"Have it check its own work and it'll be correct."** — Red flag: self-critique without explicit criteria or a bounded number of rounds inflates cost and can reinforce errors.
- **"Use LangGraph for everything" (or "frameworks are overhead, always hand-roll").** — Red flag: framework cargo-culting; no mapping of persistence/HITL/multi-agent/lock-in needs to the choice.

---

## STAR Answer Frame

**Situation:** A production LangGraph multi-agent support system I owned had a ReAct-style triage agent whose sessions occasionally ran away — latency spiked and token spend on a small fraction of conversations dwarfed the median, with a handful of sessions never returning a final answer.

**Task:** Cut tail latency and per-session cost without hurting resolution quality, and make long-running approval-gated sessions durable — all on our FastAPI/PostgreSQL stack.

**Action:** Instrumented per-iteration tool-call traces and found the loop oscillating between two overlapping tools and retrying a flaky API blindly. I (1) added an explicit max-iteration cap, wall-clock timeout, and no-progress detection for repeated identical `tool_calls`; (2) sharpened and de-duplicated tool schemas so the model stopped picking the wrong tool; (3) returned structured, actionable tool-error observations so the model could self-correct instead of retry-loop; (4) routed the one state-changing action through a human-in-the-loop interrupt with a Postgres-backed `checkpointer` so approval-pending sessions were durable and resumable; and (5) built a replay-set eval to confirm no regression in task success before rollout.

**Result:** Tail (p99) latency dropped sharply and runaway sessions went to zero; per-session token cost on the affected slice fell by roughly half, with task-success on the replay set holding flat within noise — bounded cost and latency with no accuracy loss.

---

## Red Flags Interviewers Watch For

Specific to agent design patterns:

- **No loop cap or termination strategy** — proposing a `while` loop with no max-iteration cap, timeout, or no-progress detection; the single clearest sign of no production agent experience.
- **No tool-error handling** — assuming tools always succeed; not feeding structured errors back as observations, or letting a failing tool trigger blind infinite retries.
- **Free-text tool parsing over native function calling** — reaching for regex on "Action:" strings instead of structured `tool_calls` and JSON schemas.
- **Framework cargo-culting** — defaulting to LangGraph (or refusing all frameworks) without mapping control/persistence/HITL/multi-agent/lock-in requirements to the decision.
- **Pattern dogma** — insisting ReAct (or Plan-and-Execute, or Reflection) is universally best, with no task-shape reasoning about latency/cost trade-offs.
- **Unbounded reflection** — adding self-critique with no explicit criteria and no cap on rounds, ignoring linear cost growth and diminishing returns.
- **No human-in-the-loop on irreversible actions** — letting the agent autonomously perform money-moving or destructive operations with no approval gate.
- **No evaluation** — shipping an agent with no replay-set/task-success eval and no per-iteration observability, so regressions and loop pathologies go undetected.
