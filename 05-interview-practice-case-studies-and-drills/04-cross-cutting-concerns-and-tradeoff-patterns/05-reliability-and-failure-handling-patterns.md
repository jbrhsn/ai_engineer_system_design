# Reliability & Failure-Handling Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — "what happens when the model, tool, or provider fails or misbehaves?" is the reliability follow-up in every agentic/RAG design, and it separates candidates who ship demos from those who ship production: retries, timeouts, circuit breakers, fallback chains, idempotency, dead-letter queues, bounded loops, and human escape hatches.

---

## TL;DR

Production AI systems fail in ways ordinary services do not: providers return 429/529, models emit malformed JSON, tools time out, and agent loops run away. Reliability is layered defense — retry transient errors with exponential backoff *and jitter*, bound every call with a timeout, trip a circuit breaker when a dependency is sick, and fall back down a chain (alternate model/provider → cache → degraded response) before finally dead-lettering what you cannot process. Two AI-specific twists sit on top: LLM non-determinism forces schema validation + repair, and unbounded agent recursion forces a hard step limit with a human-in-the-loop escape hatch for low confidence. **The one thing to remember: retries fix transient faults, but only an idempotency key makes a retry safe — never retry a side-effecting call without one.**

---

## ELI5 — Explain It Like I'm 5

Imagine sending a delivery driver to drop a package at a busy building. Sometimes the door is briefly locked (a transient error) — the driver waits a random little bit and knocks again instead of pounding non-stop (retry with backoff and jitter). If the whole building is on fire, the driver stops trying and goes to the backup address instead of burning up (circuit breaker + fallback). But here is the trap: if the driver already slid the package under the door and then "retries" because they didn't hear a reply, you now have two packages inside — unless each package has a unique sticker the receptionist checks so duplicates get thrown away (idempotency key). Packages that can never be delivered go into a "return to sender" bin for someone to inspect later (dead-letter queue), and if the driver has circled the block twenty times, a dispatcher pulls them off the route (bounded loop + human escalation).

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Classify a failure as transient (retry), persistent (fall back), or poison (dead-letter), and pick the matching pattern.
- [ ] Design a retry → circuit-breaker → fallback-chain → DLQ pipeline and justify each layer against its trade-off.
- [ ] Make an at-least-once delivery path safe with idempotency keys, and explain why retries without them cause double-effects.
- [ ] Handle AI-specific failure modes — malformed structured output, 429/529 provider errors, and runaway agent loops — using validation/repair, backoff honoring `retry-after`, and recursion limits with a human escape hatch.

---

## Visual Overview

### Request lifecycle: retry → circuit breaker → fallback → DLQ

```
Incoming request (idempotency_key attached)
        │
        ▼
   ┌─────────────┐  transient error (429/529/5xx/timeout)
   │  Try call   │──────────────► retry w/ backoff + jitter (honor retry-after)
   └─────────────┘                       │ exhausted retries
        │ success                        ▼
        │                        ┌────────────────┐  breaker OPEN?
        │                        │ Circuit breaker │──yes──► skip primary
        │                        └────────────────┘
        │                                │ no / half-open
        ▼                                ▼
   return result            ┌──────────────────────────┐
                            │  Fallback chain           │
                            │  primary model/provider   │
                            │     └► fallback model      │
                            │         └► cached response  │
                            │             └► degraded msg  │
                            └──────────────────────────┘
                                        │ all exhausted
                                        ▼
                              ┌──────────────────┐
                              │ Dead-letter queue │──► human inspects/redrives
                              └──────────────────┘
```

### Circuit-breaker state machine

```
        failures < threshold
        ┌───────────────┐
        ▼               │
   ┌─────────┐  failures ≥ threshold   ┌────────┐
   │ CLOSED  │────────────────────────►│  OPEN  │
   │ (pass)  │                          │ (fail  │
   └─────────┘◄───────────┐            │  fast) │
        ▲   trial success  │            └────────┘
        │                  │                 │ cooldown elapsed
        │             ┌───────────┐          ▼
        └─────────────│ HALF-OPEN │◄─────────┘
          (all trials │ (1 probe) │
           pass)      └───────────┘
                            │ probe fails
                            └──────────► back to OPEN
```

---

## The Core Problem

Every remote dependency in an AI system — the LLM provider, the vector DB, an external tool API — can fail, and the failures are *not* all alike. A 429 rate-limit or a 529 overloaded error is transient and worth retrying; a 400 malformed-request or 404 is persistent and retrying only wastes quota; a message the consumer can never process is "poison" and must be quarantined so it stops blocking the queue. Treating all three the same — retrying everything, or retrying nothing — is the most common reliability mistake, and it is exactly what an interviewer probes when they ask "what happens when the provider is down?"

AI systems add two failure modes ordinary services don't have. First, **non-determinism**: the same prompt can return valid JSON one call and prose-wrapped or truncated JSON the next, so the *success* response itself may be unusable and must be validated and repaired. Second, **unbounded autonomy**: an agent that decides its own next step can loop forever (tool → reason → tool → reason…), burning tokens and latency until something external stops it. Reliability engineering here is therefore about *classifying* each failure and applying the cheapest safe response — while never letting a retry silently duplicate a side effect and never letting a loop run away.

Two properties must be separated, because they are fixed by different mechanisms:

- **Safety of retry** — whether re-executing a call is harmless. A read is naturally safe; a write (charge a card, post a message, dispatch an order) is not, unless it carries an idempotency key the server dedupes on.
- **Recoverability of the failure** — whether the failure will clear on its own (retry), needs a different path (fallback), or never clears (dead-letter). A design that retries an unrecoverable failure forever is as broken as one that dead-letters a transient blip.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Retry + exponential backoff + jitter** | On transient error, wait `base·2ⁿ` (± random jitter), retry up to a cap | Rides out transient blips (429/529/5xx/timeout) automatically | Wasted latency/quota if applied to non-transient errors; thundering herd without jitter | The error is transient and the call is safe/idempotent |
| **Timeout** | Abort a call that exceeds a deadline | Frees threads/connections; bounds tail latency; prevents hangs | Too-tight cuts off slow-but-valid work; must be tuned per hop | Every network/model call — a call with no timeout is a latent hang |
| **Circuit breaker** | Track failures; when a threshold trips, fail fast without calling the sick dependency; probe to recover | Stops hammering a down dependency; fast failure lets fallbacks fire | Added state/config; a mis-tuned breaker rejects healthy traffic | A dependency is repeatedly failing and retries only make it worse |
| **Fallback chain** | Try primary; on failure step to fallback model → provider → cache → degraded response | Graceful degradation instead of a hard error | Fallback may be lower quality; complexity; must detect "bad" success too | Availability matters more than always using the best path |
| **Idempotency key** | Attach a unique key; server dedupes repeats of the same operation | Makes at-least-once delivery / retries safe against double-effects | Requires server-side dedupe store; key design | Any retry-able call with a side effect (write/charge/dispatch) |
| **Dead-letter queue (DLQ)** | After N failed deliveries, move the message to a separate queue | Isolates poison messages; unblocks the main queue; enables inspection/redrive | Needs monitoring/alerting or messages silently rot | Messages can fail permanently and must not block others |
| **Schema validation + repair** | Validate model output against a schema; on failure, re-prompt with the error or use structured-output mode | Turns non-deterministic text into reliable typed data | Repair round-trips add latency/cost; can loop if unbounded | Any LLM output consumed by downstream code |
| **Bounded agent loop** | Cap super-steps / recursion; degrade or escalate near the limit | Prevents runaway token/latency burn | Too-low cap truncates legitimately long tasks | Any agent that chooses its own next step |
| **Human-in-the-loop escape hatch** | On low confidence / limit reached, pause and route to a human | Safety net for ambiguous or high-stakes cases | Adds human latency; needs a queue/UI | Low-confidence or high-consequence decisions |

**Retry + exponential backoff + jitter** — on a transient error the client sleeps, then retries; each successive sleep grows exponentially (`base·2ⁿ`) so early retries are fast but repeated failures back off, and a random *jitter* term de-synchronizes many clients so they don't all retry in the same instant (a "thundering herd"). Mechanically this is a decorator/wrapper around the call (e.g. `tenacity.wait_random_exponential`). The official SDKs already do this: the Anthropic SDK retries transient failures (connection errors, rate limits, 5xx) twice by default and honors the `retry-after` header; OpenAI documents the identical pattern. The controlling knobs are max attempts and the backoff base — and critically, retry *only* transient status codes, never a 400.

**Timeout** — a deadline after which an in-flight call is aborted and treated as failed. It bounds tail latency and prevents a slow dependency from pinning your connection pool. It appears as a per-call `timeout=` argument or a client-level default. Anthropic's docs warn that a large `max_tokens` without streaming can hit a 10-minute request timeout or a dropped idle connection — for long generations, stream (see `01-latency-and-response-time-patterns.md`) or use the batch API rather than a single blocking call.

**Circuit breaker** — a small state machine wrapping a dependency: **CLOSED** passes calls and counts failures; once failures cross a threshold it flips to **OPEN** and fails fast (no call attempted) for a cooldown; then **HALF-OPEN** lets a single probe through — success closes it, failure re-opens it. It appears as a wrapper around the provider client. Its value is precisely that it *stops* retries when a dependency is truly down, letting the fallback chain fire immediately instead of every request paying full retry-timeout latency.

**Fallback chain** — an ordered list of alternatives tried in sequence: primary model → cheaper/alternate model → alternate provider → cached prior response → static degraded message. It appears as ordered `try/except` steps or a router. The subtle point: you must detect not just *thrown* errors but *bad successes* — Anthropic returns a `stop_reason` such as `refusal` or `max_tokens` on an otherwise-200 response, so "success" still needs inspection before you trust it.

**Idempotency key** — a caller-supplied unique token (e.g. a UUID) attached to a side-effecting operation; the server records processed keys and, on a repeat, returns the original result instead of re-executing. It appears as an `Idempotency-Key` header or a dedupe check keyed on a business ID. This is what makes at-least-once delivery and retries *safe*: without it, a retried "dispatch order" posts the order twice.

**Dead-letter queue** — a separate queue that receives messages a consumer failed to process after a configured number of receive attempts. In Amazon SQS this is a **redrive policy** with a `maxReceiveCount`: after that many receives without deletion, SQS moves the message to the DLQ, where you can examine it, fix the cause, and *redrive* it back. Best practice from the docs: set the DLQ's retention period *longer* than the source queue's, because the message's original enqueue timestamp is preserved on standard queues.

**Schema validation + repair** — validate the model's output against a schema (Pydantic/JSON Schema); on a validation error, either re-prompt the model with the error message to fix it, or switch to a provider structured-output mode that constrains generation. It appears as a parse-and-validate step after the completion call. Bound the repair attempts — an unbounded repair loop is just another runaway loop.

**Bounded agent loop** — a hard cap on how many steps an agent may take. In LangGraph this is the `recursion_limit` config key (default 1000 as of v1.0.6), which raises `GraphRecursionError` when exceeded; the graph also exposes a `RemainingSteps` managed value so a node can *proactively* degrade before the limit. It appears as `config={"recursion_limit": N}` passed to `invoke`/`stream`. Reactive handling catches the error outside the graph; proactive handling routes to a fallback node inside it.

**Human-in-the-loop escape hatch** — when confidence is low or a limit is hit, the system pauses and hands off to a human rather than guessing. In LangGraph this is `interrupt()`, which pauses the graph and waits for a `Command(resume=...)` value. It appears as an interrupt node on high-stakes or low-confidence branches.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Max retry attempts | How many times a transient failure is retried | 2–5 for interactive paths; retry only transient codes (429/529/5xx/timeout), never 4xx like 400/404 |
| Backoff base + jitter | Growth rate and de-synchronization of retries | Use exponential base 2 with full random jitter; always honor a `retry-after` header over your own schedule |
| Per-call timeout | Deadline before a call is aborted | Set slightly above the observed p99 of a *healthy* call; stream or batch long generations instead of raising it to minutes |
| Breaker failure threshold / cooldown | When to trip OPEN and how long to stay | Trip on a failure *rate* over a window, not a raw count; cooldown ≈ the dependency's typical recovery time |
| SQS `maxReceiveCount` | Receives before a message is dead-lettered | Set high enough for legitimate transient retries but low enough that poison messages quarantine fast (often 3–5) |
| DLQ retention period | How long dead-lettered messages persist | Set *longer* than the source queue's retention so you have time to inspect/redrive |
| LangGraph `recursion_limit` | Max super-steps before `GraphRecursionError` | Set to the worst-case steps a *correct* task needs plus a small margin — never leave it effectively unbounded |
| Repair-attempt cap | Re-prompts allowed to fix invalid output | 1–2; beyond that, fall back or escalate — a repair loop is a runaway loop |

### Worked Example: Requirement → Decision

**Given:** An agentic workflow calls an LLM to extract a structured order (items + quantities) from an email, then calls an internal `dispatch_order` API. In production you see: occasional 529 overloaded errors from the model, occasional malformed JSON, and — worst — duplicate orders when the pipeline retries after a timeout.

- **Step 1 — Identify the goal:** Reliable extraction *and* exactly-once dispatch, degrading gracefully when the model is overloaded, with no duplicate orders.
- **Step 2 — Define inputs:** The email text, the model completion (possibly malformed or a 529), and a `dispatch_order` side-effecting call.
- **Step 3 — Define outputs:** A validated order object and at most one dispatch per email, or a dead-lettered item for human review if extraction cannot succeed.
- **Step 4 — Apply constraints:** Model calls are transient-failure-prone (retry 529/5xx with backoff honoring `retry-after`); output is non-deterministic (validate + one repair attempt); `dispatch_order` has a side effect (must be idempotent); the pipeline is at-least-once (retries can double-fire).
- **Step 5 — Select the approach:** Wrap the model call in **retry + backoff + jitter** for 529/5xx, **validate the JSON with one repair re-prompt** on parse failure, and attach an **idempotency key** (derived from the email's message-ID) to `dispatch_order` so retries dedupe server-side. If extraction still fails after retries+repair, **dead-letter** the email for a human. Rationale vs alternatives: retrying the *dispatch* without an idempotency key is exactly what caused the duplicates, so idempotency — not fewer retries — is the fix; and dead-lettering (not infinite retry) is correct because a genuinely unparseable email is poison, not transient.

---

## Implementation

```python
# Scenario: our order-dispatch call is retried after a network timeout, and
# some emails now dispatch TWICE. At-least-once delivery means the same request
# can arrive more than once, so the WRITE must be idempotent, not just retried.
import uuid
import anthropic
from tenacity import retry, wait_random_exponential, stop_after_attempt, retry_if_exception_type

client = anthropic.Anthropic()

# Retry ONLY transient errors, with exponential backoff + jitter. The SDK also
# retries internally and honors `retry-after`; this wraps our own call site.
@retry(
    wait=wait_random_exponential(min=1, max=30),      # jitter de-syncs clients
    stop=stop_after_attempt(4),                        # bounded, not infinite
    retry=retry_if_exception_type(                     # transient only
        (anthropic.RateLimitError, anthropic.InternalServerError)  # 429 / 5xx / 529
    ),
)
def extract_order(email_text: str) -> str:
    resp = client.messages.create(
        model="claude-opus-4",
        max_tokens=512,
        messages=[{"role": "user", "content": f"Extract order as JSON:\n{email_text}"}],
    )
    # A 200 is not always usable: inspect stop_reason before trusting it.
    if resp.stop_reason in ("refusal", "max_tokens"):
        raise ValueError(f"unusable completion: {resp.stop_reason}")
    return resp.content[0].text

def dispatch_order(order: dict, idempotency_key: str) -> None:
    # The internal API dedupes on this key: a retry with the SAME key is a no-op,
    # so a retried dispatch can never create a second order.
    internal_api.post("/dispatch", json=order, headers={"Idempotency-Key": idempotency_key})

def handle(email):
    order_json = extract_order(email.text)
    order = validate_and_repair(order_json)               # schema + bounded repair
    # Key derived from a stable business ID (the email message-id), NOT random,
    # so every retry of THIS email reuses the same key.
    dispatch_order(order, idempotency_key=email.message_id)
```

```python
# Anti-pattern: retrying a side-effecting dispatch with a FRESH key each attempt.
# Every retry looks like a brand-new order to the server → double-posts.
@retry(stop=stop_after_attempt(3))
def dispatch_broken(order: dict) -> None:
    key = str(uuid.uuid4())                 # BUG: new key on every retry
    internal_api.post("/dispatch", json=order, headers={"Idempotency-Key": key})
# Retry #2 after a timeout sends a *different* key → server treats it as new → 2 orders.

# Correct approach: derive the key from a stable business identifier so all
# retries of the same logical operation share one key and dedupe server-side.
def dispatch_fixed(order: dict, message_id: str) -> None:
    internal_api.post("/dispatch", json=order, headers={"Idempotency-Key": message_id})
# Retry #2 sends the SAME key → server returns the original result → exactly one order.
```

```python
# Scenario: an overloaded primary model and non-deterministic JSON keep breaking
# the pipeline. We need graceful degradation (fallback chain) and a bounded
# agent loop with a human escape hatch — availability over always-best-quality.
from langgraph.graph import StateGraph, START, END
from langgraph.errors import GraphRecursionError
from langgraph.types import interrupt, Command

def generate(state):                       # fallback chain: primary → fallback → cache
    for attempt in (call_primary, call_fallback_model, read_cache):
        try:
            out = attempt(state["query"])
            if out is not None:
                return {"answer": out}
        except Exception:
            continue                        # step down the chain on failure
    return {"answer": "Service is busy — please try again shortly.", "degraded": True}

def route(state):                          # human escape hatch on low confidence
    if state.get("confidence", 1.0) < 0.6:
        decision = interrupt("Low confidence — human review needed")  # pause the graph
        return {"answer": decision}
    return state

builder = StateGraph(dict)
builder.add_node("generate", generate)
builder.add_node("route", route)
builder.add_edge(START, "generate")
builder.add_edge("generate", "route")
builder.add_edge("route", END)
graph = builder.compile()

# Bound the loop: cap super-steps so a runaway agent can't burn tokens forever.
try:
    result = graph.invoke({"query": q}, config={"recursion_limit": 25})
except GraphRecursionError:
    result = {"answer": "Could not complete within step budget.", "escalate": True}
```

---

## Common Pitfalls & Misconceptions

- **Retrying non-idempotent side effects** — beginners add a retry decorator to "make it reliable" without noticing the call charges a card or posts an order, so a retried timeout produces duplicates; the correct mental model is that retries are only safe on reads or on writes carrying a stable idempotency key the server dedupes on.
- **Retrying everything (including 4xx) with no jitter** — it feels safer to retry any error, but retrying a 400/404 just wastes quota, and synchronized retries without jitter create a thundering herd that worsens the outage; retry only transient codes (429/529/5xx/timeout) with exponential backoff *plus* random jitter, and honor `retry-after`.
- **Treating a 200 as success without checking `stop_reason`** — people assume an HTTP 200 means a usable answer, but the model may have refused or hit `max_tokens` mid-output, or returned malformed JSON; a reliable path inspects `stop_reason` and validates the schema before trusting the response, repairing or falling back otherwise.
- **Leaving the agent loop effectively unbounded** — a high or default recursion limit feels harmless until an agent loops on a bad tool result and burns thousands of tokens; set `recursion_limit` to the worst-case steps a *correct* task needs plus a small margin, and degrade or escalate proactively as `RemainingSteps` runs low.

---

## Key Definitions

| Term | Definition |
|---|---|
| Exponential backoff | A retry strategy where the wait between attempts grows as `base·2ⁿ`, so repeated failures back off progressively rather than hammering the dependency. |
| Jitter | A random component added to backoff delays so many clients don't retry in the same instant (avoiding a synchronized "thundering herd"). |
| Circuit breaker | A state machine (CLOSED/OPEN/HALF-OPEN) that fails fast without calling a dependency that is repeatedly failing, then probes to recover. |
| Idempotency key | A caller-supplied unique token attached to a side-effecting operation so the server dedupes repeats, making retries and at-least-once delivery safe. |
| At-least-once delivery | A messaging guarantee that a message is delivered one or more times (never zero), which means consumers must tolerate duplicates. |
| Dead-letter queue (DLQ) | A separate queue receiving messages that failed processing after a configured number of receive attempts, isolating "poison" messages for inspection. |
| Fallback chain | An ordered set of alternatives (fallback model → provider → cache → degraded response) tried in sequence when the primary path fails. |
| Recursion limit | A hard cap on agent super-steps/turns (LangGraph `recursion_limit`) that raises an error when exceeded, preventing runaway loops. |
| Structured output / schema repair | Validating a model's output against a schema and re-prompting or constraining generation to obtain reliable typed data despite non-determinism. |

---

## Summary / Quick Recall

- Classify first: transient → retry, persistent → fall back, poison → dead-letter. Same treatment for all three is the classic mistake.
- Retry only transient codes (429/529/5xx/timeout) with exponential backoff *and* jitter, honoring `retry-after`; never retry a 4xx.
- A retry is only safe with an idempotency key on any side-effecting call — this is the fix for at-least-once double-effects.
- Circuit breakers stop hammering a down dependency so fallbacks fire fast; fallback chains degrade gracefully (model → provider → cache → static).
- An HTTP 200 isn't always usable: check `stop_reason` and validate the schema, repairing (bounded) or falling back on bad output.
- Bound every agent loop with a `recursion_limit`; degrade proactively with `RemainingSteps` and escalate low-confidence cases to a human.
- DLQ retention should outlast the source queue's; monitor the DLQ or poison messages rot silently.

---

## Self-Check Questions

1. Name the three failure classes a reliability design must distinguish, and the pattern that matches each.

   <details><summary>Answer</summary>

   Transient failures (429/529/5xx/timeout) → **retry with backoff + jitter**; persistent failures (a path is down or degraded) → **fall back** down a chain (alternate model/provider/cache/degraded response); poison messages that can never be processed → **dead-letter queue**. Answering only "retry everything" is wrong because retrying a persistent 400 wastes quota and dead-lettering a transient blip needlessly quarantines recoverable work.

   </details>

2. Your pipeline retries an order-dispatch call after a network timeout and you observe duplicate orders. What is the fix, and why is "reduce the retry count" not the right one?

   <details><summary>Answer</summary>

   Attach a stable **idempotency key** (e.g. derived from the email/message ID) to the dispatch call so the server dedupes repeats and processes it exactly once. Reducing the retry count is the tempting wrong answer: it lowers the *probability* of a duplicate but doesn't eliminate it — any single retry after a timeout can still double-fire, because the first attempt may have succeeded server-side before the timeout. Only idempotency makes the retry actually safe.

   </details>

3. A model provider starts returning 529 overloaded errors under a traffic spike, and your service's latency balloons because every request retries several times before failing. How should the design respond?

   <details><summary>Answer</summary>

   Put a **circuit breaker** in front of the provider: after the failure rate crosses a threshold it trips OPEN and fails fast (no call, no retry-timeout wait), letting the **fallback chain** (alternate model/provider, cache, or a degraded "busy, try again" message) fire immediately; a HALF-OPEN probe later tests recovery. Continuing to retry every request is the wrong instinct — it keeps latency high and adds load to an already-overloaded provider. The breaker's job is precisely to *stop* retries when a dependency is genuinely down. (Backoff should still honor the `retry-after` header when retrying is warranted.)

   </details>

4. **Which TWO** of the following are AI-specific reliability concerns that a generic web-service reliability playbook does *not* already cover?
   - A. Retrying transient 5xx errors with exponential backoff
   - B. Validating and repairing non-deterministic structured output from the model
   - C. Adding a timeout to every network call
   - D. Bounding an agent's self-directed loop with a recursion limit
   - E. Moving repeatedly-failed messages to a dead-letter queue

   <details><summary>Answer</summary>

   **B and D.** LLM non-determinism means a *successful* response can still be malformed/refused/truncated, so schema validation + repair (B) is unique to AI systems; and an agent that chooses its own next step can loop indefinitely, so a recursion/step limit (D) addresses runaway autonomy that generic services don't have. A, C, and E are standard distributed-systems patterns that apply to any service. E is the most tempting wrong pick because DLQs feel "advanced," but they're a general messaging pattern (e.g. SQS `maxReceiveCount`), not AI-specific.

   </details>

5. Two proposals for an agent that occasionally burns thousands of tokens looping on a bad tool result: (a) lower `max_tokens` per call, or (b) set a `recursion_limit` and route to a fallback/human node as `RemainingSteps` runs low. Which addresses the root cause, and why?

   <details><summary>Answer</summary>

   Option (b). The root cause is an *unbounded number of steps* — the agent keeps re-invoking tools — so capping the loop with `recursion_limit` (and proactively degrading/escalating via `RemainingSteps` before the limit throws `GraphRecursionError`) directly stops the runaway. Option (a) is the tempting distractor: lowering `max_tokens` shrinks each individual call but does nothing about the *number* of calls, so the loop still runs away, just with shorter turns. This is why you bound the loop, not just the per-call output. (Bounding hops also helps latency — see `01-latency-and-response-time-patterns.md` — and capacity — see `02-scalability-and-throughput-patterns.md`.)

   </details>

---

## Further Reading

- [Amazon SQS — Using dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — *verified 2026-07-29* — the redrive policy and `maxReceiveCount`, redrive-allow policy, and the best practice that a DLQ's retention should exceed the source queue's.
- [Anthropic — Errors](https://docs.anthropic.com/en/api/errors) — *verified 2026-07-29* — HTTP status codes (429 `rate_limit_error`, 500 `api_error`, 529 `overloaded_error`), the SDKs' automatic retry-with-backoff honoring `retry-after`, and guidance on long requests/timeouts.
- [Anthropic — Streaming messages (error recovery)](https://docs.anthropic.com/en/docs/build-with-claude/streaming) — *verified 2026-07-29* — mid-stream `error` events (e.g. `overloaded_error`), the SSE event flow, and the capture-and-resume strategy for recovering an interrupted stream.
- [Anthropic — Stop reasons and fallback](https://docs.anthropic.com/en/docs/build-with-claude/handling-stop-reasons) — *verified 2026-07-29* — why a 200 isn't always usable: handling `max_tokens`, `refusal`, `pause_turn`, and retrying on a fallback model.
- [OpenAI — Rate limits](https://platform.openai.com/docs/guides/rate-limits) — *verified 2026-07-29* — RPM/TPM limits, rate-limit response headers, and retry-with-exponential-backoff-and-jitter mitigation (Tenacity, backoff, and a manual implementation).
- [LangGraph — Graph API (recursion limit)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — the `recursion_limit` config key (default 1000), `GraphRecursionError`, and proactive vs reactive handling via the `RemainingSteps` managed value.
