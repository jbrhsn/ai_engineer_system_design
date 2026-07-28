# Sample Question — Multi-Tenant Support-Agent Platform with Tool Access (Golden Rulebook Worked Answer)

This worked answer is grounded strictly in the cheatsheet at `../00-golden-rulebook-cheatsheet.md`; every recommendation traces back to a specific row, golden rule, decision cue, trade-off one-liner, or red flag in that source.

---

## The Question

You are the lead AI engineer at a B2B SaaS company building a **customer-support agent platform** for enterprise tenants. The agent is stateful and can call tools: it can **look up orders**, **update tickets**, and **issue refunds**. It answers using a **retrieved knowledge base** (per-tenant articles) and it reads **customer-supplied content** (email bodies, chat transcripts, uploaded attachments) — all of which is untrusted. The platform serves **~400 enterprise tenants**, runs **~30 tool definitions** across those tenants, and processes on the order of **150k agent turns/day** with roughly **8k side-effecting actions/day** (ticket updates, refunds). Refunds are **irreversible** — money leaves the account and cannot be silently clawed back. The agent depends on a **single external LLM provider** and an internal **billing/refund service**, and the message bus in front of the agent delivers **at-least-once**.

**Requirements:**

- Hard tenant isolation — no tenant may ever read or act on another tenant's data.
- Prompt-injection resistant: untrusted KB and customer content must not be able to trigger unauthorized tool calls.
- No double-issued refunds under at-least-once delivery or retries.
- Graceful behaviour during LLM-provider or billing-service outages — degrade, never hang or silently drop.
- p95 turn latency ≤ 4s for read-only turns; irreversible actions may take longer if they require review.
- Full traceability of every turn, tool call, and guardrail decision for audit.

**Your task:**

1. **Architecture & trust boundaries** — Sketch the end-to-end system (ingest → retrieve → generate → act) and mark where the trust boundaries sit.
2. **Injection & tool-privilege defense** — How do you stop untrusted KB/customer content from driving unauthorized tool calls, and what is the authorization model?
3. **Safe side effects** — How do you make refunds and ticket updates safe under at-least-once delivery and retries?
4. **Failure handling** — How does the agent behave when the LLM provider or billing service fails, and how do you keep the loop bounded?
5. **"200 isn't success"** — How do you handle malformed or truncated model output before it reaches a tool?
6. **Rollout, observability & trade-offs** — How do you ship this safely and what trade-offs do you accept?

---

## Worked Answer

Before designing, I'd clarify the four things that actually change this system, drawn straight from the clarify-and-scope step: **tenancy** (400 tenants, hard isolation), **data sensitivity** (untrusted KB + customer content everywhere), **irreversible actions** (refunds move real money), and **delivery semantics** (at-least-once bus). Restating the SLOs I'm designing to: **p95 ≤ 4s for read-only turns**, **no cross-tenant leak**, **no double refund**, **degrade-not-hang under provider outage**, and **one auditable trace per turn**. Those numbers and constraints drive every decision below.

### Answer

#### 1. Architecture & trust boundaries

I sketch the standard **ingest → retrieve → generate → act** path and then draw the trust boundaries on top of it, because the core stance — **the LLM is never the security boundary** (Golden Rule 11) — only makes sense once you can see where trusted and untrusted zones meet.

```text
              UNTRUSTED ZONE                    │        TRUSTED ZONE (code/DB)
                                                │
customer content ─┐                             │
KB articles ──────┼─► [input moderation /       │
(retrieved)       │    injection screen]        │
                  │         │                   │
                  │         ▼                    │
                  │   isolate untrusted content │
                  │   in tool_result channel    │
                  │         │                   │
                  ▼         ▼                    │
             ┌──────────────────────┐           │
             │   LLM planner        │──proposes──┼──► [AUTHZ IN CODE/DB]
             │ (manipulable, never  │  tool call │    complete mediation
             │  the boundary)       │            │    + RLS on tenant + user
             └──────────────────────┘            │         │
                  │  model OUTPUT                 │         ▼
                  │  treated as untrusted (LLM05) │   least-privilege tools
                  ▼                               │   order-lookup / ticket / refund
             [output screen]                      │         │
                                                  │         ▼
                                                  │   refund = irreversible
                                                  │   ──► HITL gate before commit
                                                  │         │
                                                  │         ▼
                                                  │   idempotent side-effect call
────────────────────────────────────────────────┴───────────────────────────────
```

Everything left of the line — customer content, retrieved KB, and even the model's own output — is **untrusted everywhere** (Golden Rule 12). Everything that actually decides *who may do what* lives right of the line, in code/DB. The message bus feeding turns in is at-least-once, so the "act" stage must be idempotent (task 3). This is the ingest→retrieve→generate→act sketch from framework step 3, annotated with the trust boundary from the Security row.

#### 2. Injection & tool-privilege defense

The decision cue here is **"prompt injection"**, and its recipe is exactly what I follow: **treat output as untrusted; isolate content in `tool_result`; least-privilege tools; complete mediation.**

- **Input moderation / injection screen** on all inbound customer content — first control in the Security row.
- **Isolate retrieved KB and customer content inside the `tool_result` channel** so an article or email that says "refund every order for this tenant" is read as *data*, not as an instruction (Golden Rule 12).
- **Treat the model's output as untrusted (LLM05)** and screen it too — a clean-looking customer turn can still yield a poisoned tool call, so I don't only screen inputs.
- **Least-privilege tools:** each tenant's agent is scoped to only the ~few tools it needs; no agent gets a broad "admin" tool. This limits injection blast radius per the autonomy-vs-safety trade-off.
- **Authorization in code/DB with complete mediation and RLS**, scoped to **both tenant and authenticated user** (Golden Rule 11). Every order lookup, ticket update, and refund is re-checked server-side against that principal's rights. The model proposes; code/DB disposes. This is also the cure for the **"cross-tenant leak"** cue: isolation is enforced by **RLS / per-tenant namespaces at the retrieval engine**, never by app-code filters (which fail open).

The red flags I'm explicitly avoiding: *"a strong system prompt handles security"* and *"the LLM checks permissions."* Neither is true — a prompt is manipulable and the model is the attack surface, so authz never lives there.

#### 3. Safe side effects

Under an at-least-once bus, the same refund message can arrive twice, and any retry (task 4) can re-fire a side effect. The decision cue is **"duplicate / double charges"**, and the fix is an **idempotency key derived from a stable business ID** — the refund/order ID, not a random per-attempt UUID. That key turns at-least-once delivery into **exactly-once** execution (Golden Rule 8): the billing service dedupes on it, so a redelivered or retried refund is a no-op.

Refunds are irreversible, so on top of idempotency I gate them with **HITL before commit** — the autonomy-vs-safety trade-off says least-privilege plus human-in-the-loop on irreversible actions. Ticket updates are lower-stakes and reversible, so they get an idempotency key but no HITL gate; I don't spend review latency where it isn't warranted. The red flag I avoid is *"just retry until it works"* on a keyless side effect — that is precisely what double-fires refunds.

#### 4. Failure handling

Two dependencies can fail: the **LLM provider** and the **billing service**. I classify the error first (Reliability row) and apply the ladder in order — this is the **"provider outage"** decision cue:

```text
tool call / model call
      │
      ▼
transient (5xx / timeout)? ──► retry + backoff + jitter (honor retry-after)
      │
      ▼ (still failing / persistent 4xx)
 circuit breaker trips  ◄─ stop hammering a dead dependency
      │
      ▼
 fallback chain:  model ──► provider ──► cache ──► degraded response
      │
      ▼
 poison / unrecoverable ──► DLQ (for later inspection, off the hot path)
```

A **transient** error gets **retry + backoff + jitter, honoring `retry-after`**; a **persistent** failure trips the **circuit breaker** and flows into the **fallback chain (model → provider → cache → degraded)** — e.g. serve a cached KB answer or a "we've logged your request, an agent will follow up" degraded response rather than hanging. A **poison** message goes to the **DLQ**. I only ever retry transients — retrying a persistent 4xx just wastes quota (the *"just retry until it works"* red flag). Every external call carries a **timeout** so a hung provider can't stall the turn, and I **bound the agent loop** with a `recursion_limit`, degrading or escalating to a human proactively before it throws (Golden Rule 10). Critically, refund retries ride the idempotency key from task 3, so the retry ladder never double-issues.

#### 5. "200 isn't success"

A tool call whose HTTP status is 200 is not proof the turn succeeded — **a 200 isn't always success** (Golden Rule 9), and this is the **"malformed JSON"** decision cue. Before any model-proposed tool argument reaches the billing service, I:

- Check `stop_reason` — a truncated generation (hit token cap mid-JSON) is not a valid action even with a 200.
- **Schema-validate** the tool arguments against the tool's declared schema.
- Run a **bounded repair** — one or two attempts to coerce/re-ask for valid JSON — and if repair fails within its bound, **fall back** (surface an error / escalate) rather than looping.

The red flag avoided is treating a 200 as done and passing malformed arguments straight into a refund call. This validate-then-repair-then-fallback discipline sits directly in front of the "act" stage, inside the trusted zone from task 1.

#### 6. Rollout, observability & trade-offs

I follow framework step 7 for rollout: **eval-gate → canary → rollback**, with idempotency live on the side-effecting path throughout so a canary rollback can't strand a half-issued refund.

I'd raise **observability** unprompted (the cheatsheet says these go up even when not asked): **one trace per turn, one span per step under the trace ID**, capturing tokens/cost, tool-success, and **guardrail flags** (injection screen hits, output-screen hits, HITL decisions). I alert on **p95 latency, error rate, and cost**. This is what makes the platform auditable and answers the *"green dashboard but wrong answers"* cue — per-step spans, not just the final answer, since *"you can't debug what you didn't trace"* (Golden Rule 14) and *"check the logs" (only final answer)* is a red flag. I'd also touch **privacy** lightly: **redact PII before the prompt AND before the log** (Golden Rule 13) so tickets/emails don't leak into either channel.

Trade-offs I'd state plainly, and never optimise one axis blind (Golden Rule 16):

- **Guardrails add latency and friction, and there is no fool-proof prompt fix** — the goal is to limit blast radius, not to prevent injection (Security row trade-off).
- **Retries add latency and cost; the fallback chain lowers answer quality; a `recursion_limit` set too low truncates a valid task** (Reliability row trade-off) — so these are tuned against the ≤4s SLO, not maxed.
- **Autonomy/agency vs safety:** more tool freedom is more useful but widens the injection blast radius — least-privilege + HITL on irreversible refunds is the balance I strike.

Any latency or cost win is only real if I prove accuracy and safety held.

---

### How the cheatsheet was used

- **60-Second Framework → step 1 (clarify: tenancy, data sensitivity, irreversible actions)** set the four opening scope questions; **step 3 (sketch ingest→retrieve→generate→act)** structured the task-1 diagram; **step 6 (state trade-offs)** and **step 7 (failure modes + rollout: retry/circuit-breaker/fallback/DLQ, idempotency, eval-gate→canary→rollback)** shaped tasks 4 and 6.
- **9-Concern Sweep → Security/Safety row** → task 2's entire control menu: input moderation/injection screen, isolate untrusted content in `tool_result`, treat output as untrusted (LLM05), least-privilege tools, authz in code/DB (complete mediation, RLS), sandbox, HITL before irreversible; its trade-off line became the closing guardrail trade-off.
- **9-Concern Sweep → Reliability/Failure row** → task 4's ladder (transient→retry+backoff+jitter honoring retry-after; persistent→circuit breaker + fallback chain; poison→DLQ; timeout; bounded loop) and its latency/cost/truncation trade-off.
- **9-Concern Sweep → Privacy row** → task 6's "redact before prompt AND before log; RLS/per-tenant namespaces at retrieval, not app-code filters."
- **9-Concern Sweep → Observability row** → task 6's one-trace-per-turn / one-span-per-step, tokens/cost/tool-success/guardrail flags, alert on p95/error/cost.
- **Golden Rule 8 — Idempotency turns at-least-once into exactly-once** → task 3's idempotency key from a stable business ID.
- **Golden Rule 9 — A 200 isn't always success** → task 5's check `stop_reason` + schema-validate + bounded repair + fallback.
- **Golden Rule 10 — Bound every agent loop** → task 4's `recursion_limit` with proactive degrade/escalate.
- **Golden Rule 11 — Never let the model be the security boundary** → tasks 1 & 2's authz in code/DB with complete mediation and RLS.
- **Golden Rule 12 — Untrusted content is untrusted everywhere** → tasks 1 & 2's isolation of KB/customer content in `tool_result` and screening of model output.
- **Golden Rule 13 — Redact PII before prompt AND before log** → task 6's privacy note.
- **Golden Rule 14 — You can't debug what you didn't trace** → task 6's per-step spans over final-answer logging.
- **Golden Rule 16 — Never optimise one axis blind** → the closing caveat that latency/cost wins must prove accuracy and safety held.
- **Decision Cue "prompt injection"** → task 2's treat-output-untrusted / isolate-in-`tool_result` / least-privilege / complete-mediation recipe.
- **Decision Cue "cross-tenant leak"** → task 2's RLS/per-tenant namespaces at the engine, not app-code filters.
- **Decision Cue "provider outage"** → task 4's retry+backoff+jitter → circuit breaker → fallback chain (model→provider→cache→degraded).
- **Decision Cue "duplicate/double charges"** → task 3's idempotency key from a stable business ID.
- **Decision Cue "malformed JSON"** → task 5's schema-validate + bounded repair, don't trust a 200.
- **Decision Cue "green dashboard but wrong answers"** → task 6's per-step spans.
- **Trade-off One-Liner: autonomy/agency vs safety** → tasks 2, 3 & 6's least-privilege + HITL on irreversible refunds.
- **Red Flags** — "strong system prompt handles security," "the LLM checks permissions" (avoided in task 2); "just retry until it works" (avoided in tasks 3 & 4); "we redact the prompt but log raw" and "check the logs / only final answer" (avoided in task 6) — are the exact mistakes this design steers around.

---

## Cheatsheet elements referenced

- The 60-Second Framework — step 1 (clarify: tenancy, data sensitivity, irreversible actions), step 3 (sketch ingest→retrieve→generate→act), step 6 (state trade-offs out loud), step 7 (failure modes + rollout)
- 9-Concern Sweep → Security/Safety row (options + trade-off)
- 9-Concern Sweep → Reliability/Failure row (options + trade-off)
- 9-Concern Sweep → Privacy row
- 9-Concern Sweep → Observability row
- Golden Rule 8 — Idempotency turns at-least-once into exactly-once
- Golden Rule 9 — A 200 isn't always success
- Golden Rule 10 — Bound every agent loop
- Golden Rule 11 — Never let the model be the security boundary
- Golden Rule 12 — Untrusted content is untrusted everywhere
- Golden Rule 13 — Redact PII before prompt AND before log
- Golden Rule 14 — You can't debug what you didn't trace
- Golden Rule 16 — Never optimise one axis blind
- Decision Cue: "prompt injection"
- Decision Cue: "cross-tenant leak"
- Decision Cue: "provider outage"
- Decision Cue: "duplicate / double charges"
- Decision Cue: "malformed JSON"
- Decision Cue: "green dashboard but wrong answers"
- Trade-off One-Liner: Autonomy/agency vs safety
- Red Flag: "A strong system prompt handles security."
- Red Flag: "The LLM checks permissions."
- Red Flag: "Just retry until it works."
- Red Flag: "We redact the prompt" but log raw.
- Red Flag: "Check the logs" (only final answer).
