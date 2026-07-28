# Vector Databases and Similarity Search Indexes

**Section:** LLM Serving & RAG Architecture → Embeddings & Vector Search | **Est. time:** 2.5 hrs | **Interview relevance:** High — "how does your retrieval hit p95 < 50ms over 50M vectors?" is a near-guaranteed follow-up in any RAG system design round.

---

## TL;DR

A vector database answers "which stored vectors are closest to this query vector?" — and at scale it cannot afford to compare against every stored vector, so it uses an **approximate nearest neighbor (ANN) index** that trades a little recall for orders-of-magnitude lower latency. The three dominant index families — HNSW (a navigable graph), IVF (clustering into probed cells), and IVFPQ (IVF plus product-quantization compression) — sit at different points on a **recall × latency × memory** triangle, and every tuning knob moves you along that triangle rather than escaping it. Metadata filtering interacts dangerously with ANN: naïve post-filtering can starve results, so you must understand pre- vs post-filter mechanics. **The one thing to remember: ANN is approximate by design — you tune knobs like `ef_search` or `nprobe` to *buy back* recall with latency, and you must measure recall against exact search rather than assume it.**

---

## ELI5 — Explain It Like I'm 5

Imagine you want the nearest open pizza place to where you're standing, in a city of a million restaurants. Checking every single restaurant's address one by one is guaranteed correct but takes all night — that's brute-force (exact) search. Instead, the city is divided into neighborhoods, and you only walk the two or three neighborhoods around you (that's the IVF "cells" idea); or you follow a chain of "nearest friend of a nearest friend" signposts that hop you across town toward the closest place in a few jumps (that's the HNSW graph idea). Both get you a *very* good answer almost every time — but not a *guaranteed* answer: occasionally the true closest pizza place sits just over a neighborhood boundary you didn't walk into. The common misconception is that a vector database "finds the nearest match"; in reality, once you add an approximate index it finds *a very likely nearest match*, and the setting that controls how many neighborhoods you walk (or how deep you follow signposts) is exactly what you tune to trade thoroughness for speed.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain why brute-force KNN is O(N·d) per query and when that stops being viable, and articulate the exact-vs-approximate trade-off.
- [ ] Describe the mechanism of HNSW, IVF/IVFFlat, and IVFPQ at the level of "what data structure, what happens at build vs query time."
- [ ] Tune `M` / `ef_construction` / `ef_search` (HNSW) and `nlist` / `nprobe` (IVF) and PQ segment count to hit a stated recall and latency target.
- [ ] Diagnose and fix post-filter starvation when combining metadata filtering with an ANN index.
- [ ] Compare a dedicated vector DB against pgvector for a given scale, and justify sharding vs replication for a vector index.

---

## Visual Overview

### HNSW Layered Graph Traversal

```
Query enters at the TOP (sparse) layer and greedily hops toward the target,
then descends layer by layer, refining as the graph gets denser.

Layer 2 (sparse)      (entry)●─────────────────►●
                                                 │ descend
Layer 1 (medium)          ●───►●─────►●─────────►●───►●
                                             │ descend
Layer 0 (all nodes)   ●─►●─►●─►●─►●─►●─►●─►●─►[★ nearest]
                                          ▲
                        ef_search = size of the candidate list kept
                        while exploring each layer (bigger = higher recall)
```

### IVF Cluster / Centroid Probing

```
Vectors are k-means clustered into nlist cells. At query time only the
nprobe cells whose centroid is closest to the query are scanned.

            ┌───────── cell A ─────────┐   ┌──── cell C ────┐
            │  · centroidA ·  · ·      │   │  · centroidC · │
   query ─► │      · · · ·             │   │   · · ·        │   (skipped)
            └──────────────────────────┘   └────────────────┘
            ┌───────── cell B ─────────┐   ┌──── cell D ────┐
            │  · centroidB · · · ·     │   │  · centroidD · │   (skipped)
            └──────────────────────────┘   └────────────────┘

   nprobe = 2  ──►  scan cells A and B only  ──►  ~2/nlist of the data
   FAILURE MODE: true neighbor lives in an unprobed cell (recall miss)
```

### The Recall × Latency × Memory Triangle

```
                         high recall
                              ▲
                              │
                   HNSW ●     │     ● IVFFlat (high nprobe)
        (high M, high ef)     │
                              │
   low memory ◄───────────────┼───────────────► low latency
                              │
                   IVFPQ ●    │
        (heavy compression;   │
         low memory, some     │
         recall traded away)  │

   You pick a CORNER to prioritise; every knob slides you along an edge.
   You never get all three maxed at once.
```

### Pre-filter vs Post-filter Flow

```
POST-FILTER (naïve, can starve):
  query ──► ANN returns top-k (e.g. 40) ──► drop rows failing WHERE ──► ??? left
                                                                     (maybe 4)

PRE-FILTER (filter-aware):
  query ──► restrict candidate set to rows matching WHERE
        ──► ANN searches only within that set ──► top-k that ALL match
```

---

## Key Concepts

### Exact (brute-force KNN) vs Approximate (ANN)

**What it is.** Exact nearest-neighbor search compares the query vector to *every* stored vector and returns the provably closest k; ANN returns the *likely* closest k using an index that skips most comparisons.

**How it works mechanistically.** Exact search is O(N·d) per query — N vectors each requiring a d-dimensional distance computation. At N = 50M and d = 1024 that is ~51 billion multiply-adds per query, which no interactive endpoint can afford. ANN indexes pre-organise the vectors (into a graph or into clusters) so that a query only touches a small, well-chosen subset. Because the subset is chosen heuristically, the true nearest neighbor can occasionally be missed — this is measured as **recall@k** (fraction of the true top-k the index actually returned).

**Where it appears in real systems.** In pgvector, no index = exact search with "perfect recall" (the docs state this explicitly); adding an HNSW or IVFFlat index switches you to ANN and you will "see different results for queries after adding an approximate index." In FAISS, `IndexFlatL2` is the exact/exhaustive baseline every other index is benchmarked against.

### HNSW (Hierarchical Navigable Small World) internals

**What it is.** A multi-layer proximity graph where each node (vector) links to its nearby neighbors, and upper layers are sparse "express lanes" for long hops.

**How it works mechanistically.** Build time: each inserted vector is linked to up to `M` neighbors per layer; the search depth used while wiring those links is `ef_construction`. Query time: search starts at the top sparse layer, greedily walks toward the query, then descends to denser layers, maintaining a dynamic candidate list of size `ef_search`; a larger list explores more of the graph and recovers more true neighbors. Because it is graph-structured, HNSW has excellent speed-recall trade-off but higher memory (it stores the neighbor links) and slower builds — and it does **not** support removing vectors in FAISS's `IndexHNSW` because deletion would break the graph.

**Where it appears in real systems.** pgvector: `CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WITH (m = 16, ef_construction = 64)` and per-query `SET hnsw.ef_search = 100`. FAISS: `IndexHNSWFlat` with `M`, `efConstruction`, `efSearch`. Qdrant builds an HNSW index per segment and can make it *filter-aware* (see filtering below).

### IVF / IVFFlat (inverted-file / clustering)

**What it is.** A "cell-probe" index that partitions the vector space into `nlist` cells via k-means, assigns every vector to its nearest centroid, and at query time scans only the `nprobe` closest cells.

**How it works mechanistically.** Build time trains a coarse quantizer (k-means) producing `nlist` centroids, then assigns each database vector to one inverted list. Query time computes distance from the query to all centroids, picks the `nprobe` nearest lists, and does exact distance computation against just the vectors in those lists. The fraction of data scanned is roughly `nprobe/nlist`, so recall rises with `nprobe`; the failure case is when the true neighbor's cell isn't among the probed ones. FAISS gives the sizing heuristic `nlist = C·√n`; pgvector suggests `rows/1000` up to 1M rows and `√rows` beyond, with a starting `probes ≈ √lists`.

**Where it appears in real systems.** pgvector: `CREATE INDEX ... USING ivfflat (embedding vector_l2_ops) WITH (lists = 100)` and `SET ivfflat.probes = 10`. FAISS: `"IVFx,Flat"` via the index factory, `index.nprobe = 5`. Critically, IVFFlat must be built *after* the table has representative data — the k-means needs real vectors to place centroids well.

### IVFPQ / Product Quantization (compression)

**What it is.** IVF combined with **product quantization**: each vector is split into `M` sub-vectors, and each sub-vector is quantized to a small codebook, so a full vector is stored as `M` compact codes instead of `d` floats.

**How it works mechanistically.** PQ splits the d-dimensional vector into M equal segments (d must be a multiple of M) and learns a codebook (typically `nbits = 8` → 256 centroids) per segment; a vector becomes M bytes instead of 4·d bytes — a ~ (4·d)/M compression ratio. Distances are estimated in the compressed domain using precomputed lookup tables (asymmetric distance computation), which is fast and memory-cheap but *lossy*, so recall drops. IVFPQ layers this onto IVF cells (`IndexIVFPQ` = coarse quantizer + PQ on residuals), the standard structure for billion-scale search; an optional refine stage (`IndexIVFPQR`) re-ranks with finer codes.

**Where it appears in real systems.** FAISS: `"IVFx,PQy"` / `IndexIVFPQ(quantizer, d, nlists, M, nbits)`; storage is `ceil(M·nbits/8)+8` bytes/vector. pgvector's analogous "compress to fit in memory" lever is **binary quantization** with re-ranking on the original vectors — same idea (shrink the index, re-rank to recover recall), different codec.

### The recall × latency × memory triangle

**What it is.** A framing device: every ANN index and its knobs trade off three quantities — recall (answer quality), latency (query speed), and memory (RAM footprint). You cannot maximise all three.

**How it works mechanistically.** HNSW spends memory (neighbor links) to buy the best speed-recall curve. IVFFlat spends less memory and builds fast but has a worse speed-recall curve. IVFPQ spends recall (lossy compression) to buy tiny memory, enabling billion-scale in RAM. Within a chosen index, `ef_search`/`nprobe` trade latency for recall, and quantization bit-width trades recall for memory. In interviews, name the corner you're prioritising *first*, then pick the index.

**Where it appears in real systems.** The pgvector docs state HNSW "has better query performance than IVFFlat ... but has slower build times and uses more memory" — that single sentence *is* the triangle. FAISS's "Guidelines to choose an index" page is organised entirely around these three axes plus dataset size.

### Metadata filtering interaction (pre- vs post-filter)

**What it is.** Combining a `WHERE`-style metadata condition (e.g. `tenant_id = 42`) with an ANN vector search.

**How it works mechanistically.** **Post-filter:** run ANN to get top-k, then discard rows failing the predicate — but ANN already narrowed to k candidates, so if the predicate matches only 10% of rows, an HNSW search with `ef_search = 40` yields ~4 surviving rows, not the requested top-k. This is *post-filter starvation*. **Pre-filter (filter-aware):** restrict the searchable set to matching rows *before/while* traversing, so every returned neighbor satisfies the predicate. Purpose-built vector DBs build filter-aware indexes; pgvector applies filtering after the index scan and offers **iterative index scans** (0.8.0+) to keep scanning until enough matches are found.

**Where it appears in real systems.** pgvector: `SET hnsw.iterative_scan = strict_order;` (or `relaxed_order`) plus partial/partitioned indexes for low-cardinality filters. Qdrant: create a **payload index** on the filtered field *before* ingesting and it produces additional filter-aware edges in the HNSW graph — filtering there speeds up rather than starves search.

### Sharding vs replication of vector indexes

**What it is.** Two orthogonal scaling axes: **sharding** splits the vectors across nodes (each holds part of the index); **replication** copies the same index to multiple nodes.

**How it works mechanistically.** Sharding addresses capacity and build/query parallelism — a query fans out to all shards, each returns its local top-k, and a coordinator merges them; this is how you exceed single-node RAM. Replication addresses throughput and availability — identical replicas serve read queries in parallel and survive node loss. They combine: shard for size, replicate each shard for QPS/HA. Note ANN recall is *per-shard local*, so merged top-k across shards is still approximate.

**Where it appears in real systems.** pgvector scales vertically first, then horizontally with read **replicas** (Postgres hot standby) for throughput and Citus/PgDog for **sharding**. Dedicated vector DBs (e.g. Qdrant) expose shard count and replication factor as first-class collection settings and support tenant-isolated indexes so one tenant's vectors don't degrade another's recall.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| HNSW `M` (max neighbors/layer) | Graph connectivity → recall ceiling & memory | Start at 16; raise to 32–64 for high-dimensional or high-recall needs — but memory and build time grow with it, so don't raise beyond what recall testing justifies. |
| HNSW `ef_construction` | Search depth while *building* the graph | Raise it (e.g. 64→200) when recall is low; costs build/insert time, not query time. Use the default first and only increase if measured recall is unsatisfactory. |
| HNSW `ef_search` | Candidate-list size at *query* time | The primary recall dial post-build: raise it (40→100→200) to improve recall at the cost of latency; lower it to cut p95 latency once recall is comfortably above target. |
| IVF `nlist` (number of cells) | Granularity of the partitioning | Set to ~`√n` for >1M rows (FAISS `C·√n`); more lists = finer cells = need higher `nprobe` for the same recall. Must be built after loading representative data. |
| IVF `nprobe` (cells scanned) | Fraction of data searched per query | The primary recall dial at query time: raise `nprobe` to improve recall at the cost of latency; start near `√nlist`. Setting `nprobe = nlist` = exact search. |
| PQ `M` (sub-quantizers/segments) | Compression ratio & recall | `d` must be divisible by `M`; more segments = less compression but higher recall. Pick the largest `M` that still fits your memory budget, then re-rank to recover recall. |
| PQ `nbits` | Codebook size per segment | Use 8 (256 centroids) as the standard; only 8/12/16 are supported for IVFPQ. Higher = better recall, larger index. |
| Iterative scan (`hnsw.iterative_scan`) | Whether to keep scanning under filters | Enable (`strict_order`/`relaxed_order`) when a selective `WHERE` clause causes too-few results; leave off for unfiltered queries to avoid extra scan cost. |

### Worked Example: Requirement → Decision

**Given:** A production RAG service must retrieve top-10 chunks from **50M** OpenAI-style embeddings (d = 1536, L2-normalised) at **p95 < 50ms**, with **recall@10 ≥ 0.95**, on a single well-provisioned box before considering sharding. Queries usually carry a `tenant_id` filter that matches ~2% of rows.

- **Step 1 — Identify the goal.** Pick an index + knobs that meet recall@10 ≥ 0.95 at p95 < 50ms, and make the tenant filter *not* starve results.
- **Step 2 — Define inputs.** 50M × 1536-dim float vectors (~307 GB raw as fp32; ~154 GB as `halfvec`/fp16), a per-query filter on `tenant_id`, and a recall/latency SLO.
- **Step 3 — Define outputs.** A ranked top-10 list where every item satisfies `tenant_id`, returned within the latency budget.
- **Step 4 — Apply constraints.** Interactive latency budget (50ms p95) rules out brute force (billions of ops/query). The filter selectivity (2%) makes naïve post-filtering starve an HNSW top-40. Memory: fp32 index likely exceeds a single box's RAM, so compression or `halfvec` is on the table.
- **Step 5 — Select the approach.** Use **HNSW** (best speed-recall curve for this latency target) with `M = 32`, `ef_construction = 200`, and tune `ef_search` upward from 100 until measured recall@10 ≥ 0.95, then trim it back to the smallest value still meeting recall to protect p95. Handle the tenant filter with a **filter-aware / pre-filter** path — in a dedicated DB, a payload index on `tenant_id` (built before ingest); in pgvector, either a partitioned/partial index per tenant or `hnsw.iterative_scan = strict_order`. Store vectors as `halfvec` (or binary-quantize with re-ranking) to fit the index in RAM. *Rationale vs alternatives:* IVFFlat builds faster and uses less RAM but its speed-recall curve makes 0.95@50ms harder; IVFPQ would fit memory trivially but its lossy codes jeopardise the 0.95 recall target without a re-rank stage — so HNSW + `halfvec` is the balanced pick, with IVFPQ held in reserve if RAM becomes the binding constraint at 500M+ vectors.

---

## Implementation

```sql
-- Scenario: 50M-vector RAG index that must hit recall@10 >= 0.95 at p95 < 50ms.
-- Build HNSW with generous construction depth, then use ef_search as the
-- per-query recall dial. Build AFTER bulk-loading data, concurrently, and
-- with large maintenance_work_mem so the graph fits in RAM during build.

SET maintenance_work_mem = '8GB';        -- graph build stays in memory
SET max_parallel_maintenance_workers = 7;

CREATE INDEX CONCURRENTLY items_embedding_hnsw
  ON items USING hnsw (embedding vector_cosine_ops)
  WITH (m = 32, ef_construction = 200);

-- Per-query: raise ef_search to buy recall; measure, then trim to protect p95.
SET hnsw.ef_search = 100;
SELECT id FROM items ORDER BY embedding <=> :query_vec LIMIT 10;
```

```sql
-- Anti-pattern: post-filtering a selective tenant predicate after the ANN scan.
-- HNSW returns ef_search (=40) candidates FIRST, THEN the WHERE drops ~98% of
-- them (tenant matches only 2% of rows), so you get ~1 row instead of 10.
-- Recall collapses and the result set is "starved."

SET hnsw.ef_search = 40;
SELECT id FROM items
WHERE tenant_id = 42                        -- applied AFTER the index scan
ORDER BY embedding <=> :query_vec
LIMIT 10;                                   -- frequently returns < 10 rows

-- Correct approach: make the search filter-aware and give it room to find
-- enough matches. Enable iterative index scans, and for a small set of tenants
-- use partitioning/partial indexes so each tenant scans its own sub-index.

SET hnsw.iterative_scan = strict_order;     -- keep scanning until 10 matches (pgvector 0.8.0+)
SET hnsw.ef_search = 100;                   -- more candidates per pass, higher recall

-- Tenant isolation via list partitioning: filter becomes partition pruning,
-- so vectors from other tenants never pollute recall or latency.
CREATE TABLE items (tenant_id int, embedding vector(1536))
  PARTITION BY LIST (tenant_id);
```

---

## Common Pitfalls & Misconceptions

- **Assuming ANN returns the exact nearest neighbors** — beginners treat the vector DB as a `k`-argmin oracle because it "just returns the closest." An approximate index returns a *probable* top-k; you must measure recall@k against exact search (in pgvector, `SET LOCAL enable_indexscan = off` to force exact) and tune `ef_search`/`nprobe` to hit your target.
- **Distance-metric mismatch between index and model** — people build the index with the default L2 ops while their embeddings are meant for cosine, because the operator class isn't obvious. The index must use the metric the model was trained for (`vector_cosine_ops` for cosine, or inner product on normalised vectors); a metric mismatch silently returns wrong neighbors with no error.
- **Post-filter starvation on selective predicates** — engineers add a `WHERE tenant_id = ...` and expect a full top-k, not realising the filter runs *after* the ANN grabbed only `ef_search` candidates. Use a filter-aware index (payload index / pre-filter) or enable iterative scans; on a highly selective filter an exact index on the filter column can even beat ANN.
- **Building an IVF index on empty or tiny data** — teams create the `ivfflat` index first, then load, and get terrible recall because k-means placed centroids on almost no data. Always load representative data *before* building IVF (and re-train `nlist` if the corpus grows a lot); HNSW is exempt since it has no training step.
- **Maxing every knob "for safety"** — raising `M`, `ef_construction`, `ef_search`, and `nprobe` all at once because "higher = better" blows up memory, build time, and latency. Each knob moves you along the recall/latency/memory triangle; change one at a time and stop the moment you clear the SLO.

---

## Key Definitions

| Term | Definition |
|---|---|
| ANN (Approximate Nearest Neighbor) | Search that returns *likely* nearest vectors by inspecting a chosen subset, trading guaranteed correctness for large speed/memory gains. |
| Recall@k | Fraction of the true top-k nearest neighbors that the approximate index actually returns; the quality metric for ANN. |
| HNSW | Hierarchical Navigable Small World — a layered proximity graph traversed greedily from sparse to dense layers to reach nearest neighbors. |
| IVF / IVFFlat | Inverted-file (cell-probe) index: k-means partitions space into `nlist` cells; query scans the `nprobe` nearest cells. |
| Product Quantization (PQ) | Compression that splits a vector into M sub-vectors, each quantized to a small codebook, storing M codes instead of d floats. |
| IVFPQ | IVF cells combined with PQ-compressed residuals — the standard structure for billion-scale, memory-constrained ANN. |
| Coarse quantizer | The index (usually flat k-means) that assigns vectors/queries to IVF cells. |
| Pre-filter vs post-filter | Applying a metadata predicate *during/before* ANN traversal (filter-aware) vs *after* the ANN returns candidates (starvation-prone). |
| Sharding | Splitting the vector set across nodes to exceed single-node capacity; queries fan out and results are merged. |
| Replication | Copying the same index to multiple nodes for higher read throughput and availability. |

---

## Summary / Quick Recall

- Brute-force KNN is O(N·d)/query and dies at millions of vectors; ANN indexes touch a subset and are approximate *by design*.
- HNSW = graph, best speed-recall, more memory, slower build, no deletes; tune `M`/`ef_construction` at build, `ef_search` at query.
- IVF/IVFFlat = cluster + probe; cheaper memory, must train on real data; `nlist ≈ √n`, raise `nprobe` for recall.
- IVFPQ = IVF + product quantization; tiny memory for billion-scale, lossy so re-rank to recover recall.
- Every knob slides you along the recall × latency × memory triangle — name the corner you want first.
- Selective metadata filters starve post-filtered ANN; use filter-aware/pre-filter indexes or iterative scans.
- Shard for size, replicate for throughput/HA; pgvector suits moderate scale, dedicated DBs shine at large filtered/multi-tenant scale.

---

## Self-Check Questions

1. What does "recall@10 = 0.9" tell you about an ANN index, and how would you measure it?

   <details><summary>Answer</summary>

   It means the index returns, on average, 9 of the 10 truly-closest vectors for a query — the other 1 is a miss caused by the approximate traversal. You measure it by comparing the index's results against exact/brute-force search on the same queries (in pgvector, force exact with `SET LOCAL enable_indexscan = off`; in FAISS, compare against `IndexFlatL2`). The tempting wrong answer is that 0.9 refers to distance accuracy or similarity score — it does not; recall is about *set overlap* with the true top-k, independent of the distance values.

   </details>

2. You have an HNSW index and your `recall@10` is 0.82, below your 0.95 target. The index is already built. What is the first knob to change and why?

   <details><summary>Answer</summary>

   Raise `hnsw.ef_search` (e.g. 40 → 100 → 200). It enlarges the query-time candidate list so the traversal explores more of the graph, recovering more true neighbors — and it requires **no rebuild**, unlike `M` or `ef_construction`. Changing `M` would help the recall ceiling but forces a full rebuild and more memory; `ef_construction` also only takes effect on rebuild. So `ef_search` is the correct first move because it's the post-build recall dial, paid for in latency.

   </details>

3. **Which TWO** of the following will reliably improve recall for an IVFFlat index at query time or reduce the risk of missing the true neighbor?
   - A. Increase `nprobe`
   - B. Decrease `nprobe`
   - C. Build the index on an empty table then insert data
   - D. Ensure the index's distance metric matches the embedding model's metric
   - E. Set `nlist` to 1

   <details><summary>Answer</summary>

   **A and D.** Raising `nprobe` scans more cells, so the true neighbor's cell is more likely included (higher recall, higher latency). Matching the distance metric (e.g. cosine ops for cosine-trained embeddings) ensures "closest" is computed correctly — a mismatch silently returns wrong neighbors. B lowers recall. C is the classic IVF anti-pattern: k-means trains on no data and places centroids badly, wrecking recall (build *after* loading). E collapses to a single cell — that's effectively exact search but defeats the purpose and offers no partitioning benefit; it's the tempting distractor because "search everything = perfect recall," but it abandons the speed gain the index exists for and isn't how you'd tune IVF.

   </details>

4. A query with `WHERE tenant_id = 42` (matches ~2% of rows) against an HNSW index keeps returning only 1–3 results instead of the requested top-10. Diagnose the cause and give one fix for pgvector and one for a dedicated vector DB.

   <details><summary>Answer</summary>

   This is **post-filter starvation**: HNSW first returns `ef_search` (default 40) candidates, then the 2%-selective predicate discards ~98% of them, leaving ~1. Fix in pgvector: enable `hnsw.iterative_scan = strict_order` (0.8.0+) so it keeps scanning until it finds enough matches, and/or use partial/partitioned indexes per tenant so the filter becomes partition pruning; raising `ef_search` also helps marginally. Fix in a dedicated DB (e.g. Qdrant): create a **payload index** on `tenant_id` before ingest so the HNSW graph is *filter-aware* (pre-filter). The wrong diagnosis is "the index is corrupt" or "recall is just low" — the result count drops specifically because filtering happens after the ANN candidate cutoff.

   </details>

5. You must serve 500M vectors but the fp32 HNSW index won't fit in a single node's RAM, and you cannot yet add nodes. Compare using IVFPQ vs sharding, and state which trade-off each makes.

   <details><summary>Answer</summary>

   **IVFPQ** compresses each vector into `M` PQ codes (e.g. from 4·d bytes to M bytes), shrinking the index enough to fit one node's RAM — the trade-off is **recall**, because PQ is lossy; you mitigate with a re-rank stage (`IVFPQR` / re-rank on original vectors). **Sharding** keeps full-precision vectors but splits them across nodes — the trade-off is **operational cost and complexity** (multiple nodes, fan-out/merge, per-shard-local recall), and it's ruled out here by "cannot yet add nodes." Under the single-node constraint, IVFPQ (with re-ranking) is the pragmatic choice; you accept some recall loss to stay on one box. The tempting wrong answer is "just use HNSW with lower `ef_search`" — that cuts latency, not the memory footprint, so it doesn't solve the binding RAM constraint.

   </details>

---

## Further Reading

- [pgvector — README (HNSW & IVFFlat indexing, filtering, iterative scans, scaling)](https://github.com/pgvector/pgvector) — *verified 2026-07-29* — Authoritative reference for `m`/`ef_construction`/`ef_search`, `lists`/`probes`, post-filter behavior, and replication/sharding options.
- [FAISS — Faiss indexes (index families, IVF cell-probe, PQ, HNSW parameters)](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes) — *verified 2026-07-29* — Mechanistic reference for `IndexFlat`, `IndexHNSWFlat`, `IndexIVFFlat`, `IndexIVFPQ`, the `nlist = C·√n` heuristic, and PQ `M`/`nbits`.
- [Qdrant — Indexing (payload/filterable HNSW, tenant & principal indexes)](https://qdrant.tech/documentation/manage-data/indexing/) — *verified 2026-07-29* — Dedicated vector DB perspective on filter-aware HNSW, building payload indexes before ingest, and multitenancy.
- [Qdrant — Filtering (must/should/must_not clauses and pre-filter semantics)](https://qdrant.tech/documentation/search/filtering/) — *verified 2026-07-29* — How metadata conditions combine with vector search and why payload indexes make filtering performant rather than starving.
