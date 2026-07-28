# Relational vs NoSQL vs Vector Storage Trade-offs

**Section:** 01 Classical Systems Refresher → APIs, Databases & Data Platforms for AI | **Est. time:** 2 hrs | **Interview relevance:** High — every RAG/agentic design question forces a storage choice, and "just use Postgres" vs "dedicated vector DB" is a signature trade-off interviewers probe.

---

## TL;DR

Relational databases (Postgres) give you ACID transactions, joins, and strong consistency for structured data; NoSQL stores trade those guarantees for horizontal scale and flexible schemas on specific access patterns (documents, key-value, wide-column); vector databases specialise in *approximate nearest-neighbour* similarity search over embeddings. For an AI system the real decision is rarely "which one" but "how do these compose" — embeddings, their source rows, and the metadata you filter on almost always need to live together. The `pgvector` extension lets Postgres do vector search inline with your relational data, which beats a separate vector DB until scale or specialised features force the split. **The one thing to remember: pick the store by the *shape of the query* (transactional lookup, key fetch, or similarity search), not by the buzzword — and default to "the Postgres you already have" until a measured limit pushes you off it.**

---

## ELI5 — Explain It Like I'm 5

Imagine three ways to organise a giant library. The first is a card catalogue with strict rules: every book has an exact ID, and you can trace which reader borrowed which book with a paper trail that is never wrong — that is a relational database. The second is a wall of labelled bins where you toss whole toy boxes in and grab them fast by their label, but nobody checks that every box has the same contents — that is NoSQL. The third is a magic librarian who, when you describe a book by *vibe* ("something cosy about space and friendship"), instantly hands you the ten closest matches even though you never said a title — that is a vector database. The common mistake is thinking the magic librarian replaces the card catalogue; really you want the librarian to *find* candidates and the catalogue to tell you the exact, trustworthy details about each one.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Compare relational, NoSQL, and vector stores by query shape, consistency model, and scaling behaviour, and justify a choice under stated constraints.
- [ ] Explain how HNSW and IVF approximate-nearest-neighbour indexes work and set their tuning parameters from a recall/latency target.
- [ ] Design a `pgvector` schema that co-locates embeddings with relational metadata and supports fast filtered similarity search.
- [ ] Diagnose why a vector query returns too few, slow, or wrong results (missing index, wrong distance metric, post-filter starvation).
- [ ] Decide when "use the Postgres you already have" beats a dedicated vector database, and articulate the tipping points.

---

## Visual Overview

### Which Store Do I Pick? (decision tree on query shape)

```
What is the dominant query?
├── "Give me THE row(s) matching exact keys / ranges, with joins + transactions"
│      └──► Relational (Postgres): ACID, foreign keys, SQL
│
├── "Fetch/write one document or value by a known key, at huge scale"
│      ├── whole JSON document by id ──► Document store (MongoDB / DynamoDB doc)
│      └── single value by key, µs reads ──► Key-value store (Redis / DynamoDB)
│
└── "Find the items most SIMILAR to this embedding" (semantic search / RAG)
       ├── vectors also need joins + filters + ACID, < ~10-50M rows
       │      └──► Postgres + pgvector  (keep it in one DB)
       └── billions of vectors, or need managed sharding / specialised ANN
              └──► Dedicated vector DB (Pinecone / Milvus / Weaviate / Qdrant)
```

### Three Storage Models Side by Side

```
 RELATIONAL              NoSQL (document/KV)        VECTOR
 ┌───────────────┐       ┌───────────────┐          ┌────────────────────┐
 │ id │ name │... │       │ key ─► { blob }│          │ id │ embedding[768] │
 ├───────────────┤       │ key ─► { blob }│          │ id │ embedding[768] │
 │ joins, FKs    │       │ no joins       │          │ + metadata columns  │
 │ ACID txns     │       │ eventual consist│          │ ANN index (HNSW)    │
 │ scale ↑ (vert)│       │ scale → (horiz) │          │ query = "closest k" │
 └───────────────┘       └───────────────┘          └────────────────────┘
   exact match &            fast key access,           similarity ranking,
   relationships            flexible schema            NOT exact match
```

### HNSW ANN Index (why approximate search is fast)

```
Query vector q ──► enter at TOP layer (few, long-range links)
   Layer 2:   ●─────────────────●          greedily hop toward q
                \               /
   Layer 1:      ●───●─────●───●            descend, denser links
                  \   \   /   /
   Layer 0:   ●─●─●─●─●─●─●─●─●─●  (all points, local links) ──► k nearest
              └── ef_search controls how wide the candidate list is here ──┘

 Skip-list-of-graphs: start coarse at the top, refine downward.
 Trades a small chance of missing the true nearest neighbour for
 sub-linear query time instead of scanning every vector.
```

---

## Key Concepts

### Relational Databases & ACID

**What it is.** A store of tables with typed columns and enforced relationships, queried with SQL, whose defining promise is the ACID transaction (Atomicity, Consistency, Isolation, Durability).

**How it works under the hood.** Statements are wrapped in a transaction (`BEGIN … COMMIT`); intermediate states are invisible to other sessions until commit, and every committed change is written to a durable write-ahead log before the commit is acknowledged, so a crash cannot lose it (PostgreSQL docs, §3.4). If any step fails, `ROLLBACK` discards the whole unit — the "all-or-nothing" property. Concurrency is managed so a transaction never sees another's half-finished work.

**Where it appears in real systems.** In an AI product this is the system of record: users, tenants, documents, permissions, billing, audit logs. In Postgres you get B-tree indexes for equality/range lookups and GIN indexes for JSONB and full-text — and, via `pgvector`, embeddings in the same transactional database so a document row and its vector commit or roll back together.

### NoSQL Families (document, key-value, wide-column)

**What it is.** A group of non-relational stores that relax joins and/or strong consistency to buy horizontal scale and schema flexibility, each specialised to one access pattern.

**How it works under the hood.** *Key-value* stores (Redis, DynamoDB) hash a key to a partition and return an opaque value in near-constant time — no query planner, no joins. *Document* stores (MongoDB) keep self-contained JSON-like documents so all data for one entity is co-located and fetched in one read, avoiding joins by denormalising. *Wide-column* stores (Cassandra) organise data by partition + clustering key for massive write throughput. Many use eventual consistency: a write propagates to replicas over time, so a read just after a write may return stale data.

**Where it appears in real systems.** Session/token caches and rate-limit counters (key-value), chat/session histories or agent state blobs (document), high-volume event/telemetry ingestion (wide-column). In AI systems NoSQL is common for *serving* precomputed artefacts fast, not for the relationships or transactions the relational store owns.

### Vector Databases & Similarity Search

**What it is.** A store optimised to find the *k* items whose embedding vectors are closest to a query vector under a distance metric (cosine, inner product, or L2) — semantic search rather than exact match.

**How it works under the hood.** Text/images are turned into fixed-length float vectors by an embedding model; "meaning similarity" becomes geometric closeness. Exact search compares the query to *every* stored vector (perfect recall, linear cost). To go faster, an approximate-nearest-neighbour (ANN) index pre-organises vectors so a query only touches a small subset, trading a little recall for large speed-ups. The chosen distance metric **must match how the embedding model was trained** — e.g. OpenAI embeddings are normalised, so inner product and cosine are equivalent and fastest.

**Where it appears in real systems.** The retrieval step of RAG, semantic dedup, recommendation, and agent memory. In `pgvector` it is a `vector(n)` column plus distance operators: `<->` (L2), `<#>` (negative inner product), `<=>` (cosine), queried with `ORDER BY embedding <=> $1 LIMIT k` (pgvector docs).

### ANN Indexes — HNSW vs IVFFlat

**What it is.** The two index structures `pgvector` (and most vector DBs) offer to make similarity search sub-linear: HNSW (a layered proximity graph) and IVFFlat (inverted lists via k-means clustering).

**How it works under the hood.** *HNSW* builds a multi-layer graph: upper layers have few nodes with long-range links for coarse navigation, lower layers are dense with local links; a query greedily hops toward the target from the top down, keeping a candidate list of size `ef_search`. It has the best speed–recall trade-off but slower builds and more memory, and needs no training step. *IVFFlat* runs k-means to split vectors into `lists` clusters; a query only scans the `probes` clusters nearest to it. It builds faster and uses less memory but has lower query quality, and it **must be built after the table holds representative data** or recall collapses (pgvector docs).

**Where it appears in real systems.** `CREATE INDEX … USING hnsw (embedding vector_cosine_ops)` or `USING ivfflat (…) WITH (lists = N)`. Rule of thumb from the docs: HNSW for latency-sensitive production reads; IVFFlat when build speed/memory matters and you can tolerate lower recall. Note an index exists *per distance operator class*, so you index for the metric you actually query with.

### Hybrid Search (BM25 / full-text + vector)

**What it is.** Combining lexical keyword search (BM25-style term matching) with semantic vector search so exact terms *and* meaning both contribute to ranking.

**How it works under the hood.** Vector search misses rare exact tokens (product SKUs, error codes, names) because they may sit far apart in embedding space; keyword search misses paraphrases. Hybrid runs both retrievers and fuses their ranked lists — most commonly with Reciprocal Rank Fusion (RRF), which scores each document by the sum of `1/(k + rank)` across lists, so agreement near the top of either list wins. In Postgres the lexical side uses full-text search: documents become `tsvector`s, queries become `tsquery`s, matched with `@@` and ranked with `ts_rank_cd` (PostgreSQL FTS docs).

**Where it appears in real systems.** Production RAG retrieval that must not fail on exact identifiers. pgvector's docs show combining `ORDER BY ts_rank_cd(...)` results with vector results via an RRF helper. Dedicated vector DBs (Weaviate, Qdrant, Milvus) expose hybrid search as a first-class query mode.

### Metadata Filtering (pre-filter vs post-filter)

**What it is.** Restricting a similarity search to rows matching structured predicates — `tenant_id = 42`, `status = 'published'`, `created_at > now() - interval '30 days'`.

**How it works under the hood.** With an **approximate** index the filter is applied *after* the ANN scan, so if a condition matches few rows you may get far fewer than *k* results — the classic "post-filter starvation". Fixes: create a plain B-tree on the filter column so a selective filter uses fast exact NN instead; use partial indexes (`… WHERE category_id = 123`) or `LIST` partitioning for high-cardinality tenants; or enable iterative index scans (`SET hnsw.iterative_scan = strict_order`) so pgvector automatically scans more of the graph until it finds enough matches (pgvector docs).

**Where it appears in real systems.** Multi-tenant RAG where every query must be scoped to one customer's documents — the difference between correct isolation and a security leak. It appears as the `WHERE` clause alongside `ORDER BY embedding <=> $1 LIMIT k`, plus the supporting B-tree/partial/partition strategy.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `m` (HNSW) | Max connections per node per layer (default 16) | Leave at 16; raise toward 32–64 only for high-dimensional data needing higher recall — costs memory and build time. |
| `ef_construction` (HNSW) | Candidate-list size while building the graph (default 64) | Raise (e.g. 128–200) when recall is low; higher = better recall but slower builds/inserts. Set once at index creation. |
| `hnsw.ef_search` (HNSW) | Candidate-list size at query time (default 40) | Increase (100–400) to raise recall/results per query; the direct latency-vs-recall dial. Must be ≥ `LIMIT`. |
| `lists` (IVFFlat) | Number of k-means clusters | Start at `rows/1000` up to 1M rows, `sqrt(rows)` beyond. Too few = slow scans; too many with little data = poor recall. |
| `ivfflat.probes` (IVFFlat) | Clusters scanned per query (default 1) | Start at `sqrt(lists)`; raise for recall, lower for speed. `probes = lists` gives exact search (index unused). |
| `hnsw.iterative_scan` | Auto-scan more of the index when filters starve results | Set to `strict_order` (exact order) or `relaxed_order` (better recall) when filtered queries return < k rows. |
| distance operator class | Which metric the index serves (`vector_cosine_ops`, `vector_ip_ops`, `vector_l2_ops`) | Match the embedding model: normalised embeddings → inner product (`<#>`) or cosine; unnormalised → the metric the model was trained on. |

---

### Worked Example: Requirement → Decision

**Given:** You are building a customer-support RAG assistant. Support articles (~2M chunks) must be retrievable by semantic similarity, always scoped to the asking customer's tenant, and biased toward recent articles. The app already runs on Postgres for users, tenants, and article metadata. Retrieval latency budget is p95 < 150 ms; strict tenant isolation is a hard security requirement.

- **Step 1 — Identify the goal.** Return the top-k article chunks most semantically similar to a user question, filtered to one tenant, ranked with a recency nudge, fast enough for interactive chat.
- **Step 2 — Define inputs.** A 768-dim (normalised) query embedding, `tenant_id`, and the current time; stored side: chunk text, its embedding, `tenant_id`, `published_at`.
- **Step 3 — Define outputs.** k=8 chunks with id, text, and distance, all belonging to the requesting tenant, ready to stuff into the LLM prompt.
- **Step 4 — Apply constraints.** 2M vectors (well within single-node Postgres), a hard tenant filter (post-filter starvation risk), p95 < 150 ms, strict isolation (a cross-tenant result is a breach), and an existing Postgres investment (no new datastore to operate).
- **Step 5 — Select the approach.** Use **Postgres + pgvector with an HNSW index (cosine/inner-product) plus a B-tree on `tenant_id`**, scoping every query with `WHERE tenant_id = $1` and enabling `hnsw.iterative_scan` so the tenant filter never starves results. Rationale vs alternatives: a dedicated vector DB adds an operational system and a cross-store consistency problem for only 2M vectors that fit comfortably in Postgres; IVFFlat would build faster but miss the p95 recall target HNSW hits; keeping vectors beside relational metadata makes tenant isolation enforceable in one `WHERE` clause instead of two systems that could drift.

---

## Implementation

```sql
-- Scenario: co-locate support-article embeddings with tenant metadata in the
-- Postgres you already run, so a single filtered query does tenant-scoped
-- semantic retrieval for RAG (no second datastore to keep consistent).
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE doc_chunks (
    id           bigserial PRIMARY KEY,
    tenant_id    int         NOT NULL,
    content      text        NOT NULL,
    published_at timestamptz NOT NULL DEFAULT now(),
    embedding    vector(768) NOT NULL          -- normalised model output
);

-- HNSW for the metric the model was trained with (cosine).
-- Build concurrently in prod to avoid blocking writes.
CREATE INDEX CONCURRENTLY ON doc_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128);

-- Supporting B-tree so the hard tenant filter stays fast.
CREATE INDEX CONCURRENTLY ON doc_chunks (tenant_id);

-- Query time: raise recall for this session, then filtered top-k search.
SET hnsw.ef_search = 100;
SELECT id, content, embedding <=> $1 AS distance
FROM doc_chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1          -- <=> = cosine distance
LIMIT 8;
```

```sql
-- Anti-pattern: brute-force scan with the WRONG distance metric and no index.
-- The embeddings are cosine-normalised, but this uses L2 (<->) with no HNSW
-- index and no LIMIT-friendly ordering, so Postgres scans EVERY row on every
-- query. Slow at scale, and L2 ranking disagrees with how the model measures
-- similarity — recall quality silently degrades.
SELECT id, content
FROM doc_chunks
WHERE embedding <-> $1 < 0.5;      -- distance filter, no ORDER BY + LIMIT ⇒ no index use

-- Correct approach: use the metric matching the model (cosine <=>), an HNSW
-- index for that operator class, and ORDER BY + LIMIT so the index is used.
CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);

SELECT id, content, embedding <=> $1 AS distance
FROM doc_chunks
ORDER BY embedding <=> $1
LIMIT 8;
-- Index requires: an ORDER BY on the distance operator, ascending, plus LIMIT.
```

---

## Common Pitfalls & Misconceptions

- **"A vector database replaces my relational database."** Beginners see semantic search as strictly more powerful and assume it supersedes SQL. In reality vectors answer "what's similar?" and cannot enforce foreign keys, transactions, or exact lookups — the correct model is a system of record (relational) *plus* a similarity index, ideally co-located via pgvector.
- **Post-filter starvation with approximate indexes.** People add a `WHERE tenant_id = 42` and expect `LIMIT 10` to return 10 rows, not realising ANN filtering happens *after* the graph scan, so a selective filter can return almost nothing. Fix by adding a B-tree on the filter column, using partial/partitioned indexes, or enabling `hnsw.iterative_scan`.
- **Wrong distance metric for the embedding model.** Defaulting to L2 because it "feels standard" mismatches models trained for cosine/inner product and quietly hurts ranking quality. The metric — and the index's operator class — must match how the model was trained (normalised → inner product/cosine).
- **Building an IVFFlat index on an empty or tiny table.** Newcomers create the index first out of habit; IVFFlat's clusters come from k-means on existing data, so building early yields garbage clusters and poor recall. Load representative data first, then build (HNSW has no such requirement).
- **Reaching for a dedicated vector DB too early.** The word "vector database" sounds mandatory for AI, so teams add one on day one. For up to tens of millions of vectors, pgvector inside your existing Postgres avoids a second system to operate and a cross-store consistency problem — split only when a measured limit forces it.

---

## Key Definitions

| Term | Definition |
|---|---|
| ACID | The transactional guarantee (Atomicity, Consistency, Isolation, Durability) that a group of statements commits all-or-nothing and, once committed, survives crashes. |
| Embedding | A fixed-length float vector produced by a model such that semantic similarity corresponds to geometric closeness. |
| ANN (Approximate Nearest Neighbour) | Search that returns *probably* the closest vectors by touching a subset of data, trading a little recall for large speed gains over exact search. |
| HNSW | Hierarchical Navigable Small World — a layered proximity-graph ANN index with the best speed–recall trade-off; tuned via `m`, `ef_construction`, `ef_search`. |
| IVFFlat | Inverted-file ANN index that k-means-clusters vectors into `lists` and scans only the nearest `probes`; faster to build, lower query quality; must be built on populated data. |
| Recall | Fraction of the true nearest neighbours an approximate search actually returns; the quality axis traded against latency. |
| pgvector | Postgres extension adding a `vector` type, distance operators (`<->`, `<#>`, `<=>`), and HNSW/IVFFlat indexes so vector search runs inside Postgres. |
| Hybrid search | Retrieval that fuses lexical (BM25/full-text) and vector rankings, commonly with Reciprocal Rank Fusion, to capture both exact terms and semantics. |

---

## Summary / Quick Recall

- Choose the store by **query shape**: exact/relational → Postgres; key/document fetch at scale → NoSQL; "most similar" → vector search.
- ACID + joins are relational's moat; horizontal scale + flexible schema are NoSQL's; approximate similarity ranking is the vector store's.
- **HNSW** = best speed/recall, more memory, no training, tuned by `ef_search`; **IVFFlat** = cheaper build, needs data first, tuned by `lists`/`probes`.
- Match the **distance metric to the embedding model** and index the operator class you query; use `ORDER BY <op> … LIMIT` or the index won't be used.
- With approximate indexes, **filters apply after the scan** — mitigate post-filter starvation with B-trees, partial/partition indexes, or iterative scans.
- **Hybrid search** (BM25 + vector via RRF) rescues exact-token queries semantic search alone fails on.
- Default to **pgvector in the Postgres you already run**; move to a dedicated vector DB only when scale or specialised features force it.

---

## Self-Check Questions

1. What guarantee distinguishes a relational database's transaction, and what does each letter of ACID mean?

   <details><summary>Answer</summary>

   The transaction is **all-or-nothing and durable**. ACID = **A**tomicity (all steps commit or none do), **C**onsistency (the DB moves between valid states honouring constraints), **I**solation (concurrent transactions don't see each other's partial work), **D**urability (once committed, changes survive a crash because they're logged to durable storage first). The tempting wrong answer — "it just makes queries faster" — confuses indexing (a performance feature) with transactional correctness; ACID is about integrity, not speed.

   </details>

2. You store 768-dim OpenAI embeddings (normalised) and need low-latency top-10 semantic search in Postgres. Which index and distance operator do you choose, and why?

   <details><summary>Answer</summary>

   An **HNSW index** with an **inner-product (`<#>`) or cosine (`<=>`) operator class**. HNSW gives the best speed–recall trade-off for latency-sensitive reads, and because the embeddings are normalised, inner product and cosine are equivalent (and inner product is fastest). Choosing L2 (`<->`) would be wrong here — the model's similarity is angular, so L2 ranking disagrees with it; choosing IVFFlat would build faster but sacrifice recall you don't need to sacrifice at this scale.

   </details>

3. A multi-tenant RAG query `WHERE tenant_id = 42 ORDER BY embedding <=> $1 LIMIT 10` on an HNSW index returns only 2 rows even though tenant 42 has thousands of documents. What's happening and how do you fix it?

   <details><summary>Answer</summary>

   **Post-filter starvation**: with an approximate index the `tenant_id` filter is applied *after* HNSW scans its default `ef_search = 40` candidate list, so if most candidates belong to other tenants, few survive. Fixes: add a **B-tree on `tenant_id`** (a selective filter can then use fast exact NN), use a **partial index or LIST partition per tenant**, or enable **`hnsw.iterative_scan = strict_order`** so pgvector scans more of the graph until it finds 10 matches. Simply raising `LIMIT` does not fix it — the candidate list, not the limit, is the bottleneck.

   </details>

4. **Which TWO** of the following are correct reasons to keep embeddings in Postgres via pgvector rather than adopting a dedicated vector database?
   - A. pgvector performs approximate search faster than any dedicated vector DB at billion-vector scale.
   - B. Embeddings commit or roll back in the same ACID transaction as their source rows, avoiding cross-store drift.
   - C. A single `WHERE` clause can enforce tenant/metadata filtering alongside similarity, without syncing two systems.
   - D. pgvector removes the need to choose a distance metric.
   - E. Dedicated vector DBs cannot do metadata filtering at all.

   <details><summary>Answer</summary>

   **B and C.** B is correct because keeping vectors in Postgres means an insert of a document and its embedding is one transaction — no eventual-consistency gap between a separate vector store and the system of record. C is correct because co-location lets one query both filter on relational columns and rank by similarity, avoiding a two-system sync. A is wrong — dedicated vector DBs are designed for and typically outperform pgvector at billion-scale sharded workloads (that's exactly the tipping point to leave Postgres). D is wrong — you still choose an operator class matching the model. E is wrong — dedicated vector DBs do support metadata filtering; the most tempting distractor, A, overstates pgvector's scaling ceiling.

   </details>

5. Your team must decide between HNSW and IVFFlat for a 5M-vector index that is rebuilt nightly from a batch pipeline, with a strict overnight build window but relaxed daytime recall needs. Which do you pick and what's the trade-off you're accepting?

   <details><summary>Answer</summary>

   Lean **IVFFlat**. Its k-means-based build is faster and uses less memory, which fits a tight nightly build window, and it's built on the fully-populated batch table (satisfying IVFFlat's "build on real data" requirement). The trade-off accepted is **lower query-time recall/speed** than HNSW — acceptable because daytime recall needs are relaxed. Choosing HNSW would give better daytime quality but slower, more memory-hungry builds that could blow the overnight window; the decision is explicitly buying build-time headroom at the cost of some query quality. Tune `lists ≈ sqrt(5M)` and `probes ≈ sqrt(lists)` as starting points.

   </details>

---

## Further Reading

- [pgvector — Open-source vector similarity search for Postgres](https://github.com/pgvector/pgvector) — *verified 2026-07-28* — Authoritative reference for the `vector` type, distance operators, HNSW/IVFFlat parameters, filtering, and hybrid search.
- [PostgreSQL 18 Documentation — 3.4. Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html) — *verified 2026-07-28* — Canonical explanation of ACID atomicity, durability, and isolation via `BEGIN`/`COMMIT`/`ROLLBACK`.
- [PostgreSQL 18 Documentation — 12.1. Full Text Search Introduction](https://www.postgresql.org/docs/current/textsearch-intro.html) — *verified 2026-07-28* — `tsvector`/`tsquery`, the `@@` match operator, and ranking — the lexical half of hybrid search.
- [PostgreSQL 18 Documentation — 11.2. Index Types](https://www.postgresql.org/docs/current/indexes-types.html) — *verified 2026-07-28* — B-tree, GIN, and other index types used for the metadata-filter columns alongside vector indexes.
