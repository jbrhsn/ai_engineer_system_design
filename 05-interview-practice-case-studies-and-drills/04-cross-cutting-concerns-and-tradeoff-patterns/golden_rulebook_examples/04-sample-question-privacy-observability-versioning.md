# Sample Question — Regulated Multi-Tenant RAG Assistant (Golden Rulebook Worked Answer)

This worked answer is grounded strictly in the cheatsheet at `../00-golden-rulebook-cheatsheet.md`; every recommendation traces back to a specific row, golden rule, decision cue, trade-off line, or red flag in that source.

---

## The Question

**Context.** You are the tech lead on an ML platform team standing up a multi-tenant RAG assistant over regulated customer records — support tickets, some carrying healthcare and financial fields — for **300 enterprise tenants**. The corpus holds roughly **40M documents** across **three residency regions** (EU, US, APAC). Records contain multiple PII classes (names, emails, national IDs, card fragments, clinical notes). Your team ships changes frequently: **prompt edits several times a week, a base-model swap roughly monthly, and an embedding-model swap once or twice a year.** Legal has flagged two live concerns: a tenant has invoked **GDPR right-to-erasure** (contractual SLA: **erasure completed within 30 days, audited**), and users are reporting **wrong answers while every dashboard stays green.**

**Requirements:**

- **Tenant isolation:** tenant A must be physically unable to retrieve tenant B's data — zero cross-tenant leakage.
- **PII confinement:** no raw PII in prompts sent to the model or in any stored log/trace.
- **Residency:** each tenant's data and processing pinned to its home region (EU/US/APAC).
- **Right-to-erasure:** on request, the record must stop being retrievable everywhere within the 30-day SLA, with an audit trail proving it.
- **Observability:** localize a bad answer to the exact pipeline stage; catch quality regressions before users report them.
- **Change management:** ship model/prompt/embedder changes without breakage or unbounded blast radius, and reproduce any past answer on demand.

**Your task:**

1. **Architecture & data lifecycle** — sketch the end-to-end pipeline with trust and tenant/region boundaries.
2. **Tenant isolation + PII redaction** — enforce isolation and redact PII on both the prompt path and the log/trace path.
3. **Right-to-erasure cascade** — design the deletion cascade and its audit trail against the 30-day SLA.
4. **Per-request observability** — design tracing that solves "green dashboard but wrong answers."
5. **Safe change management** — version pinning, eval-gate → canary → rollback for a base-model swap, dual-index cutover for an embedder swap, and reproducing a past answer.
6. **Trade-offs & what you'd monitor** — the axes you'd balance and the signals you'd watch.

---

## Worked Answer

Before designing, I'll clarify and scope from the cheatsheet's Clarify step: **data sensitivity** (PII — yes, multiple classes; multi-tenant — 300 tenants; residency/deletion duty — yes, three regions + GDPR erasure), **debuggability** (need to localize which step broke live; user feedback available), and **change cadence** (prompts weekly, model monthly, embedder ~yearly; must reproduce past answers). That routes me straight into the Privacy, Observability, and Versioning concerns, and I'll name the binding SLOs/obligations up front: **zero cross-tenant leakage, no raw PII in prompt or log, region-pinned processing, erasure within 30 days with audit, and reproducibility of any historical answer.**

### Answer

#### 1. Architecture & data lifecycle

I walk the standard RAG lifecycle — **ingest → chunk → embed → index → retrieve → rerank → generate → cite** — and overlay the boundaries the requirements demand: a **region boundary** (EU/US/APAC each get their own stack, region-pinned) and a **tenant boundary** enforced at the retrieval engine, not in app code. Untrusted document content is isolated from the control path, and model output is treated as untrusted downstream.

```text
                REGION-PINNED STACK (one per EU / US / APAC)
  ┌───────────────────────────── tenant boundary at engine ─────────────────────────────┐
  │                                                                                      │
  │  ingest ─► REDACT PII ─► chunk ─► embed ─► INDEX (per-tenant namespace / RLS)         │
  │   (regulated source)      │                                                          │
  │                           └── redaction happens BEFORE anything is persisted          │
  │                                                                                      │
  │  query ─► scope to tenant+region ─► retrieve ─► rerank ─► generate ─► cite ─► answer  │
  │              (engine-enforced)                     │                                  │
  └────────────────────────────────────────────────────┼──────────────────────────────┘
                                                        ▼
                          trace/log path ─► REDACT PII before write ─► store (TTL)
```

The critical lifecycle point (Golden Rule 13): PII must be redacted **before** it ever lands in the index, the prompt, or the log — so redaction sits at ingest and again on the log/trace path, not as an afterthought.

#### 2. Tenant isolation + PII redaction

**Isolation → enforced at the engine.** This is the "cross-tenant leak" decision cue: **RLS or per-tenant namespaces at the retrieval engine, not app-code filters.** I scope every query to `(tenant, region)` at the vector store / query layer, so tenant A's query physically cannot reach tenant B's vectors. The Privacy row's trade-off warns **app-code filters fail open** — a forgotten `WHERE tenant_id = ?` leaks everything — whereas engine-enforced isolation fails closed. Per Golden Rule 11, **the LLM is never the security boundary**: authz lives in code/DB with complete mediation and RLS.

**PII redaction → both paths.** Golden Rule 13's other half: **redact before the prompt AND before the log.** Redacting only the model input is the Red Flag "we redact the prompt (but log it raw)" — confinement solved at the model, broken at the log. So redaction runs on the prompt path and the trace/log path, paired with **retention TTL** on stored logs and **region-pin** to honor residency. Redaction at ingest (task 1) means stored vectors never carry raw PII either.

#### 3. Right-to-erasure cascade

This is the "GDPR erasure" cue, and Golden Rule 13's core insight: **deletion must cascade** or the "forgotten" record still answers. Deleting the source row alone leaves the PII propagated into chunks, embedding vectors, and caches — the "no training ≠ no retention" trap. Against the 30-day SLA the cascade is: **source → chunks → embeddings → caches → re-index, then write an audit trail.**

```text
GDPR erasure request  (SLA: 30 days, audited)
        │
        ▼
[1] delete SOURCE record ─────────────► region-scoped, tenant-scoped
        │
        ▼
[2] delete derived CHUNKS
        │
        ▼
[3] delete EMBEDDINGS in the index   ◄── the step people forget
        │
        ▼
[4] purge CACHES
        │
        ▼
[5] RE-INDEX  ─►  write AUDIT TRAIL (who/what/when — proves erasure)
```

The audit trail is what lets us demonstrate compliance to Legal within the SLA window; without step 3 the record stays retrievable even after the source is gone.

#### 4. Per-request observability

This is the "green dashboard but wrong answers" cue exactly. Green dashboards prove the endpoint is *up* — they say nothing about whether retrieval fetched the wrong chunks or the reranker dropped the right one. The remedy is **one trace per request, one span per step under a propagated trace ID** (Golden Rule 14: you can't debug what you didn't trace). The Red Flag "check the logs" is the trap — a log of only the final answer can't localize the failure.

```text
one trace per request  (propagated trace ID)
  ├─ span: retrieve   → which chunks? recall?
  ├─ span: rerank     → did the right chunk survive?
  ├─ span: generate   → tokens / cost / cache-hit / tool-success / guardrail flags (OTel GenAI)
  └─ signals: p95 / error / cost alerts  +  online quality + drift
              user feedback ──► eval
```

On top of spans I capture the LLM signals — **tokens / cost / cache-hit / tool-success / guardrail flags** — and alert on **p95 / error / cost**. To close the "wrong while green" gap I add **online quality + drift monitoring** and route **feedback → eval**, so regressions surface as monitored signals, not just complaints. Per task 2, captured spans must not hold raw PII — the Observability row explicitly flags **"PII in captured prompts"** as a hazard.

#### 5. Safe change management

Foundation is Golden Rule 15: **pin every version — never `-latest` in prod — and log the (model, prompt, index) triple** per output. That triple is the direct answer to "which version produced this answer?": with it logged, I can **reproduce any past answer** by rehydrating the exact model/prompt/index used.

- **New base model (monthly):** the "ship a new model safely" cue and framework step 7 — **eval-gate → canary (small %, ramp) → instant rollback**, previous version kept revertible. The Versioning trade-off keeps me honest: **passing offline eval ≠ safe at 100%**, so I ramp rather than flip, watching the task-4 online quality/drift signals on the canary.
- **Embedder swap (~yearly):** the "swap embedding model" cue — **dual-index cutover, flipping the query embedder and reader in lockstep.** You can't mix vectors from two embedders in one index, so build the new index in parallel, then flip query-side embedder and reader together at cutover, old index live for instant rollback.

```text
dual-index cutover (embedder swap)
  old embedder ──► old index ─┐
                              ├─ query embedder + reader flip in LOCKSTEP
  new embedder ──► new index ─┘        │
                                       ▼
                          canary ramp; prev index kept revertible
```

Every change is designed eval-first (Golden Rule 7) and shipped eval-gate → canary → rollback.

#### 6. Trade-offs & what you'd monitor

Stating trade-offs out loud (framework step 6):

- **Privacy — utility/simplicity vs exposure.** Redaction, cascade deletes, and engine-level isolation cost simplicity and some retrieval utility, but a "no training" claim doesn't buy "no retention," and a filter that fails open isn't isolation.
- **Versioning — velocity vs reproducibility.** Floating to `-latest` ships faster but loses reproduce/revert; I pin and eval-gate instead. Also **rollout cost vs blast radius** (canary costs effort, shrinks blast radius).
- **Observability — sampling vs full-capture cost** and **metric-cardinality blow-up**; I sample traces and bound label cardinality.

**What I'd monitor:** cross-tenant access anomalies, erasure-SLA aging (records approaching 30 days), redaction-failure rate on both paths, per-step span success, online quality/drift, canary quality delta, and the (model, prompt, index) triple coverage per output. Per Golden Rule 16, no single axis gets optimized blind.

---

### How the cheatsheet was used

- **Task 1 (architecture):** the 60-Second Framework's lifecycle **ingest → chunk → embed → index → retrieve → rerank → generate → cite** became the skeleton; **region-pin** (Privacy row) and **isolate untrusted content / treat output as untrusted** (Security row) set the boundaries; Golden Rule 13's "redact before prompt AND log" placed redaction at ingest and the log path.
- **Task 2 (isolation + redaction):** Decision Cue **"cross-tenant leak → RLS / per-tenant namespaces at engine, not app-code filters"** → engine-enforced scoping; Privacy row **"app-code filters fail open"** → refusal to use app filters; Golden Rule 11 **"LLM is never the security boundary"** → authz in code/DB; Golden Rule 13 + Red Flag **"we redact the prompt (but log it raw)"** → dual-path redaction; Privacy row **retention TTL / region-pin** → stored-log controls.
- **Task 3 (erasure):** Decision Cue **"GDPR erasure → cascade delete → chunks → embeddings → caches, re-index; audit trail"** → the five-step cascade; Golden Rule 13 **"deletion cascades to embeddings, or the 'forgotten' record still answers"** and the **"no training ≠ no retention"** trap → why step 3 is non-negotiable.
- **Task 4 (observability):** Decision Cue **"green dashboard but wrong answers → online quality + drift; per-step spans not endpoint logs"** → the whole diagnosis; Observability row **one trace/request, one span/step under propagated trace ID + LLM signals (OTel GenAI) + p95/error/cost + feedback → eval** → the span design; Golden Rule 14 **"can't debug what you didn't trace"**; Red Flag **"check the logs"** → the trap avoided; Observability row **"PII in captured prompts"** → spans stay redacted.
- **Task 5 (change management):** Golden Rule 15 **"pin versions; never `-latest`; log the (model, prompt, index) triple"** → reproducibility foundation; Decision Cue **"which version produced this answer? → triple per output"** → reproduce-a-past-answer; Decision Cue **"ship a new model safely → eval-gate → canary → rollback"** → base-model rollout; Decision Cue **"swap embedding model → dual-index cutover; flip query embedder + reader in lockstep"** → embedder swap; Golden Rule 7 **"eval-gate every change."**
- **Task 6 (trade-offs / monitoring):** Trade-off One-Liner **velocity vs reproducibility**; Privacy/Observability/Versioning row trade-offs (utility vs exposure; sampling vs full-capture cost + metric cardinality; rollout cost vs blast radius; offline eval ≠ safe at 100%); Golden Rule 16 **"never optimise one axis blind."**

---

## Cheatsheet elements referenced

- The 60-Second Framework — lifecycle sketch (step 3), request-path walk (step 4), state trade-offs (step 6), failure modes + rollout eval-gate → canary → rollback (step 7)
- Clarifying scope — data sensitivity, debuggability, change cadence
- 9-Concern Sweep → Data Privacy/Governance row (options + trade-off)
- 9-Concern Sweep → Observability/Monitoring row (options + trade-off)
- 9-Concern Sweep → Versioning/Change-mgmt row (options + trade-off)
- 9-Concern Sweep → Security row (LLM is never the security boundary; isolate untrusted content; treat output as untrusted)
- Golden Rule 7 — Design eval before build; eval-gate every change
- Golden Rule 11 — Never let the model be the security boundary; authz in code/DB, complete mediation, RLS
- Golden Rule 13 — Redact PII before prompt AND before log; deletion cascades to embeddings
- Golden Rule 14 — You can't debug what you didn't trace
- Golden Rule 15 — Pin versions; never `-latest`; log the (model, prompt, index) triple
- Golden Rule 16 — Never optimise one axis blind
- Decision Cue: "cross-tenant leak"
- Decision Cue: "GDPR erasure"
- Decision Cue: "which version produced this answer?"
- Decision Cue: "ship a new model safely"
- Decision Cue: "swap embedding model"
- Decision Cue: "green dashboard but wrong answers"
- Trade-off One-Liner: velocity vs reproducibility
- Red Flag: "we redact the prompt" (but log it raw)
- Red Flag: "check the logs"
