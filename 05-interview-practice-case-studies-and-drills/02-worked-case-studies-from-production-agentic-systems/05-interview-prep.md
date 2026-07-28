# Worked Case Studies from Production Agentic Systems — Interview Prep

**Section:** 05 Interview Practice → Worked Case Studies from Production Agentic Systems | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

This is the interview-prep companion to the four worked case studies in this chapter. Its purpose is **pattern transfer**: you will not be asked those exact four prompts, so the skill being drilled here is recognizing which case study a *new* prompt maps onto, and reusing its architecture, trade-offs, and safety invariants. The four re-usable engines are: **(1) bounded extract → route → dispatch pipeline** with confidence gating, HITL, idempotency, and a DLQ; **(2) NL→SQL under RBAC** where the database (not the prompt) enforces the access boundary; **(3) conversational NL→SQL** where accuracy is capped by follow-up resolution and schema grounding, verified by an execute-and-repair loop; and **(4) router/supervisor dispatch across heterogeneous sources** with per-source access checks and cited multi-source synthesis. Almost every enterprise agentic prompt is one of these — or a composition of two.

---

## Core Conceptual Questions

These test whether you understand the fundamentals well enough to reuse them under a new prompt.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| When do you choose a bounded multi-stage pipeline / router over a single free-running agent? | Use a bounded graph when the workload is *known* (fixed set of lanes/sources) — it is testable, auditable, cheaper (deterministic call count), and cannot loop unboundedly; reserve free-running agents for genuinely open-ended tasks. Cite the extraction/dispatch and query-routing cases: known lanes → Router topology, not orchestration. Name OWASP LLM06 (Excessive Agency) and LLM10 (Unbounded Consumption) as what a mega-agent invites. | "Agents are more flexible, so use an agent" — buys flexibility you don't need at the cost of predictability, cost control, and auditability you can't sacrifice. |
| Where do you enforce access control in an LLM system, and why never in the prompt? | Enforcement must live in **deterministic code below the model**: the database (RLS + column GRANTs under a `SELECT`-only `NOBYPASSRLS` role) for NL→SQL, and per-source tool checks keyed off a *verified* identity for query routing. The LLM is untrusted input; it decides *which source*, code decides *who may see what*. Filter unauthorized data out of context *before* the model sees it, plus a backend re-check (defense in depth). | "A strong, detailed system prompt tells the model to only show HR data to HR" — a prompt instruction is bypassable by injection or hallucination; it puts authorization in the least reliable layer. |
| How do you make NL→SQL reliable in production? | Accuracy is a chain: *rewrite quality × schema-linking recall × generation × repair*. Spend the budget upstream: focused schema retrieval (embed descriptions not just identifiers, over-retrieve top-8, inject PK/FK) + few-shot examples, then an **execute-and-repair loop** using the DB as a free verifier, then verify-before-present for silently-wrong results. Measure by **execution accuracy** on a golden set, never string-diff. | "Use a bigger model / prompt-tune the generator" — the generator's ceiling is set upstream; it can't recover a column that was never retrieved or a follow-up that was never resolved. |
| How do you route across heterogeneous sources accurately, and decide *not* to answer? | LLM router with **structured output + confidence**, backed by a cheap embedding-classifier cross-check; emit `{sources, scores}` and gate: below FLOOR → fallback ("can't answer"), tie within MARGIN → clarify, one ≥ τ → single route, several ≥ τ → parallel fan-out (`Send`, capped width). Sharp discriminating tool descriptions beat a bigger model. A confident wrong-source answer is worse than "I don't know." | "The LLM will just figure out the routing" — a router that always returns a source and has vague descriptions produces confident misroutes, which for cross-department data is a leak, not just a quality miss. |
| How does confidence gating with human-in-the-loop work, and what confidence do you gate on? | Each LLM decision emits *produced-not-assumed* confidence (cross-checked with deterministic signals — e.g. arithmetic consistency, OCR confidence). Gate on `min(extraction_conf, routing_conf) ≥ τ`; below τ, branch to HITL via a LangGraph `interrupt()` that checkpoints and resumes on reviewer `Command(resume=...)`. τ is the dial trading automation rate vs error rate. Extraction and routing are *separately graded* so you can abstain on either. | "Trust the model's self-reported confidence" or "gate only on extraction confidence" — a correct payload sent to the wrong handler is still an expensive wrong outcome; self-report alone is often weaker than an independent consistency check. |
| Why must dispatch be idempotent, and how do you get exactly-once effects with a DLQ? | Queues give **at-least-once** delivery, and LangGraph nodes **re-run from the top on resume** after an `interrupt()`/retry, so any pre-interrupt side effect runs again. Use a stable `doc_id` as an idempotency key so downstream handlers upsert → effectively exactly-once; keep non-idempotent side effects *after* the HITL pause; dead-letter poison messages after a bounded max-receive count so one bad doc can't jam the pipeline. | "Wrap the dispatch in try/except" or "at-most-once fire-and-forget is simpler" — try/except neither prevents duplication nor lets the interrupt propagate, and at-most-once silently drops or duplicates financial records. |
| Why separate extraction from routing (and rewrite from generation) into independently-graded decisions? | Fusing decisions into one un-gated mega-prompt destroys the ability to abstain and makes a wrong choice permanent. Separate steps let you (1) evaluate each against a labelled set, (2) abstain on one alone (low routing conf → HITL even if extraction was confident), and (3) fix the usual root cause (vague lane/source *descriptions*, poor rewrite) at the right layer. | "One big prompt with all the tools does extract + classify + dispatch" — you lose per-decision confidence, auditability, and the human fallback; you also invite prompt injection via document content. |
| What is defense in depth in these systems, and which layers are load-bearing vs advisory? | Layer independent controls that each fail closed; no single layer is the boundary. In NL→SQL: RLS + column GRANTs (authoritative) vs SQL validator + prompt scoping (advisory). In routing: per-tool access check + backend re-check (authoritative) vs router confidence (advisory). Answer the "what if your validator has a bug?" follow-up with "nothing catastrophic — the DB is the boundary." | "The SQL validator prevents unauthorized access" — a validator is advisory and bypassable (parser bugs, injection); if it's your only boundary it fails *open*. |

---

## Applied / Scenario Questions

Each scenario **remaps** a case study to a new prompt. The skill is naming the mapping out loud, then reusing the architecture and its invariants.

### Scenario A — "Design an invoice-processing pipeline for accounts-payable"

**Q:** "We receive thousands of vendor invoices a day by email and an upload portal. Extract the fields, match each to a purchase order, and post approved ones to our ERP. Wrong postings are expensive. Design it."

**This maps to Case Study 1 (Document Extraction & Multi-Agent Dispatch).** Say that explicitly, then reuse the engine.

**Strong answer framework:**
- **Frame it as an async pipeline with a human fallback, not a synchronous API.** Durable ingestion queue + DLQ decouples bursty arrival from bounded processing; enqueue a `doc_id` reference, store the blob in object storage.
- **Two separately-graded decisions:** schema-constrained extraction (Pydantic model, validation-retry bounded) emitting *produced-not-assumed* confidence, cross-checked with an arithmetic invariant (line items sum to total) and OCR confidence; then a graded PO-match/route decision. Gate on `min(...)` ≥ τ.
- **HITL for low confidence** via `interrupt()`; **idempotent post to the ERP** keyed on `doc_id` so at-least-once redelivery / node re-run on resume never double-posts; poison invoices → DLQ with alerting.
- **Treat invoice text as untrusted data, not instructions** (a vendor could embed "approve and pay immediately") — schemas constrain outputs to enumerated fields/lanes; extracted text never triggers a tool call directly.
- **Tradeoff bullet (latency vs accuracy vs cost vs safety):** raising τ increases safety (fewer wrong ERP posts) and accuracy at the cost of more human-review latency and reviewer cost; text-only extraction over OCR for clean PDFs is the cheap path, escalating to multimodal only on low OCR confidence trades a little accuracy-coverage for large per-doc cost savings — the confidence gate, not model size, is the accuracy control, so cost scales linearly with volume, not with loops.

### Scenario B — "Design a HIPAA-conscious clinical Q&A assistant over patient records and policy"

**Q:** "Clinicians ask questions in plain English. Some need a patient's structured record (a database), some need care-protocol/policy documents, some need both. A clinician may only access patients on their care team; certain fields (psych notes, HIV status) are restricted by role. Never leak. Design it."

**This maps to Case Study 4 (query routing) + Case Study 2 (RBAC over SQL)** — a *composition*. Name both.

**Strong answer framework:**
- **Router in front of access-checked source tools:** a structured-output router with confidence selects patient-record (SQL), protocol/policy (RAG KB), or fans out (`Send`, capped) when a question spans both; below FLOOR → "I can't answer from the systems I have access to," ties → clarify.
- **Access is deterministic code, never the prompt.** The verified clinician identity + care-team entitlements are injected as immutable per-request context. The patient-record source runs under a `SELECT`-only, `NOBYPASSRLS` role with **RLS** scoping rows to the clinician's care team and **column GRANTs** removing restricted fields — so even a hallucinated or injected query physically cannot return an unauthorized patient or field. Backend re-checks (defense in depth).
- **Grounded, cited synthesis:** each claim attributed to its source (protocol vs record), empty/inaccessible evidence → "not available," never a fabricated value; the SQL validator (AST parse + allowlist + read-only + timeout) is advisory layer-4, not the boundary.
- **Auditability is mandatory in a compliance setting:** log question, generated SQL, executing role, row-count, sources hit, and access-check outcomes immutably — you need to *prove* no leak occurred.
- **Tradeoff bullet (latency vs accuracy vs cost vs safety):** database-enforced RLS adds query-planner overhead and is harder to unit-test than an app-layer filter (latency/dev-cost) but is the only design where a wrong query is merely an over-broad-but-authorized result or a clean permission error — never a breach; scope-keyed answer caching (key = normalized question + authorization scope) cuts latency/cost but a question-only key would be a cache-driven RBAC bypass, so you accept a lower hit rate for safety.

---

## System Design / Architecture Questions

**Q:** "Design an internal enterprise assistant that (a) lets employees **upload documents** (contracts, invoices, expense receipts) and have structured data extracted and filed to the right downstream system, AND (b) answers **conversational analytics questions** over a data warehouse with strict per-user access — 'what did my team spend on travel last quarter?'. One product, both capabilities."

This deliberately combines **Case Study 1 (extract → route → dispatch)**, **Case Study 3 (conversational NL→SQL)**, and **Case Study 2 (RBAC + SQL validation)**. Do not try to build one agent that does everything — compose the three bounded engines behind a thin front-door router.

**Approach:**

1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity).** Establish two *different* interaction models: the ingestion path is an **async pipeline** (p95 seconds-to-a-minute acceptable, ~90% auto / ~10% human review, high downstream-error cost, PII), while the analytics path is **interactive** (p95 < ~8 s, execution-accuracy target ≥ ~90% on a golden set, strict row/column RBAC, read-only). Confirm identity is verified upstream (SSO/OIDC) and entitlements come from an authoritative identity/HR system. State throughput and burstiness for uploads.

2. **Propose high-level architecture (agent topology, retrieval layer, guardrails).** A **front-door classifier** splits each request into the upload-ingestion lane or the analytics-chat lane (this is the Router topology again).
   - *Ingestion lane:* durable queue + DLQ → preprocess (OCR/layout) → schema-constrained extraction (confidence, arithmetic cross-check) → graded classifier/router → confidence gate → idempotent dispatch to the right downstream system, with a HITL `interrupt()` lane for low-confidence docs. Idempotency key = `doc_id`.
   - *Analytics lane:* per-`thread_id` conversation state → context-resolution (rewrite follow-up into a standalone question, carry *structured* scope) → schema grounding (retrieved schema slice + few-shot, PK/FK injected) → SQL generation → **execute-and-repair loop** under a `SELECT`-only `NOBYPASSRLS` role with **RLS + column GRANTs** and `statement_timeout` → verify-before-present → answer with echoed SQL + restatement.
   - *Shared guardrails:* untrusted-input discipline (document text and questions are data, not instructions), AST-based SQL validation (advisory), bounded work everywhere (no free-running loops), and an immutable audit log across both lanes.

3. **Justify choices and name tradeoffs explicitly (cost, latency, complexity, security).**
   - *Why two lanes, not one agent:* the two workloads have opposite latency profiles and different safety boundaries (idempotent dispatch vs read-only RBAC); one mega-agent would be un-auditable, could loop, and would fuse decisions you need to grade separately (LLM06/LLM10).
   - *Security:* the analytics boundary is the database (RLS + column GRANTs), not the SQL validator or prompt — so a validator bug or injection yields an over-broad-but-authorized result, never a leak; ingestion never lets extracted text trigger a dispatch tool.
   - *Cost/latency:* deterministic call counts on both lanes (bounded DAG + capped repair/retry) → cost scales with volume, not with loops; extraction modality and router model are per-request cost levers; the confidence gate τ (ingestion) and execution-accuracy eval (analytics) are the accuracy controls, decoupled from model size.
   - *Complexity you accept:* two pipelines and a front-door split, more roles/GRANTs to manage, a review UI, and eval harnesses for extraction/routing accuracy and NL→SQL execution accuracy — worth it because each is independently testable and no single defect is catastrophic.

---

## Vocabulary That Signals Expertise

Use these naturally — they show you've reasoned about the boundary, not just the happy path:

- **Router / supervisor topology** — when the workload is a known set of lanes/sources; a classification step dispatches to specialists and results are synthesized centrally (vs decentralized handoffs, which add coordination bugs for independent verticals).
- **Confidence gating on `min(...)`** — gate on the *minimum* of separately-graded confidences (extraction, routing) so you can abstain on either decision independently.
- **Human-in-the-loop (`interrupt()`)** — the mechanism for pausing a bounded graph, checkpointing state, and resuming on a reviewer decision; pair it with the "nodes re-run on resume" caveat.
- **Dead-letter queue (DLQ)** — where poison messages go after a bounded max-receive count so one bad document can't jam the pipeline.
- **Idempotent dispatch / idempotency key** — a stable `doc_id` that turns at-least-once delivery into an exactly-once downstream effect via upsert.
- **Row-Level Security (RLS) + column GRANTs** — database-enforced, authoritative access; the boundary that survives a wrong or hostile query.
- **`SELECT`-only, `NOBYPASSRLS` role** — least-privilege execution identity; caps blast radius even if the validator has a bug.
- **SQL allowlist (AST parse, fails closed)** — permit only approved tables/columns/statement-types by parsing to an AST, not regex; advisory defense in depth.
- **Schema grounding / schema linking** — the dominant NL→SQL accuracy lever; embed descriptions, over-retrieve, inject PK/FK.
- **Execute-and-repair (self-repair) loop** — use the database as a free verifier, feeding the verbatim error back for bounded regeneration.
- **Context resolution / query rewrite with structured scope** — fold a follow-up into a standalone question by carrying resolved filters/dimensions, not raw transcript.
- **Multi-route fan-out (`Send`, bounded width)** — parallel dispatch when a query legitimately spans sources; latency ≈ slowest branch, not the sum.
- **Citation / attribution invariant** — every claim carries the citation of the source that produced it; facts may not migrate between sources.
- **Grounding invariant / abstention** — state only what's in the evidence; empty evidence → "not available," never a guess.
- **Defense in depth (fails closed)** — layered independent controls where no single layer is load-bearing for security.
- **Verify-before-present** — cheap sanity checks (row bounds, no all-NULL keys, non-degenerate aggregate) to catch silently-wrong-but-plausible results the repair loop can't.
- **Execution accuracy** — the correct NL→SQL metric (compare result sets on a fixed snapshot), never SQL string diff.

---

## Vocabulary That Signals Weakness

Avoid these — each signals that you'd put the boundary in the wrong (bypassable) place or don't know the failure modes:

- **"The LLM will just figure out the routing"** — routing is a graded, evaluable decision needing confidence gating and a fallback; hand-waving it produces confident misroutes, which for per-user data is a leak.
- **"Execute the generated SQL directly"** — no validation, no read-only role, no timeout; one injection or hallucination reads or melts the warehouse.
- **"One big prompt with all the tools"** — fuses decisions you must grade separately, invites Excessive Agency (LLM06) and Unbounded Consumption (LLM10), and is un-auditable.
- **"RBAC in the prompt" / "a strong system prompt enforces access"** — authorization in the least reliable, most bypassable layer; a prompt-injection or hallucination away from a leak.
- **"Just append a `WHERE region = ...` clause"** — string manipulation on untrusted model output; a trailing `;`, existing `WHERE`, `UNION`, or subquery neutralizes it, and it needs a broad DB user.
- **"Trust the model's confidence"** — self-reported confidence is often weaker than an independent consistency check (arithmetic, OCR, execution).
- **"Grade NL→SQL by comparing the SQL strings"** — two correct queries differ syntactically; you want execution accuracy.
- **"The validator prevents unauthorized access"** — a validator is advisory and bypassable; the database is the boundary.
- **"Retry with try/except so it doesn't crash"** — swallows the `interrupt()` exception and doesn't prevent duplicate side effects; you need idempotency keys and bounded, explicit retries.
- **"Dump the whole schema into the prompt for accuracy"** — irrelevant tables distract the model, *lower* accuracy, and inflate cost/latency; retrieve a focused slice.
- **"Let the agent loop until it's done"** — unbounded cost/latency (LLM10); bound the graph with a hard cap and a human-review fallback.

---

## STAR Answer Frame

A plausible, generic production narrative drawn from building an extraction-and-dispatch pipeline — no proprietary specifics.

**Situation:** An operations team was manually keying data from ~40k inbound documents/day (invoices, claims, contracts) arriving via email, an upload API, and an SFTP drop. A first attempt used a single LLM agent handed OCR text and all three downstream "post" tools; it occasionally posted the wrong document type to the wrong system, had no way to say "I'm not sure," and once double-posted a batch when the queue redelivered messages after a worker crash.

**Task:** I owned redesigning the pipeline to auto-process the bulk of documents safely, keep wrong data out of downstream systems, and make delivery exactly-once — without blowing the reviewer team's capacity (~10% of volume).

**Action:** I replaced the mega-agent with a **bounded LangGraph state machine per document**: a durable queue + DLQ (enqueuing a `doc_id`, blob in object storage), OCR/layout preprocess, **schema-constrained extraction** (Pydantic model, bounded validation-retry) emitting a confidence I cross-checked against an arithmetic invariant (line items vs total) and OCR confidence, and a **separately-graded classifier/router**. A conditional edge gated on `min(extraction_conf, routing_conf) ≥ τ`; below τ the doc hit a **human-review `interrupt()`** and resumed on the reviewer's correction. Dispatch was made **idempotent on `doc_id`** (downstream upsert), with non-idempotent effects kept after the HITL pause so a node re-run on resume couldn't double-post; poison docs dead-lettered after a bounded max-receive count. I tuned τ on a labelled sample to hit the target auto-rate and built an eval harness scoring extraction and routing accuracy independently.

**Result:** Auto-processing settled at roughly 90% of volume within reviewer capacity, wrong-destination postings dropped to near zero (routing evaluated independently against the labelled set), duplicate downstream posts went to zero after idempotency + DLQ shipped, and per-document cost became deterministic — it scaled linearly with volume instead of spiking on hard documents, because the pipeline is a bounded DAG with no free-running loop. Reviewer corrections were fed back as new labelled examples to raise the auto-rate over time.

---

## Red Flags Interviewers Watch For

Specific to agentic-system design — these are the failures that separate a shipped-it engineer from a slideware one:

- **Over-engineering with agents.** Reaching for a free-running multi-agent orchestration when the workload is a *known* set of lanes/sources that a bounded router/pipeline handles more cheaply, testably, and auditably. Interviewers watch whether you can justify *not* using an agent.
- **Unbounded loops.** A router→tool or extract→repair loop with no hard cap and no human-review fallback → runaway cost and latency (OWASP LLM10). Every loop must carry a cap and a deterministic worst-case call count.
- **No access control per source / boundary in the wrong layer.** Enforcing authorization in the prompt or an app-layer string filter instead of database RLS + column GRANTs (analytics) or deterministic per-tool checks keyed off a verified identity (routing). The tell: you can't answer "what if your validator/prompt has a bug?" with "nothing catastrophic — the boundary is below the model."
- **Executing unvalidated / irreversible actions.** Running raw LLM-generated SQL directly, or letting extracted document text trigger a dispatch tool — no AST validation, no read-only role, no timeout, no idempotency; one injection exfiltrates data, melts the warehouse, or fires a wrong irreversible action.
- **Fusing decisions that must be graded separately.** Extraction + routing + dispatch (or rewrite + generation) in one un-gated prompt — you lose per-decision confidence, the ability to abstain, and independent evaluation.
- **No abstention / no fallback.** A router that always returns a source, or a synthesizer that answers on empty evidence — confident wrong-source or fabricated answers, which for per-user data is a leak, not just a quality miss.
- **Trusting model self-report over independent verification.** No arithmetic/OCR cross-check on extraction, no execute-and-repair or verify-before-present on NL→SQL, no execution-accuracy eval — you can't distinguish confident-and-wrong from correct.
- **Treating retrieved/document content as instructions.** Not delimiting or labelling untrusted context (document text, ticket bodies, schema comments) → indirect prompt injection (OWASP LLM01) steering routing, extraction, or synthesis.
