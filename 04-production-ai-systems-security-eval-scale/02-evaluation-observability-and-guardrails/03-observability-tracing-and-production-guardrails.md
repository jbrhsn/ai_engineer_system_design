# Observability, Tracing, and Production Guardrails

**Section:** 04 Production AI Systems → Evaluation, Observability & Guardrails | **Est. time:** 3 hrs | **Interview relevance:** High — "how do you debug and safely operate an agent in production?" is the question that separates a demo-builder from an engineer who can run the thing; you must be able to describe the trace tree of a multi-step run and where guardrails sit without hand-waving.

---

## TL;DR

An LLM/agentic system is non-deterministic, multi-step, and calls external tools, so a single "request succeeded, HTTP 200" metric tells you almost nothing about whether it actually did the right thing. **Observability** for these systems means **tracing** every step — each LLM call, tool call, and retrieval as a **span** nested inside the full trace tree of a run — and logging prompts, completions, token usage, cost, latency-per-step, and the model version, so a bad answer can be replayed and the failing step located. The emerging standard is the **OpenTelemetry GenAI semantic conventions** (`gen_ai.*` spans/metrics), with **LangSmith** as the reference commercial platform. **Guardrails** are the runtime controls that run *around* the model — **input guardrails** (PII redaction, prompt-injection/jailbreak detection, topic filtering) before the model, and **output guardrails** (toxicity, PII-leakage, groundedness, schema validation) after — and every guardrail forces a trade-off between latency/false-positives and safety, plus a **fail-open vs fail-closed** decision for when the guardrail itself errors. **The one thing to remember: logging the final answer is not observability and a good system prompt is not a guardrail — you need per-step traces to debug and enforced runtime checks to be safe, because the model will do surprising things no prompt fully prevents.**

---

## ELI5 — Explain It Like I'm 5

Imagine a self-driving delivery car that makes several stops on one trip: it picks a route, checks a map, asks a warehouse for a package, then drives to your door. **Observability** is the car's flight recorder plus a live dashboard: it writes down every single leg — which turn it took, how long each leg lasted, how much fuel it burned, which map version it used — so that if the wrong package arrives, you can replay the whole journey and see *exactly which stop* went wrong, instead of only knowing "the delivery was bad." **Guardrails** are the lane barriers and the speed limiter: some are checked *before* the car pulls out of the depot (is this even a safe destination? is there a hidden note in the order telling the car to rob a bank?), and some are checked *while it drives and before it hands you the box* (is the package leaking? is it the right one?). The mistake people make is thinking that writing "deliver safely" on the dashboard (a good prompt) means the car will never leave its lane — it will, so you install real barriers that physically stop it, and you decide in advance whether a *broken* barrier means "stop the car" (fail-closed) or "let it through" (fail-open).

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why traditional service metrics (uptime, HTTP status, p99 latency alone) are insufficient for LLM/agentic systems and what tracing adds.
- [ ] Diagram the trace tree of a multi-step or multi-agent run as nested spans and identify what to log at each span (prompt, completion, tokens, cost, latency, tool I/O, model version).
- [ ] Distinguish input guardrails from output guardrails, place them correctly around the model and tools, and give a concrete detector for each (PII, injection, toxicity, groundedness, schema).
- [ ] Decide fail-open vs fail-closed per guardrail and size a guardrail latency budget, justifying the safety-vs-latency/false-positive trade-off.
- [ ] Instrument and guard a customer-facing agent end-to-end, choosing trace sampling, signals to alert on, and where each guardrail runs.

---

## Visual Overview

### Trace Tree of a Multi-Step Agent Run (nested spans)

```
TRACE: "resolve customer refund request"        (root span — whole run)
│  metadata: user_id, session_id, model=gpt-4o-2024-11-20, total_tokens, total_cost, latency=3.9s
│
├─ SPAN: input_guardrails                        18 ms
│   ├─ pii_redact            ──► PASS (1 email masked)
│   └─ prompt_injection_scan ──► PASS
│
├─ SPAN: llm_call #1 (plan)                       820 ms   in=1,240 tok  out=90 tok  $0.004
│   └─ emits tool_calls=[lookup_order]
│
├─ SPAN: tool_call lookup_order                   210 ms   args={order_id:551}  ok
│
├─ SPAN: retrieval refund_policy                  95 ms    k=4 chunks  scores=[.81,.79,..]
│
├─ SPAN: llm_call #2 (answer)                     1.5 s    in=2,010 tok out=180 tok $0.011
│
└─ SPAN: output_guardrails                        140 ms
    ├─ toxicity        ──► PASS
    ├─ pii_leak_scan   ──► PASS
    ├─ groundedness    ──► FAIL (claim not supported by retrieved policy)  ◄── root cause
    └─ schema_validate ──► (skipped, blocked upstream)
```

### Input → Model → Output Guardrail Pipeline

```
user input
   │
   ▼
┌─────────────────────┐   fail ──► block / redact / safe-refusal  (never reaches model)
│  INPUT GUARDRAILS   │
│  pii · injection ·  │
│  topic filter       │
└─────────────────────┘
   │ pass (possibly redacted)
   ▼
┌─────────────────────┐        ┌───────────────────────────┐
│      MODEL /        │◄──────►│  TOOL / RETRIEVAL          │  (execution rails wrap tool I/O)
│    AGENT LOOP       │        └───────────────────────────┘
└─────────────────────┘
   │ raw completion
   ▼
┌─────────────────────┐   fail ──► block / regenerate / mask / fallback
│  OUTPUT GUARDRAILS  │
│  toxicity · pii-leak│
│  groundedness ·     │
│  schema validate    │
└─────────────────────┘
   │ pass
   ▼
response to user
```

### Fail-Open vs Fail-Closed (when the guardrail itself errors)

```
Guardrail call raised / timed out. What ships?
├── FAIL-OPEN  ──► let the unchecked output through   (availability > safety)
│                  use for: low-risk, latency-critical, advisory checks
└── FAIL-CLOSED ─► block, return safe fallback         (safety > availability)
                   use for: PII leak, toxicity, regulated / high-risk actions
```

### The Observe → Detect → Alert Loop

```
production traffic ──► TRACES (spans, tokens, cost, latency, guardrail results)
        │                        │
        │                        ▼
        │              sampled online evals (ch. 02)  ──► quality score
        ▼                        │
   dashboards / metrics ◄────────┘
        │
   threshold breached? (error-rate ↑, cost/req ↑, p95 ↑, groundedness ↓)
        └── yes ──► ALERT ──► human investigates the failing trace
```

---

## Key Concepts

### Why Traditional Metrics Aren't Enough for LLM Apps

**What it is.** The observability gap: classic service monitoring (uptime, HTTP status codes, request/error/duration "RED" metrics) reports whether the *service* responded, not whether the *answer* was correct, grounded, safe, or cost-effective.

**How it works mechanistically.** A traditional web endpoint is deterministic: same input → same output, and a 200 means success. An LLM call is stochastic, so an HTTP 200 with a confident, fluent, *wrong* or *hallucinated* answer is indistinguishable from a good one at the transport layer. Worse, an agent is *multi-step* — one user request fans out into several LLM calls, tool calls, and retrievals — so an end-to-end latency or error number cannot tell you *which* step was slow, expensive, or failed. The failure modes are new (hallucination, prompt injection, silent tool errors that the model "recovers" from with a fabricated answer, runaway token cost) and none of them trip a conventional alert.

**Where it appears in real systems.** This is why LLM observability platforms center on the **trace** rather than the counter: LangSmith models a trace as "a collection of runs for a single operation" and explicitly analogizes a trace to an OpenTelemetry collection of spans, and a run to a single span. You still keep RED metrics, but you add quality, cost, and per-step visibility on top.

### Tracing and Spans

**What it is.** A **trace** is the complete record of one operation (one user request / one agent run); a **span** is one unit of work inside it (an LLM call, a tool call, a retrieval step). Spans nest to form the trace tree.

**How it works mechanistically.** Each span records a start/end timestamp (→ latency), a type, inputs and outputs, and attributes (tokens, cost, model, error). A child span is linked to its parent by a shared trace ID and a parent-span ID, so the harness can reconstruct the tree: root run → plan LLM call → tool call → answer LLM call → guardrail check. Because every step is captured with its own inputs/outputs, you can *replay* the run and pinpoint the exact span where a multi-step run went wrong — the value tracing adds over logging only the final response, where a wrong answer gives you no signal about *which* retrieval or tool caused it. In a multi-agent run, each sub-agent's work is a sub-tree under the orchestrator's span.

**Where it appears in real systems.** In LangSmith you get automatic tracing via **integrations** (LangChain, LangGraph, OpenAI, Anthropic, CrewAI capture inputs/outputs/metadata with no code change) or **manual instrumentation** — the `@traceable` decorator on any function, the `trace` context manager for a code block, or the low-level `RunTree` API. Multi-turn conversations are linked into a **thread** via a `session_id`/`thread_id` metadata key. In OpenTelemetry terms these map onto `gen_ai.*` spans.

### What to Log (tokens, cost, latency, versions, tool I/O)

**What it is.** The per-span payload that makes a trace actionable: the prompt, the completion, token usage (input/output), latency, computed cost, tool inputs/outputs, and the exact model/version.

**How it works mechanistically.** Latency comes from span start/end; **cost** is derived by multiplying input/output token counts by the model's per-token price, so token usage must be captured per call to attribute spend to a step, a user, or a feature. Logging the **model version** (e.g. `gpt-4o-2024-11-20`) is what lets you correlate a quality regression with a silent model upgrade. Tool inputs/outputs are logged so you can see the agent passed a malformed argument or a tool returned an error the model then papered over. The one hard trade-off is **sensitive data**: prompts/completions frequently contain PII, so what you log must itself be governed (masking, retention limits) — LangSmith SaaS, for example, retains trace data for 400 days by default.

**Where it appears in real systems.** OpenTelemetry's GenAI conventions define standard signals for exactly this — **metrics** (e.g. token usage, operation duration), **model spans** for inference operations, **agent spans** for agent operations, and **events** for GenAI inputs/outputs — under the `gen_ai.*` attribute namespace, so token/latency/model attributes are named consistently across vendors.

### OpenTelemetry GenAI Semantic Conventions + LangSmith

**What it is.** The emerging *standard* (OpenTelemetry GenAI semantic conventions) for naming GenAI telemetry, and the reference *platform* (LangSmith) that ingests and visualises it.

**How it works mechanistically.** OpenTelemetry semantic conventions define agreed attribute names and span/metric shapes so that any instrumented app emits telemetry a backend can interpret without custom parsing. The GenAI subset (now maintained in the dedicated `semantic-conventions-genai` repository) specifies signals for **model spans**, **agent spans**, **metrics**, **events**, and **exceptions**, plus provider-specific conventions (OpenAI, Anthropic, Azure AI Inference, AWS Bedrock) and Model Context Protocol (MCP). The status is *Development*, meaning names can still change — cite it as the direction of travel, not a frozen v1. LangSmith sits on the platform side: it captures traces (via integrations or manual instrumentation), lets you filter/compare/share them, build dashboards, set alerts, run online evaluations, and collect human feedback.

**Where it appears in real systems.** You emit OTel `gen_ai.*` spans from an instrumented service and route them to a backend; LangSmith is a purpose-built backend where a "run = span" and you inspect the tree in the UI. Framework auto-instrumentation (LangChain/LangGraph) is the equivalent of OTel auto-instrumentation.

### Production Signals: Latency, Cost, Error-Rate, Online Quality

**What it is.** The four families of signal you actually watch: **latency** (per-step and end-to-end), **token/cost** tracking, **error/failure rates**, and **online quality** measured via sampled evals.

**How it works mechanistically.** Latency is tracked both end-to-end (user-perceived) and per-span (to find the slow step — often a tool or retrieval, not the model). Cost is aggregated from per-span token counts; you watch cost-per-request and total spend because a looping agent is a runaway-cost bug. Error/failure rates include hard errors (tool exceptions, timeouts, schema-validation failures, guardrail blocks) *and* soft failures. Because there is no ground-truth label in production, **online quality** is measured by **sampling** a fraction of live traces and scoring them — with an LLM-as-judge or a metric like groundedness (ties directly to chapter 02's online-evaluation and RAG metrics) — plus captured user feedback (thumbs-up/down). Alerts fire on threshold breaches across these signals.

**Where it appears in real systems.** LangSmith dashboards chart these signals and let you attach alerts; **rules/automations** can trigger **online evaluations** and webhooks on sampled runs, and **feedback** attaches a score to a run. This is the operational loop that closes chapter 01 (offline metrics) and chapter 02 (online eval) into a running system.

### Input Guardrails (PII, injection/jailbreak, topic filter)

**What it is.** Runtime checks applied to the *user input* (and, in RAG, retrieved content) *before* it reaches the model; an input guardrail can **reject** the input or **alter** it (e.g. mask PII).

**How it works mechanistically.** The input is passed through one or more detectors before the LLM call: a **PII detector** (NER/regex) redacts or masks entities like emails, SSNs, card numbers so sensitive data never enters the prompt or the logs; a **prompt-injection/jailbreak detector** classifies whether the input is trying to override system instructions or bypass safety (OWASP LLM01 distinguishes direct injection, indirect injection via retrieved/external content, and jailbreak as the safety-bypass variant); a **topic/off-limits filter** rejects requests outside the assistant's allowed scope (legal advice, competitor questions, politics). A failing input rail short-circuits — the model is never called — which is both safer and cheaper. Guardrails are a *runtime security control*: OWASP explicitly lists input filtering, constrained behavior, and segregating untrusted external content as prompt-injection mitigations.

**Where it appears in real systems.** NeMo Guardrails names these **input rails** ("applied to the input from the user; an input rail can reject the input... or alter the input, e.g. to mask sensitive data") and **retrieval rails** for RAG chunks, and ships built-in jailbreak/injection detection and sensitive-data (PII) masking (`sensitive_data_detection` with entity lists like `PERSON`, `EMAIL_ADDRESS`). OpenAI's Moderation endpoint (`omni-moderation-latest`) classifies input harm categories for a first-pass filter.

### Output Guardrails (toxicity, PII leak, groundedness, schema validation)

**What it is.** Runtime checks applied to the *model's output* before it is returned to the user or used downstream; an output guardrail can **block**, **regenerate**, **mask**, or **fall back**.

**How it works mechanistically.** After generation, the completion is inspected: a **toxicity/moderation** check scores harmful categories; a **PII-leakage** scan catches sensitive data the model emitted (including data the model itself pulled from a tool); a **groundedness/hallucination** check verifies the answer is supported by the retrieved context (the same faithfulness metric from chapter 02, run inline); **schema/format validation** enforces that a structured output parses against the expected JSON schema before downstream code consumes it; and business rules like **competitor-mention** filters strip disallowed content. This is the direct control for OWASP LLM05 *Improper Output Handling* — treating model output as untrusted and validating it. A failing output rail must trigger an explicit action, not silently pass.

**Where it appears in real systems.** NeMo Guardrails names these **output rails** ("applied to the output generated by the LLM; an output rail can reject the output... or alter it, e.g. removing sensitive data") and ships `self check facts`, `self check hallucination`, and moderation flows; OpenAI Moderation returns per-category `flagged`/`category_scores` on generated output; schema validation is typically a Pydantic/JSON-Schema parse step in your code (or the model's `strict` structured-output mode).

### Guardrail Placement + Fail-Open vs Fail-Closed

**What it is.** *Where* each guardrail runs (before the model, after the model, or around tool calls) and *what happens when the guardrail itself fails* (fail-open lets output through; fail-closed blocks it).

**How it works mechanistically.** Placement follows the data flow: input rails before the model, output rails after, and **execution/tool rails** around tool inputs/outputs (so a tool result containing an injection or PII is caught before the model or the user sees it). Each guardrail adds latency and can produce **false positives** (blocking a legitimate request), so you place only the checks each stage needs. The critical operational decision is the fail mode: if the guardrail service times out or errors, **fail-closed** means block and return a safe fallback (correct for PII-leak, toxicity, regulated actions — safety over availability); **fail-open** means ship the unchecked output (acceptable only for low-risk, advisory checks where availability matters more). The dangerous default is *silent* fail-open — an exception in guardrail code that gets swallowed so raw output ships unchecked.

**Where it appears in real systems.** NeMo Guardrails' five rail types map to placement directly: **input**, **dialog**, **retrieval**, **execution** (tool I/O), and **output** rails. The fail mode is your `try/except` policy around each guardrail invocation — and it must be explicit, logged as a span, and alerted on.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Trace sampling rate | Fraction of production runs fully traced / sent to online eval | Trace 100% of runs' *core* metadata (cheap: tokens, latency, cost, guardrail results); sample the *expensive* online quality evals (e.g. 1–10%) since LLM-judge scoring costs money and adds no user-facing latency if done async. |
| Guardrail placement (input / output / both / tool) | Which stages run checks | Put PII redaction + injection detection on input; toxicity + PII-leak + groundedness + schema on output; add execution rails around tools only when tools return untrusted/external data. Don't run every check at every stage — each adds latency. |
| Fail-open vs fail-closed (per guardrail) | Behavior when the guardrail errors/times out | Fail-**closed** for PII-leak, toxicity, and regulated/high-risk actions (safety > availability); fail-**open** only for low-risk advisory checks where an outage would block legitimate traffic. Never rely on a *silent* fail-open. |
| Guardrail latency budget | Max time guardrails may add to a request | Size against the end-to-end SLA; run independent guardrails in **parallel** and prefer cheap classifiers/regex over an extra LLM call. If a check can't fit the budget, move it async (log-and-alert) rather than blocking. |
| Alert thresholds | When a signal breach pages a human | Set on rate-of-change and absolute bounds: error/guardrail-block rate spike, cost-per-request above a ceiling, p95 latency breach, and online groundedness/quality dropping below a floor. Alert on the leading signal (rising block rate) not just the lagging one (user complaints). |
| Detector confidence threshold | Score above which a guardrail blocks | Tune per guardrail on a labeled set to balance false positives (blocking good requests) vs false negatives (letting harm through); high-harm categories (self-harm, PII) warrant a lower block threshold than a topic filter. |

### Worked Example: Requirement → Decision

**Given:** You are launching a customer-facing support agent for a fintech app. It answers billing questions using a RAG policy store and two read tools (`lookup_account`, `lookup_transactions`). Inputs contain PII (names, card last-4). The business requires: no PII may appear in logs or in the response beyond what the user already provided, answers must be grounded in policy docs, replies must be plain text (no leaking system prompt), and end-to-end latency ≤ 4 s. It is regulated, and a wrong or leaked answer is a compliance incident.

- **Step 1 — Identify the goal:** Operate the agent so that (a) any bad answer can be root-caused from a trace, and (b) unsafe inputs/outputs are blocked at runtime, within the latency budget.
- **Step 2 — Define inputs:** User message (PII-bearing); retrieved policy chunks (could carry indirect injection); tool outputs (account/transaction data — more PII). A model with a known version pinned.
- **Step 3 — Define outputs:** A grounded, PII-safe, plain-text answer *or* a safe refusal/fallback; plus a full trace tree per run with tokens, cost, per-step latency, model version, and each guardrail's result.
- **Step 4 — Apply constraints:** Regulated (fail-closed on PII/toxicity/groundedness); ≤ 4 s (guardrails must fit the budget → run in parallel, prefer classifiers over extra LLM calls); logs must be PII-safe (redact *before* logging); RAG means retrieved content is untrusted (retrieval rail).
- **Step 5 — Select the approach:** Trace **100%** of runs (metadata is cheap and this is regulated) with a platform like LangSmith (or OTel `gen_ai.*` spans to your backend); **input guardrails** = PII redaction + prompt-injection detection + topic filter (fail-closed); a **retrieval rail** scanning chunks for injection; **output guardrails** = PII-leak scan + toxicity + inline groundedness + JSON/plain-text schema validation, all **fail-closed**; run the independent output checks in **parallel** to stay under 4 s; **sample ~5%** of runs for an async LLM-judge quality eval; alert on rising guardrail-block rate, cost-per-request, p95 latency, and falling groundedness. *Rationale vs alternatives:* a "good system prompt only" approach is rejected because a prompt is not an enforced control (OWASP treats input/output filtering as the actual mitigation); fail-open is rejected because the domain is regulated; sampling *quality* rather than tracing everything is chosen because LLM-judge scoring is the expensive part, while span metadata is nearly free and needed for every incident.

---

## Implementation

```python
# Scenario: A customer-facing RAG agent gives a wrong refund answer in production and
# we must find WHICH step failed. We instrument each step as a span so the run can be
# replayed and the failing step (a bad retrieval vs a hallucinating LLM call) is visible.
# API verified against LangSmith observability docs (docs.langchain.com/langsmith): the
# @traceable decorator + run_type, and trace metadata for cost/version attribution.
from langsmith import traceable

MODEL_VERSION = "gpt-4o-2024-11-20"  # log the exact version to catch silent upgrades

@traceable(run_type="retriever")
def retrieve_policy(query: str) -> list[str]:
    chunks = vector_store.similarity_search(query, k=4)
    # inputs/outputs are captured automatically; add scores as metadata for debugging
    return [c.page_content for c in chunks]

@traceable(run_type="llm", metadata={"model_version": MODEL_VERSION})
def answer(query: str, context: list[str]) -> dict:
    resp = client.chat.completions.create(model=MODEL_VERSION, messages=_build(query, context))
    usage = resp.usage
    # token usage is logged per-call so cost can be attributed to THIS span
    return {"text": resp.choices[0].message.content,
            "input_tokens": usage.prompt_tokens, "output_tokens": usage.completion_tokens}

@traceable(run_type="chain")            # root span: the whole run becomes one trace tree
def handle_request(query: str) -> str:
    context = retrieve_policy(query)     # child span 1
    result = answer(query, context)      # child span 2 (nested under the root)
    return result["text"]
# Now a wrong answer is debuggable: open the trace, see if retrieve_policy returned the
# wrong chunks (retrieval bug) or the LLM ignored good context (generation/hallucination).
```

```python
# Anti-pattern: an output guardrail that fails-OPEN silently. If the guardrail call
# raises (timeout, provider error), the except swallows it and the RAW, UNCHECKED model
# output ships to the user. In a regulated app this leaks PII / toxicity on every
# guardrail outage — the worst possible failure, and it's invisible in the happy path.
def guard_output_broken(text: str) -> str:
    try:
        result = moderation_api.check(text)   # network call — can time out
        if result.flagged:
            return SAFE_FALLBACK
    except Exception:
        pass                                   # BROKEN: swallow error -> raw text ships
    return text                                # unchecked output on any guardrail failure

# Correct approach: fail-CLOSED for high-risk checks, and record the guardrail as a span
# so a spike in guardrail errors is alertable rather than silent.
from langsmith import traceable

@traceable(run_type="tool", name="output_guardrail")
def guard_output(text: str) -> str:
    try:
        result = moderation_api.check(text)
        if result.flagged:                     # detected harm -> block
            return SAFE_FALLBACK
        return text                            # explicit pass
    except Exception as e:
        # guardrail itself failed. Regulated/high-risk => fail CLOSED, not open.
        logger.error("output guardrail failed; failing closed", exc_info=e)
        return SAFE_FALLBACK                   # safe fallback, and the span records the error
# What breaks without this: the silent fail-open ships unchecked output precisely when the
# safety layer is down, and because the exception is swallowed there is no span and no
# alert — you learn about it from a compliance incident, not a dashboard. Fail-closed +
# a recorded span turns a silent leak into a visible, alertable degradation.
```

```python
# Scenario: Guardrails must add < 300 ms to a 4 s SLA, but we have three INDEPENDENT
# output checks (toxicity, PII-leak, groundedness). Running them sequentially blows the
# budget; running them in parallel keeps guardrail latency ~= the slowest single check.
import asyncio

async def output_guardrails(text: str, context: list[str]) -> str:
    # independent checks -> run concurrently instead of summing their latencies
    toxic, pii, ungrounded = await asyncio.gather(
        check_toxicity(text),
        check_pii_leak(text),
        check_groundedness(text, context),
    )
    if toxic or pii or ungrounded:             # any fail -> fail closed to safe fallback
        return SAFE_FALLBACK
    return text
# Anti-pattern would be `await check_toxicity(...); await check_pii_leak(...); ...` which
# makes guardrail latency the SUM of all three. Parallelizing (and preferring cheap
# classifiers/regex over an extra LLM call) is how you fit the guardrail latency budget.
```

---

## Common Pitfalls & Misconceptions

- **"Logging the final answer is observability."** Beginners treat an LLM app like a stateless endpoint and log only the request/response, because that is enough for a normal API. But an agent is multi-step: without a **span per step**, a wrong answer gives you no signal about *which* retrieval, tool, or LLM call failed — the correct model is a nested trace tree you can replay.
- **"A good system prompt makes guardrails unnecessary."** It feels like clearly instructing the model ("never reveal PII, stay on topic") should suffice, because prompts strongly shape behavior. But a prompt is guidance, not an enforced control — prompt injection and stochasticity defeat it, so OWASP treats input/output *filtering* as the actual mitigation; guardrails are runtime checks that run outside the model and cannot be talked out of.
- **"Fail-open is the safe default."** Engineers instinctively keep the service available, so on a guardrail error they let output through. For high-risk checks this is backwards: an unchecked output during a guardrail outage is exactly when a leak happens — fail-**closed** (safe fallback) is correct for PII/toxicity/regulated actions; reserve fail-open for low-risk advisory checks.
- **"More guardrails = safer, so run every check everywhere."** Safety sounds monotonic in the number of checks, so teams stack them at every stage. But each guardrail adds latency and false positives (blocking legitimate users); place only the checks a stage needs, run independents in parallel, and tune confidence thresholds — over-guarding degrades UX and can be as damaging as under-guarding.
- **"Tracking end-to-end latency/cost is enough."** A single aggregate number hides the culprit, because people monitor the app as one box. In a multi-step run the slow/expensive step is usually a specific tool, retrieval, or a looping sub-agent — you need **per-span** latency and per-call token/cost to attribute and fix it, not just the total.
- **"Logging prompts/completions verbatim is fine."** Capturing everything feels like good observability, but prompts and completions routinely contain PII, so raw logs become a compliance liability with retention limits — redact/mask sensitive data *before* it is written to the trace store.

---

## Key Definitions

| Term | Definition |
|---|---|
| Observability (for LLM apps) | The ability to understand a system's internal behavior from its outputs — for LLM/agentic systems, per-step traces plus quality/cost/latency signals, not just uptime/status. |
| Trace | The complete record of one operation (one agent run / user request); a collection of spans (runs) bound by a shared trace ID. |
| Span (run) | One unit of work inside a trace — an LLM call, tool call, or retrieval — with its own inputs, outputs, timing, and attributes. |
| Trace tree | The nested parent/child span structure of a multi-step (or multi-agent) run, reconstructable via trace/parent-span IDs. |
| OpenTelemetry GenAI semantic conventions | Standard `gen_ai.*` attribute names and span/metric/event definitions for GenAI telemetry (model spans, agent spans, metrics, events); status: Development. |
| LangSmith | Reference LLM-observability platform: captures traces (integrations or `@traceable`/`trace`/`RunTree`), dashboards, alerts, online evals, and feedback. |
| Guardrail (rail) | A runtime control that inspects/alters/blocks input, output, or tool I/O to enforce safety/policy outside the model's own reasoning. |
| Input guardrail | Check applied to user input (and retrieved content) before the model; can reject or alter (e.g. PII redaction, injection detection, topic filter). |
| Output guardrail | Check applied to model output before use; can block/regenerate/mask (toxicity, PII-leak, groundedness, schema validation). |
| Fail-open / fail-closed | Behavior when a guardrail itself errors: fail-open ships the unchecked output (availability); fail-closed blocks to a safe fallback (safety). |
| Online quality eval | Scoring a sampled fraction of live production traces (LLM-judge / metric / user feedback) since no ground-truth label exists in production. |

---

## Summary / Quick Recall

- Traditional metrics (uptime, HTTP 200, aggregate latency) can't tell a fluent-but-wrong answer from a good one, nor *which* step of a multi-step agent failed — you need tracing.
- A **trace** = one run; a **span** = one step (LLM/tool/retrieval); nested spans form the trace tree you replay to root-cause a failure.
- Log per span: prompt, completion, input/output **tokens**, **cost**, **latency**, tool I/O, and the exact **model version** (to catch silent upgrades) — and redact PII before logging.
- The standard is **OpenTelemetry GenAI semantic conventions** (`gen_ai.*`, status Development); the reference platform is **LangSmith** (run = span).
- Watch four signals: per-step + end-to-end **latency**, **token/cost**, **error/failure rate**, and **online quality** via *sampled* evals (ties to ch. 02); alert on leading breaches.
- **Input guardrails** (PII redaction, injection/jailbreak detection, topic filter) run before the model; **output guardrails** (toxicity, PII-leak, groundedness, schema validation) run after; **execution rails** wrap tool I/O.
- Every guardrail trades latency + false positives against safety; decide **fail-open vs fail-closed per guardrail** (closed for PII/toxicity/regulated), fit a latency budget by parallelizing, and never rely on a *silent* fail-open.

---

## Self-Check Questions

1. What is the difference between a *trace* and a *span* in LLM observability, and why is a trace tree more useful for debugging an agent than logging only the final response?

   <details><summary>Answer</summary>

   A **trace** is the full record of one operation (one agent run / user request); a **span** (a "run" in LangSmith) is a single unit of work within it — one LLM call, tool call, or retrieval — with its own inputs, outputs, timing, and attributes. Spans nest via a shared trace ID + parent-span ID into a **trace tree**. It beats logging only the final response because an agent is multi-step: a wrong final answer alone gives no signal about *which* step caused it, whereas the trace tree lets you replay the run and see (for example) that the retrieval returned the wrong chunks vs the LLM ignored good context. The tempting wrong answer — "they're the same thing / a trace is just the response log" — misses that observability's value here is *per-step* attribution, which a single response log cannot provide.

   </details>

2. You are logging every prompt and completion verbatim to your trace store for a healthcare assistant, and cost per request has started climbing on some turns. Which two instrumentation changes address the compliance risk and the cost-attribution problem respectively?

   <details><summary>Answer</summary>

   (1) **Redact/mask PII before it is written to the trace** (an input-side PII guardrail feeding the logger), because prompts/completions in healthcare carry PHI and raw logs become a compliance liability with retention limits. (2) **Capture input/output token counts per span** so cost can be attributed to the specific step/turn that's climbing — cost is derived from per-call token usage, so without per-span tokens you only see an aggregate and can't find the runaway step (often a looping agent or a bloated context). The tempting wrong answer — "just log less / turn off tracing" — throws away the debuggability you need; the fix is to *govern* what's logged and to log tokens per span, not to stop tracing.

   </details>

3. **Which TWO** of the following are correctly classified as *input* guardrails (run before the model)?
   - A. Prompt-injection / jailbreak detection on the user message.
   - B. Groundedness / hallucination check against retrieved context.
   - C. PII detection and redaction on the incoming request.
   - D. JSON schema validation of the model's structured output.
   - E. Toxicity moderation of the generated response.

   <details><summary>Answer</summary>

   **A and C.** Prompt-injection/jailbreak detection (A) and PII redaction (C) both operate on the *user input before the model runs* — they can reject or alter the input, and a failing input rail short-circuits so the model is never called (cheaper and safer). B, D, and E are all **output** guardrails: groundedness (B), schema validation (D), and toxicity moderation (E) all require the model's completion to already exist before they can run. The most tempting distractor is **B (groundedness)** — it *feels* input-adjacent because it involves the retrieved context, but it checks the *answer* against that context, so it can only run after generation.

   </details>

4. Your team sets all output guardrails to fail-open "so the service stays up." During a 20-minute outage of your moderation provider, the app keeps responding normally and no alert fires. Why is this dangerous, and what should the design have been?

   <details><summary>Answer</summary>

   Fail-open means that when the guardrail errors, the **raw, unchecked output ships** — so for the entire 20-minute outage every response bypassed toxicity/PII checks, which is exactly when a harmful or leaking output escapes; and because a *silent* fail-open swallows the error, there was no span recorded and no alert, so you'd learn about it from a complaint or compliance incident, not a dashboard. High-risk checks (toxicity, PII-leak, regulated actions) should be **fail-closed** — on guardrail error, return a safe fallback — and the guardrail failure should be recorded as a span and alerted on (rising guardrail-error rate). The tempting reasoning — "availability is always the priority" — is wrong for safety-critical checks, where an unchecked output is worse than a safe fallback; fail-open is only acceptable for low-risk advisory checks.

   </details>

5. You must add three independent output checks (toxicity, PII-leak, groundedness) to an agent with a strict 4 s end-to-end SLA, and each check takes ~150–250 ms. A naive implementation `await` s them one after another and occasionally breaches the SLA. Analyze the trade-offs and give the design that keeps safety while fitting the budget.

   <details><summary>Answer</summary>

   Running the checks **sequentially** makes guardrail latency the *sum* (~450–750 ms) of the three, which is what breaches the budget. Since the checks are **independent**, run them **concurrently** (e.g. `asyncio.gather`), so total guardrail latency is roughly the *slowest single check* (~250 ms) instead of the sum — this preserves all three checks (safety unchanged) while fitting the budget. Complementary levers: prefer cheap classifiers/regex over an extra LLM call, and if a specific check still can't fit, move it to an **async log-and-alert** path rather than dropping it. The tempting wrong answer — "drop the groundedness check to save time" — trades away safety unnecessarily; parallelization recovers the latency without weakening the guardrail set. Note the trade-off is real: more checks always cost some latency and risk false positives, so you still tune confidence thresholds per check.

   </details>

---

## Further Reading

- [LangSmith Observability](https://docs.langchain.com/langsmith/observability) — *verified 2026-07-29* — Reference LLM-observability platform overview: tracing setup, investigating traces, dashboards, alerts, online evals, and feedback.
- [LangSmith Observability concepts (traces, runs/spans, threads, instrumentation)](https://docs.langchain.com/langsmith/observability-concepts) — *verified 2026-07-29* — Precise definitions of trace/run/span, how a trace maps to OpenTelemetry spans, `@traceable`/`trace`/`RunTree`, and data retention.
- [OpenTelemetry GenAI Semantic Conventions (semantic-conventions-genai repo)](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/README.md) — *verified 2026-07-29* — The emerging standard's signal definitions: model spans, agent spans, metrics, events, exceptions, and provider/MCP conventions under `gen_ai.*` (status: Development).
- [NVIDIA NeMo Guardrails (open-source toolkit)](https://github.com/NVIDIA-NeMo/Guardrails) — *verified 2026-07-29* — Authoritative taxonomy of the five programmable rail types (input, dialog, retrieval, execution/tool, output), plus built-in jailbreak/injection detection, PII masking, fact-checking, and hallucination flows.
- [OpenAI Moderation guide](https://platform.openai.com/docs/guides/moderation) — *verified 2026-07-29* — Canonical input/output content-moderation flow: `omni-moderation-latest`, `flagged`/`categories`/`category_scores`, and treating scores as policy signals not automatic blocks.
- [OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/) — *verified 2026-07-29* — The security framing for input guardrails: direct/indirect injection and jailbreak, plus input/output filtering, output-format validation, and segregating untrusted content as mitigations.

---
