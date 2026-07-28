# APIs, Databases, and Data Platforms for AI — Interview Prep

**Section:** 01 Classical Systems Refresher → APIs, Databases, and Data Platforms for AI | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| In FastAPI, what is the practical difference between an `async def` and a `def` path operation for an LLM-serving route, and when does a blocking call become catastrophic? | An `async def` route runs directly on the single-threaded event loop, so one `await` (e.g. an async HTTP call to the model) lets the loop serve thousands of other requests; a `def` route is run in an external threadpool, so a blocking call inside it does not freeze the loop. A blocking call inside `async def` (a synchronous SDK, a blocking DB driver) freezes the loop for the whole multi-second generation, collapsing throughput to ~one request at a time per worker. For I/O-bound model calls, `async def` + an async client is the high-concurrency default. | "Just mark everything `async` and it's automatically concurrent." Async only helps at `await` points; it does not make the model faster and does not save you from a blocking call. |
| Why does an inference endpoint use `POST` with Pydantic validation rather than `GET`, and what does Pydantic buy you at the edge? | `POST` because the body is large, non-idempotent (the model may return different text per call), and must not be cached by intermediaries; versioned paths (`/v1/`) let schemas evolve without breaking clients. Pydantic parses, coerces, validates, and returns `422` *before the handler runs*, so the model is never invoked on invalid/oversized input. `Field(gt=0, le=2048)` on `max_tokens` is a concrete cost/latency guardrail enforced at the edge. | "Validate the input inside the handler with `if` checks." That runs after dispatch, duplicates what Pydantic already does, and misses the point that `422` fires before the model is touched. |
| Compare relational, NoSQL, and vector stores by the *shape of the query* each is built for. | Relational (Postgres): exact-key/range lookups, joins, ACID transactions — the system of record. NoSQL: fetch/write one document or value by known key at horizontal scale, trading joins/strong consistency for scale and flexible schemas. Vector: approximate-nearest-neighbour similarity over embeddings — "what's most similar," not exact match. The real design question is how they *compose*, not "which one." | "Pick the store by the buzzword" — e.g. reaching for a NoSQL doc store because it "scales" without naming the access pattern, or treating "AI project" as automatically meaning "vector DB." |
| How does an HNSW index make similarity search sub-linear, and which parameter is the query-time recall/latency dial? | HNSW is a multi-layer proximity graph: upper layers have few nodes with long-range links for coarse navigation, lower layers are dense with local links; a query greedily hops toward the target from the top down, keeping a candidate list of size `ef_search`. It trades a small chance of missing the true nearest neighbour for sub-linear query time. `hnsw.ef_search` (default 40) is the direct latency-vs-recall dial at query time — raise it (100–400) for higher recall, and it must be ≥ `LIMIT`. | "HNSW is just a faster exact search." It is *approximate* — you accept a recall trade-off for speed; claiming exactness misunderstands the whole ANN premise. |
| What does idempotency mean for a data-pipeline task, and how does it map to RAG re-indexing? | Idempotency means re-running (retry or backfill) converges to the same end state with no duplicates or corruption. Airflow's guidance: use `UPSERT` not `INSERT`, read a specific partition keyed by `data_interval_start` (never `now()`). In RAG this maps to a hash-based record manager + upsert keyed by a deterministic chunk/source id, so a re-run of unchanged docs is a no-op (`num_skipped` high). | "Just add more retries." Retries without idempotency multiply damage — each retry re-runs the non-idempotent side effect (duplicate rows, double-charged embeddings). |
| What is post-filter starvation in a filtered vector query, and what causes it? | With an approximate index the metadata filter (`WHERE tenant_id = 42`) is applied *after* the ANN scan of the `ef_search` candidate list, so if most candidates belong to other tenants, a `LIMIT 10` can return far fewer than 10 rows even when the tenant has thousands of documents. Fixes: a B-tree on the filter column (enables fast exact NN for selective filters), partial/partitioned indexes per tenant, or `hnsw.iterative_scan = strict_order`/`relaxed_order`. | "Just raise the `LIMIT`." The candidate list, not the `LIMIT`, is the bottleneck — raising `LIMIT` alone does not fix starvation. |
| When does "the Postgres you already have + pgvector" beat a dedicated vector database, and what are the tipping points? | For up to tens of millions of vectors that also need joins/filters/ACID, pgvector co-locates embeddings with relational metadata: an insert of a document and its embedding is one transaction (no cross-store drift), and a single `WHERE` clause enforces tenant isolation. Tipping points to leave Postgres: billions of vectors, need for managed sharding, or specialised ANN features. | "Always use a dedicated vector DB for anything AI." Adds an operational system and a cross-store consistency problem for a workload Postgres handles — and splits tenant isolation across two systems that can drift. |

## Applied / Scenario Questions

**Q:** You must expose a streaming `/v1/chat/completions` endpoint for a web chat UI. The model sits behind a remote provider that averages ~800 ms to first token and ~6 s to full completion. The UI must feel responsive, one misbehaving client must not degrade others, and a client who closes the tab should stop costing you tokens.

**Strong answer framework:**
- Use an `async def` route so one worker holds many concurrent connections while each `await`s the I/O-bound provider; pair it with an async HTTP client. A synchronous SDK call inside `async def` would freeze the loop for the full 6 s per request.
- Return an `EventSourceResponse` (SSE) that `yield`s tokens as they arrive, ending with a `data: [DONE]` sentinel (sent as `raw_data` so it isn't JSON-quoted). This optimises *first-token* latency — what the user perceives — instead of buffering the full answer as JSON.
- Load the model/HTTP client once in a `lifespan` context manager and inject it with `Depends`, so it's shared (pooled) across requests, not rebuilt per call; override it in tests for deterministic, network-free runs.
- Bound input with Pydantic (`max_tokens: Field(gt=0, le=2048)`), add a rate-limit middleware that returns `429` with `Retry-After` *before* the request reaches the GPU, and set separate connect/read timeouts with capped-with-backoff retries on transient errors (`429`/`5xx`/timeout) only.
- Handle `http.disconnect` so the async generator is cancelled at its next `await`, stopping the upstream generation.
- **Tradeoff to name:** streaming trades a slightly more complex client/proxy setup (SSE keep-alives, GZip must skip `text/event-stream`) for dramatically better *perceived latency*; capped retries trade a little tail-latency for avoiding retry storms that amplify load during an upstream incident; the rate limit trades some client convenience for protecting shared GPU throughput and safety (blocking oversized/injection-via-size payloads at the edge).

**Q:** Design the storage for a multi-tenant customer-support RAG assistant: ~2M article chunks, retrieval must always be scoped to the asking tenant, biased toward recent articles, p95 < 150 ms, and strict tenant isolation is a hard security requirement. The app already runs on Postgres.

**Strong answer framework:**
- Keep vectors in Postgres via `pgvector`: co-locating embeddings with `tenant_id`/`published_at` metadata means one query both filters and ranks, and a document + its embedding commit in one ACID transaction (no cross-store drift that could leak or lose data).
- Use an **HNSW** index on the operator class matching the (normalised) embedding model — cosine (`<=>`) or inner product (`<#>`), never L2 by default — because HNSW gives the best speed/recall trade-off for latency-sensitive reads; IVFFlat would build faster but miss the p95 recall target.
- Scope every query with `WHERE tenant_id = $1` backed by a **B-tree on `tenant_id`**, and enable `hnsw.iterative_scan` so the hard tenant filter never starves results to fewer than *k*.
- Add a recency nudge in ranking, and tune `hnsw.ef_search` (e.g. 100) as the recall/latency dial, keeping it ≥ `LIMIT`.
- **Tradeoff to name:** staying in pgvector trades theoretical billion-scale headroom for operational simplicity and *enforceable* tenant isolation in a single `WHERE` clause (a security win — two systems could drift and leak cross-tenant results); raising `ef_search`/`iterative_scan` trades latency for recall; choosing HNSW over IVFFlat trades build time/memory for query quality. Revisit only when a measured limit (billions of vectors, sharding) forces the split.

## System Design / Architecture Questions

**Q:** Design the API + storage + ingestion for a document-QA RAG service. Users upload documents and ask questions; answers must cite sources, stay fresh within a day of edits, and the system must be multi-tenant and cost-aware.

**Approach:**

1. **Clarify requirements (scale, latency, freshness, sensitivity).**
   - Scale: how many tenants, documents, and chunks (thousands? tens of millions?) — this decides pgvector vs a dedicated vector DB.
   - Latency budget: interactive chat implies a first-token target (streaming) and a retrieval p95 (e.g. < 150 ms).
   - Freshness SLA: "within 24 h of edits" points to nightly batch, not streaming, ingestion.
   - Data sensitivity: multi-tenant means hard tenant isolation on every retrieval and idempotent, auditable ingestion.
   - Hallucination tolerance: citations required, so provenance metadata (source id, character offset) must survive chunking.

2. **Propose high-level architecture.**
   - **Serving API (FastAPI/ASGI):** `POST /v1/query` returning an SSE `EventSourceResponse` that streams tokens with a `[DONE]` sentinel; Pydantic bounds `max_tokens`/`top_k`; a shared retrieval client + LLM client loaded once via `lifespan` and injected with `Depends`; rate-limit + request-id middleware; `422`/`429`/`504` error contract; readiness probe flips true only after clients are loaded.
   - **Storage:** Postgres + pgvector as the system of record and vector index — `doc_chunks(id, tenant_id, source, content, published_at, embedding vector(n))` with an HNSW index on the model's operator class, a B-tree on `tenant_id`, and `hnsw.iterative_scan` for the tenant filter. Raw uploaded files land cheaply in object storage (lake), optionally under a lakehouse format (Delta/Iceberg) for ACID snapshots and reproducible eval sets.
   - **Retrieval:** filtered top-k `ORDER BY embedding <=> $q WHERE tenant_id = $t LIMIT k`, with hybrid search (full-text `tsvector`/BM25 fused with vector via RRF) so exact identifiers (SKUs, error codes) aren't lost by semantic search alone.
   - **Ingestion pipeline (Airflow DAG, `@daily`, `retries=2`):** extract (read the partition keyed by `data_interval_start`, not `now()`) → chunk (`RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200, add_start_index=True)` for provenance) → embed → upsert via LangChain `index(..., cleanup="incremental", source_id_key="source")` with a hash `key_encoder`, so only changed docs are re-embedded and stale chunks pruned. Backfill (`--reprocess-behavior=completed`, bounded `--max-active-runs` to respect embedding rate limits) re-embeds a date range after a chunking/model change (`force_update=True` for a deliberate full pass).

3. **Justify choices and name tradeoffs explicitly.**
   - **Cost:** incremental hash-based upsert re-embeds only the ~changed docs instead of the whole corpus nightly; embeddings are the pipeline's cost/latency hotspot, so batch beats always-on streaming for a 24 h SLA.
   - **Latency:** SSE streaming optimises perceived (first-token) latency; HNSW + `ef_search` tuning hits retrieval p95; a synchronous route or full-JSON response would fail the latency budget.
   - **Complexity/consistency:** pgvector avoids a second datastore and cross-store drift until scale forces a dedicated vector DB; a lakehouse adds a transaction log so concurrent ingestion jobs never expose half-written data.
   - **Security/safety:** tenant isolation is one `WHERE` clause on co-located data (not a cross-system sync); Pydantic bounds + rate limits block oversized/abusive payloads at the edge; citations via `add_start_index` provenance reduce ungrounded answers.

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:
- **ASGI / event loop** — when explaining why one `async` worker holds thousands of concurrent I/O-bound model calls, and why a blocking call on the loop is fatal.
- **Server-Sent Events (SSE) / `EventSourceResponse` / `[DONE]` sentinel** — when designing token streaming and discussing first-token vs total latency and client-disconnect cancellation.
- **Dependency injection (`Depends`) + `lifespan`** — when explaining how a heavy model/HTTP client is loaded once and shared, and how you override it for testing.
- **Readiness vs liveness probe** — when discussing safe rollouts so traffic doesn't arrive before the model is loaded.
- **ACID / write-ahead log** — when justifying Postgres as the system of record and why co-located embeddings commit or roll back atomically.
- **HNSW / `ef_search` / operator class** — when tuning the recall-vs-latency dial and matching the distance metric to how the embedding model was trained.
- **IVFFlat / `lists` / `probes`** — when a tight build window and relaxed recall make a cheaper-to-build index the right call.
- **Hybrid search / BM25 / Reciprocal Rank Fusion (RRF)** — when retrieval must not fail on exact identifiers that semantic search alone misses.
- **Post-filter starvation / `hnsw.iterative_scan`** — when discussing why a filtered ANN query under-returns and how to fix it.
- **pgvector** — when arguing for co-locating vectors with relational metadata until a measured limit forces a dedicated vector DB.
- **ETL vs ELT** — when explaining where transformation runs and why cloud warehouse compute made ELT the default.
- **Idempotent upsert / backfill / `data_interval_start` / logical date** — when making a pipeline safe to retry and reprocess over past intervals.
- **Incremental re-indexing / record manager / content hash / `cleanup="incremental"`** — when explaining how RAG ingestion re-embeds only changed content.
- **Lakehouse / Delta / Iceberg / MERGE / time travel** — when raw object storage needs ACID, upserts, and reproducible snapshots.

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:
- **"Just use a vector DB for everything."** — Ignores that vectors can't enforce foreign keys, transactions, or exact lookups, and that pgvector often wins until billion-scale; signals buzzword-driven design.
- **"Make the endpoint sync, it's simpler."** — For an I/O-bound model call this caps per-worker throughput at ~one request at a time (if blocking inside `async`) and abandons the concurrency ASGI gives you.
- **"Re-embed everything on each run to be safe."** — Confuses correctness with a blind full rebuild; it's not idempotent w.r.t. cost, blows embedding rate limits, and can duplicate on retry. Idempotency is hash-diff + upsert.
- **"Just add retries until it works."** — Retries without idempotency multiply damage; you must fix the side effect (UPSERT, specific partition) first.
- **"Return the whole answer as JSON, streaming is over-engineering."** — Fails the first-token latency budget users actually feel and wastes upstream tokens on disconnected clients.
- **"Raise the `LIMIT` to fix the vector query returning too few rows."** — Misdiagnoses post-filter starvation; the `ef_search` candidate list is the bottleneck.
- **"Use L2 distance, it's the standard."** — Mismatches models trained for cosine/inner product and silently degrades ranking quality.
- **"Streaming is always better because it's fresher."** — Ignores that streaming pays always-on compute and exactly-once complexity; a batch DAG meets most RAG freshness SLAs cheaper.
- **"Just dump the files in S3, it's our database."** — A raw lake has no transaction layer; concurrent jobs see inconsistent data — that's what a lakehouse format fixes.

## STAR Answer Frame

**Situation:** A production customer-support RAG assistant re-embedded its entire ~50k-article help-centre corpus every night. Nightly runs took hours, regularly tripped the embedding provider's rate limits (`429`s), and a mid-run failure occasionally left duplicate chunks in the vector store — degrading answer quality and cost.

**Task:** I owned making ingestion cost-efficient, idempotent, and reliably fresh within the 24 h SLA, without moving off the existing Postgres/pgvector stack.

**Action:** I replaced the delete-then-`add_documents` full rebuild with a nightly Airflow DAG (`schedule="@daily"`, `default_args={"retries": 2}`) whose extract task read the partition keyed by `data_interval_start` (never `now()`), then chunked with `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200, add_start_index=True)` and loaded via LangChain `index(..., cleanup="incremental", source_id_key="source")` backed by a `SQLRecordManager` with a `sha256` `key_encoder`. Unchanged docs hashed identically and were skipped; only edited articles were re-embedded and their stale chunks pruned; upsert semantics made retries and backfills converge to one correct chunk per source. I bounded `--max-active-runs` on backfills to stay within the embedding API's rate limits and used `force_update=True` only for the one deliberate full pass when we switched embedding models.

**Result:** Nightly embedding calls dropped from ~50k to the ~300 genuinely changed articles (>99% reduction), the run finished in minutes instead of hours, `429` rate-limit failures went to zero, and re-running the pipeline immediately produced an all-`num_skipped` result — a verifiable proof of idempotency — with no more duplicate chunks in retrieval.

## Red Flags Interviewers Watch For

- **Putting a blocking/synchronous model or DB call inside an `async def` route** and not recognising it freezes the event loop — the single most common FastAPI-for-AI mistake.
- **No input bounds at the API edge** — omitting Pydantic `Field` limits on `max_tokens`/payload size, leaving cost, latency, and injection-via-oversized-input unguarded.
- **Buffering a streamed answer as one JSON blob**, or forgetting that GZip/compression must skip `text/event-stream`, or omitting the `[DONE]` sentinel and disconnect handling.
- **Rebuilding the model client per request** instead of loading it once in `lifespan` and injecting via `Depends`; confusing liveness with readiness on rollout.
- **Defaulting to a dedicated vector DB "because AI"** without sizing the workload, or the inverse — insisting Postgres scales to billions of vectors with sharding.
- **Choosing the wrong distance metric / operator class** for the embedding model, or expecting a query to use the index without `ORDER BY <op> ... LIMIT`.
- **Not anticipating post-filter starvation** in multi-tenant filtered vector search, or "fixing" it by raising `LIMIT`.
- **Using `INSERT` or delete-then-add in a pipeline task** and calling `datetime.now()` inside transform logic — non-idempotent, unsafe to retry or backfill.
- **Proposing streaming ingestion for a 24 h freshness SLA**, paying always-on compute and exactly-once complexity for no benefit.
- **Treating a raw S3 data lake as if it had ACID guarantees**, or proposing a full nightly re-embed instead of hash-based incremental re-indexing.
