# Data Privacy & Governance Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — as soon as a design touches user documents, customer records, or multiple tenants, the interviewer pivots to "how do you handle PII, tenant isolation, deletion requests, and data residency?" This axis is about *data handling and compliance* — where sensitive data flows, who can retrieve it, how long it lives, and how you delete it — distinct from the attack-defence framing of `06-security-and-safety-patterns.md`.

---

## TL;DR

Privacy and governance in AI systems is about controlling the *flow, scope, and lifetime* of sensitive data across the pipeline: detect and redact PII before it enters a prompt or a log, isolate each tenant's data so one customer can never retrieve another's, pin data to a region, encrypt it in transit and at rest, retain prompts/responses only as long as needed, and be able to *delete* a record on request — including its embeddings in the vector store. The recurring trade-off is utility and simplicity versus exposure: every place raw data lands (a log line, a shared index, a third-party model provider) is a place it can leak or violate a deletion request. **The one thing to remember: redact before you log, isolate before you retrieve, and make deletion a first-class operation — governance you bolt on afterwards is governance you cannot prove.**

---

## ELI5 — Explain It Like I'm 5

Imagine a shared office building where many companies keep filing cabinets in one big room. Before any document goes into a cabinet, a clerk at the door blacks out the personal details (names, phone numbers) with a marker, and stamps each folder with its company's colour. When someone asks for a file, the guard only hands back folders in *their* company's colour — never a competitor's — and writes down who asked for what in a logbook. If a company leaves, the clerk must find and shred every folder in their colour, not just the ones on top. The common mistake is thinking the marker and the colour-stamp are optional extras; in this building they are the whole reason many companies trust one room, and skipping them once means someone reads a file that was never theirs.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Trace where sensitive data (PII) flows through an AI pipeline and place redaction *before* both the model call and the log write.
- [ ] Choose a multi-tenant isolation mechanism (row-level security, per-tenant namespaces, separate tables/indexes) and enforce it at retrieval time.
- [ ] Design retention/TTL and right-to-erasure so a deletion request removes both source records *and* their embeddings, with an audit trail.
- [ ] Justify data-use boundaries (region pinning, zero-retention endpoints, not sending sensitive data to third parties) against their cost and complexity in an interview.

---

## Visual Overview

### PII redaction before the model and before logs

```
User input (may contain PII)
   │
   ▼
┌────────────────┐   detect + redact (regex/NER classifier)
│ PII scrubber   │──►  "email <EMAIL>, SSN <SSN>"  (reversible token map kept
└────────────────┘                                    server-side, never logged)
   │                    │
   │ redacted prompt    └──────────────► Observability / log store
   ▼                                        (stores REDACTED text only)
┌────────────────┐
│  LLM provider  │  ◄── zero-retention / region-pinned endpoint
└────────────────┘
   │  response
   ▼
 re-hydrate tokens (server-side) ──► user     [logs still see redacted form]
```

### Per-tenant isolation in one shared vector store

```
                 query from tenant B (user X)
                          │
                          ▼
              ┌───────────────────────┐
              │  retrieval service     │  attaches tenant_id = B
              │  (enforces the filter) │  + user ACL scope
              └───────────┬───────────┘
                          │  filter: namespace/metadata == B  AND doc_acl ⊇ X
                          ▼
      ┌──────────────── shared index ────────────────┐
      │  ns=A ┆ ns=B ┆ ns=C     (namespaces/partitions)│
      │        └───────► only ns=B rows are candidates │
      └───────────────────────────────────────────────┘
                          │
                          ▼
              only tenant B, user-X-permitted chunks
```

---

## The Core Problem

An AI pipeline moves data through many stages — ingestion, embedding, storage in a vector index, retrieval, prompt assembly, a model call (often a third-party API), a response, and logs/traces. Each stage is a potential exposure point: raw PII in a prompt is sent to an external provider; the same PII copied verbatim into a log line persists in your observability stack for weeks; a shared vector index without a tenant filter lets one customer's query surface another customer's chunks; a deleted source document leaves its embedding behind, so a "forgotten" record still answers queries. The interview question is rarely "is it compliant" — it is "show me where sensitive data goes, who can reach it, how long it lives, and how you erase it."

Two properties must be reasoned about separately, because different mechanisms fix them:

- **Confinement** — sensitive data must not reach places or parties it shouldn't (a third-party model with training retention, a shared log, another tenant's query). Fixed by redaction, isolation filters, and data-use boundaries.
- **Lifecycle** — data must not outlive its purpose and must be *deletable* on demand. Fixed by retention TTLs, and by deletion that cascades from source record → chunks → embeddings → caches.

A design that redacts prompts but logs them raw has solved confinement at the model and broken it at the log. A design that isolates tenants at query time but shares one approximate index can still leak recall/latency signal across tenants. Name both axes before proposing fixes.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **PII detection & redaction** | Classifier/regex/NER masks PII before prompt + before log; keep a server-side re-hydration map | Sensitive data never reaches provider or logs in the clear | False negatives leak; false positives break answers; NER latency | Any input that can contain personal data |
| **Row-level security (RLS)** | DB enforces per-row visibility by tenant/user predicate at the engine level | Isolation that holds even if app code has a bug | Policy design/testing; owner/BYPASSRLS pitfalls | Relational store shared across tenants/users |
| **Per-tenant vector namespaces / partitions** | Separate namespace, partition, or table per tenant in the vector store | One tenant's vectors cannot appear in another's search | More indexes/partitions to manage; cross-tenant queries harder | Multi-tenant RAG on a shared vector DB |
| **Data residency / region pinning** | Route storage (and sometimes inference) to a specific region | Meets residency/sovereignty requirements | Fewer models/features per region; possible cost uplift; latency | Regulated data that must stay in-region |
| **Retention TTL + zero-retention endpoints** | Cap how long prompts/responses persist; use provider zero-data-retention mode | Bounds exposure window; limits third-party storage | Loses history for debugging; ZDR disables some features | Sensitive workloads / short lawful basis |
| **Right-to-erasure (cascade delete + re-index)** | Deletion removes source, chunks, embeddings, caches; re-index if needed | Provable erasure for a subject/tenant | Must track every derived copy; re-index cost | Any system holding personal data of individuals |
| **Retrieval access control** | Filter candidates by the requesting user's permissions, not just tenant | User only retrieves what they're allowed to see | ACL sync with source system; per-query filter cost | Documents with per-user/per-group permissions |

**PII detection & redaction** — a detector (regex for structured PII like emails/cards, or an NER/classifier model for names/addresses) scans text and replaces spans with placeholders (`<EMAIL>`, `<SSN>`) before the text is used. The masked form is what goes to the model *and* to logs; a reversible token→value map is held server-side (never logged) so the response can be re-hydrated for the user. Mechanically it sits as middleware on the request path and again on the logging path — two enforcement points, not one. It appears as a pre-prompt scrubbing step plus a log-formatter filter. The risk is recall: a missed pattern (false negative) leaks, so structured detectors are paired with a model-based detector for free-text.

**Row-level security (RLS)** — the database itself refuses to return rows a policy predicate rejects, so isolation does not depend on every query carrying a correct `WHERE tenant_id = …`. In PostgreSQL you `ALTER TABLE … ENABLE ROW LEVEL SECURITY` and `CREATE POLICY … USING (tenant_id = current_setting('app.tenant'))`; with RLS enabled and no matching policy, the default is deny. It appears as policies attached to tables plus a per-request session variable the app sets. The classic trap: table owners and `BYPASSRLS` roles skip policies, so the application must connect as a non-owner role.

**Per-tenant vector namespaces / partitions** — instead of one flat vector index for everyone, each tenant gets its own namespace (managed vector DBs), list partition, or separate table (`pgvector`: `PARTITION BY LIST(tenant_id)` or separate tables). A query is scoped to the tenant's partition, so cross-tenant vectors are not even candidates. It appears as a `namespace`/`tenant_id` argument on the search call or a partitioned table. Beyond correctness, pgvector's docs note a shared *approximate* index lets one tenant's vectors affect another's recall and speed — isolation via partitioning avoids that.

**Data residency / region pinning** — customer content is stored (and, where supported, processed) in a chosen region so it never leaves a jurisdiction. Managed providers expose this as a per-project region setting and a region-specific API domain; note that regional *storage* does not always imply regional *processing*, and some regions require additional retention amendments. It appears as a region selector at project creation plus a regional endpoint host. The cost: a narrower model/feature set per region and, for newer models, a price uplift.

**Retention TTL + zero-retention endpoints** — you cap how long your own store keeps prompts/responses (a TTL/expiry job) and, at the provider, use zero-data-retention or modified-retention modes so the vendor does not persist customer content in abuse-monitoring logs. By default a provider may retain API logs for a bounded window (e.g. ~30 days) unless you are approved for zero retention. It appears as a TTL field on your log/history store and a data-controls setting on the provider account. The trade-off: ZDR can disable stateful features (server-side conversation storage, some tools) and shorten your own debugging history.

**Right-to-erasure (cascade delete + re-index)** — a deletion request must remove not just the source row but every *derived* copy: chunk records, their embeddings in the vector store, semantic-cache entries, and any provider-side stored objects (files, vector stores) that support deletion. In `pgvector` this is an ordinary `DELETE` of the embedding rows (with vacuum/reindex for approximate indexes); with a managed store it is a delete-by-id or delete-by-filter call. It appears as a deletion pipeline keyed on a subject/tenant id that fans out across all stores. The trap is forgetting a derived copy — a deleted document whose embedding survives still answers queries.

**Retrieval access control** — tenant isolation says "only tenant B's data"; access control narrows further to "only what *this user* in tenant B may see." Each chunk carries an ACL (owner, group, visibility) mirrored from the source system, and the retriever filters candidates by the requester's permissions *before* they reach the prompt. It appears as an ACL metadata filter on the vector query joined to the user's group memberships. The hard part is keeping ACLs in sync when source-system permissions change — a stale ACL over-shares.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Redaction strictness (detector set + threshold) | How aggressively PII is masked | Combine regex (structured) + NER (free-text); tune the NER threshold so recall on a labelled PII set beats a stated floor — a missed span is worse than an over-mask |
| Tenant-filter enforcement point | Where the tenant scope is applied (app code vs DB engine vs index namespace) | Enforce at the *engine/namespace* (RLS, per-tenant namespace), not only in app code — app-only filters fail open on a bug |
| Retention TTL (prompts/responses/logs) | How long your own store keeps content | Set to the shortest window that supports debugging + lawful basis; default long TTLs silently accumulate PII |
| Provider retention mode (default / ZDR / modified) | Whether the third party persists customer content | Use zero/modified retention for sensitive data; accept the disabled features rather than let PII sit in vendor logs |
| Region (storage / processing) | Jurisdiction where data lives and is processed | Pin to the required region; verify whether the region supports *processing*, not just storage, before promising residency |
| Deletion cascade scope | Which derived stores a delete touches | Enumerate every derived copy (chunks, embeddings, caches, provider files) and delete all in one keyed pipeline — partial deletion is non-compliance |

### Worked Example: Requirement → Decision

**Given:** A RAG assistant serves several enterprise customers (tenants) over documents that contain employee PII. Requirements: no customer can retrieve another's documents; PII must not land in logs or be used to train the model provider; and any customer must be able to demand deletion of a named employee's data, provably.

- **Step 1 — Identify the goal:** Confine data (isolate tenants, keep PII out of logs/provider training) *and* control lifecycle (erase an individual on request). Two axes, both in scope.
- **Step 2 — Define inputs:** Multi-tenant document corpus with per-document PII; user queries carrying a tenant id and user identity; a shared relational + vector store.
- **Step 3 — Define outputs:** Answers grounded only in the requesting tenant's (and permitted) documents; logs containing redacted text only; a deletion operation that removes an employee's records and embeddings everywhere.
- **Step 4 — Apply constraints:** Third-party model provider (must use a no-training / retention-limited mode); observability must not store raw PII (links to `08-observability-and-monitoring-patterns.md`); deletion must be auditable.
- **Step 5 — Select the approach:** (1) **RLS** on the relational store keyed on `tenant_id` *and* **per-tenant vector partitions** so isolation holds at the engine, not just in app code; (2) **PII redaction middleware** on both the prompt path and the log formatter, with a server-side re-hydration map; (3) a **zero/modified-retention** provider mode so prompts are not persisted or trained on; (4) a **cascade-delete pipeline** keyed on employee id that removes source rows, chunk rows, and embeddings, then re-indexes. Rationale vs alternatives: relying on app-code `WHERE` filters alone is one bug from cross-tenant leakage; redacting only at the prompt still leaves PII in logs; deleting only the source document leaves a live embedding that keeps answering. Each choice closes a specific leak the cheaper alternative leaves open.

---

## Implementation

```python
# Scenario: user-typed support messages contain emails/phone numbers. We must
# keep PII out of BOTH the third-party model call AND our own logs, while still
# returning a natural answer to the user. So we redact once, remember the map
# server-side, call the model on the masked text, then re-hydrate for the user.
import re

PATTERNS = {
    "EMAIL": re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+"),
    "PHONE": re.compile(r"\+?\d[\d\s().-]{7,}\d"),
}

def redact(text: str):
    token_map = {}                       # placeholder -> original (server-side only)
    def sub(kind, pat, s):
        def repl(m):
            ph = f"<{kind}_{len(token_map)}>"
            token_map[ph] = m.group(0)
            return ph
        return pat.sub(repl, s)
    for kind, pat in PATTERNS.items():
        text = sub(kind, pat, text)
    return text, token_map               # map is NEVER logged or sent to provider

def rehydrate(text: str, token_map: dict) -> str:
    for ph, original in token_map.items():
        text = text.replace(ph, original)
    return text

def handle(user_msg: str, call_model, log):
    masked, token_map = redact(user_msg)
    log.info("prompt=%s", masked)         # log stores REDACTED text only
    answer = call_model(masked)           # provider sees REDACTED text only
    return rehydrate(answer, token_map)   # user sees the real values back
```

```python
# Anti-pattern: logging the raw prompt "for debugging" and filtering tenants
# only in application code. The log now permanently holds PII, and a single
# missing WHERE clause lets one tenant read another's rows.
def handle_bad(user_msg, tenant_id, db, log):
    log.info("raw prompt: %s", user_msg)                     # PII persisted in logs!
    rows = db.execute("SELECT chunk FROM docs")              # forgot tenant filter -> leak
    return generate(user_msg, rows)

# Correct approach: redact before logging, and enforce tenant isolation in the
# DB engine with Row-Level Security so it cannot fail open on an app-code bug.
# --- one-time SQL setup (PostgreSQL) ---
#   ALTER TABLE docs ENABLE ROW LEVEL SECURITY;   -- default-deny once enabled
#   CREATE POLICY tenant_isolation ON docs
#       USING (tenant_id = current_setting('app.tenant')::int);
#   -- app connects as a NON-owner role (owners bypass RLS by default)
def handle_good(user_msg, tenant_id, db, log):
    masked, token_map = redact(user_msg)
    log.info("prompt=%s", masked)                            # redacted only
    db.execute("SET app.tenant = %s", (tenant_id,))          # engine enforces scope
    rows = db.execute("SELECT chunk FROM docs")              # RLS filters to this tenant
    return rehydrate(generate(masked, rows), token_map)
```

```python
# Scenario: an enterprise customer invokes right-to-erasure for one employee.
# A "forgotten" record must stop answering queries, so deletion has to cascade
# to the embeddings, not just the source row. (pgvector: embeddings are rows.)
def erase_subject(db, subject_id: int):
    # 1) delete derived vector rows first so nothing surfaces mid-delete
    db.execute("DELETE FROM doc_embeddings WHERE subject_id = %s", (subject_id,))
    # 2) delete source chunks/records
    db.execute("DELETE FROM docs WHERE subject_id = %s", (subject_id,))
    # 3) drop any semantic-cache entries keyed to this subject
    db.execute("DELETE FROM response_cache WHERE subject_id = %s", (subject_id,))
    # 4) reindex the approximate index so deleted tuples are reclaimed
    db.execute("REINDEX INDEX CONCURRENTLY doc_embeddings_hnsw_idx")
    audit_log(action="erasure", subject_id=subject_id)       # provable trail
```

---

## Common Pitfalls & Misconceptions

- **Redacting the prompt but logging it raw** — teams add redaction at the model call and consider the job done, forgetting the logging path is a *second* place raw PII lands; treat redaction as two enforcement points (before the model *and* before the log), driven by the same scrubber.
- **Tenant isolation only in application code** — a `WHERE tenant_id = ?` in every query feels sufficient until one query omits it and fails *open*, exposing all tenants; enforce isolation in the engine (RLS) or via per-tenant namespaces so a missing filter fails *closed*.
- **Deleting the source but not the embedding** — beginners equate "right to be forgotten" with removing the original document, but the vector store holds a derived copy that keeps answering queries; deletion must cascade to chunks, embeddings, and caches, then re-index.
- **Assuming a provider never retains or trains on your data** — the default for many APIs is no-training but *bounded retention* (e.g. abuse logs kept for a window), which is not the same as zero; for sensitive data, explicitly enable zero/modified-retention rather than assuming the default is enough.

---

## Key Definitions

| Term | Definition |
|---|---|
| PII redaction | Detecting and masking personally identifiable information (emails, IDs, names) before text is sent to a model or written to a log, optionally with a server-side map to re-hydrate the response. |
| Row-Level Security (RLS) | A database feature where per-row visibility is enforced by a policy predicate at the engine level; once enabled the default is deny, so no policy means no rows. |
| Tenant isolation | Guaranteeing one customer (tenant) can never read or retrieve another tenant's data, ideally enforced below the application (RLS, per-tenant namespaces/partitions). |
| Data residency / region pinning | Configuring where customer content is stored (and sometimes processed) so it remains within a required jurisdiction. |
| Zero data retention (ZDR) | A provider mode that excludes customer content from the vendor's retained logs; may also disable stateful features that require storage. |
| Right to erasure | The requirement to delete an individual's or tenant's data on request, cascading to all derived copies (chunks, embeddings, caches). |
| Retrieval access control | Filtering retrieval candidates by the requesting user's permissions (ACLs), not just their tenant, before they enter the prompt. |

---

## Summary / Quick Recall

- Reason on two axes: **confinement** (redact, isolate, restrict data-use) and **lifecycle** (TTL, deletable-by-design).
- Redact PII at *two* points — before the model call and before the log write — using the same scrubber; keep the re-hydration map server-side only.
- Enforce tenant isolation in the engine (RLS) or via per-tenant namespaces/partitions, not app-code filters that fail open.
- Right-to-erasure must cascade: source → chunks → embeddings → caches, then re-index; a surviving embedding still answers.
- For third parties, "no training" ≠ "no retention" — enable zero/modified retention and pin the region when data is sensitive or regulated.
- Retrieval access control narrows below tenant to what *this user* may see; keep ACLs synced with the source system.
- This axis is data handling & compliance; the adversarial framing (prompt injection, jailbreaks, data exfiltration attacks) lives in `06-security-and-safety-patterns.md`.

---

## Self-Check Questions

1. Name the two distinct points in an AI pipeline where PII redaction must be applied, and why one is not enough.

   <details><summary>Answer</summary>

   Redaction must happen (a) before the prompt is sent to the model and (b) before the text is written to logs/traces. Applying it only at the model call is insufficient because the logging path is a separate place raw PII lands and persists for the log's retention window — a leak the model-side redaction never touches. Answering "just before the model" is incomplete for exactly this reason.

   </details>

2. A multi-tenant RAG system filters retrieval by adding `WHERE tenant_id = ?` in application code. During a review you're asked to harden isolation. What do you change and why?

   <details><summary>Answer</summary>

   Move enforcement into the engine: enable PostgreSQL Row-Level Security with a policy like `USING (tenant_id = current_setting('app.tenant'))` (and/or use per-tenant vector namespaces/partitions). The app-code filter fails *open* — one query that forgets the clause exposes every tenant — whereas RLS defaults to deny and holds even if a query omits the filter. The tempting wrong answer is "add the filter to more queries," which still relies on every future query being correct.

   </details>

3. A customer invokes right-to-erasure for one employee. Engineering deletes the employee's source documents, but the assistant still surfaces that employee's information in answers. What was missed?

   <details><summary>Answer</summary>

   The *derived* copies were not deleted. The embeddings for those documents (and any semantic-cache entries) still live in the vector store and remain retrievable, so the assistant keeps grounding answers on them. Erasure must cascade to chunk rows, embeddings, and caches — then re-index the approximate index — not just remove the original document. Deleting only the source is the classic incomplete fix.

   </details>

4. **Which TWO** of the following are properties of *confinement* (keeping sensitive data out of places/parties it shouldn't reach) rather than *lifecycle* (bounding how long data lives / deleting it)?
   - A. Per-tenant vector namespaces
   - B. Retention TTL on stored prompts
   - C. Sending only redacted text to a third-party model
   - D. Cascade deletion of embeddings on an erasure request
   - E. A nightly job that expires logs older than 30 days

   <details><summary>Answer</summary>

   **A and C.** Per-tenant namespaces stop one tenant's data reaching another's query, and redaction stops PII reaching the third-party provider — both are confinement (controlling *where/to whom* data flows). B, D, and E are lifecycle: TTLs and log expiry bound *how long* data lives, and cascade deletion is erasure. D is the most tempting wrong pick because deletion clearly protects privacy, but it operates on data lifetime, not on confining a live flow.

   </details>

5. For a regulated workload you must promise that customer content stays in-region *and* is never retained by the model vendor. A teammate says "we picked the EU region, so we're covered." Why is that not automatically sufficient?

   <details><summary>Answer</summary>

   Two gaps. First, regional *storage* does not always imply regional *processing* — some regions store content in-region but may process it elsewhere, so you must verify the region supports processing, not just storage, before promising residency. Second, region choice is orthogonal to retention: the vendor's default may still retain customer content in abuse-monitoring logs for a bounded window, so you also need an explicit zero/modified-retention mode. "We picked the EU region" addresses residency-storage only and leaves both processing location and vendor retention unverified.

   </details>

---

## Further Reading

- [PostgreSQL — Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) — *verified 2026-07-29* — how `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY … USING (…)` enforce per-row tenant/user isolation at the engine, the default-deny behaviour, and the owner/`BYPASSRLS` bypass pitfalls.
- [pgvector — README (multitenancy, filtering, delete)](https://github.com/pgvector/pgvector) — *verified 2026-07-29* — tenant isolation via `PARTITION BY LIST` or separate tables, metadata/`WHERE` filtering for access control, `DELETE`/`REINDEX` for erasing embeddings, and why a shared approximate index leaks recall across tenants.
- [OpenAI — Data controls in the OpenAI platform](https://platform.openai.com/docs/guides/your-data) — *verified 2026-07-29* — default no-training, bounded abuse-monitoring retention (~30 days), Zero/Modified Data Retention modes, per-endpoint storage behaviour, and region/data-residency controls.
- [Anthropic — Is my data used for model training? (commercial)](https://privacy.anthropic.com/en/articles/7996868-is-my-data-used-for-model-training) — *verified 2026-07-29* — commercial products (Claude for Work, Anthropic API) do not use inputs/outputs to train models by default, and how feedback submission is the explicit opt-in exception.
