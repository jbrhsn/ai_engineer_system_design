# Embeddings and Vector Search — Interview Prep

**Section:** LLM Serving & RAG Architecture → Embeddings & Vector Search | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is the difference between cosine similarity and dot product, and when are they equivalent? | Cosine measures the *angle* (magnitude-ignoring, range −1..1); dot product combines angle **and** magnitude so longer vectors score higher; they produce identical rankings **only when vectors are L2-normalized**, in which case dot product is just faster. Pick the metric the model was trained/documented for. | "They're always the same" or "cosine is always the right default" — ignores normalization as the precondition, and that a metric mismatch with the model silently returns wrong neighbors with no error. |
| Why use approximate nearest neighbor (ANN) instead of exact KNN at scale? | Exact KNN is O(N·d) per query — at 50M × 1536 that's ~51B multiply-adds per query, impossible under interactive latency. ANN pre-organizes vectors (graph/clusters) so a query touches a small subset, trading a little recall (measured as recall@k vs exact) for orders-of-magnitude lower latency. | "ANN finds the nearest match" — ANN returns a *probable* top-k; treating it as an exact oracle, and not measuring recall@k against brute force, is the tell. |
| Explain the HNSW `ef_search` knob and its tradeoff. | `ef_search` is the query-time candidate-list size; larger = explores more of the graph = higher recall but higher latency. It's the *post-build* recall dial (no rebuild needed), unlike `M` and `ef_construction` which only take effect on rebuild. Raise until recall clears target, then trim to protect p95. | "Just crank all the knobs for safety" — maxing `M`, `ef_construction`, `ef_search` together blows up memory, build time, and latency; each knob slides you along the recall × latency × memory triangle. |
| Why is RRF preferred over adding raw BM25 and cosine scores? | RRF sums `1/(k+rank)` (default k=60) per doc — it fuses by **rank**, not score, sidestepping the incomparable-scale problem (BM25 unbounded and corpus-dependent; cosine ~[-1,1]). It needs no tuning and works even when the two signals are unrelated. Raw addition lets the larger-magnitude BM25 score silently drown cosine. | "Just add the scores and weight them" — without min-max normalization the scales are incomparable; also claiming RRF replaces a reranker (it improves recall/fusion, orthogonal to precision reranking). |
| Contrast a bi-encoder and a cross-encoder — why is one for retrieval and the other for reranking? | Bi-encoder encodes query and doc *separately* → doc vectors precompute and index → million-scale retrieval, but query never attends to doc, capping accuracy. Cross-encoder encodes query+doc *jointly* → full cross-attention → far more accurate, but one forward pass **per query-doc pair at query time**, no precompute → only rerank tens–hundreds. | "The cross-encoder is just a bigger/better model" — misses that the cost is *joint, per-pair, query-time* compute, and that you therefore can't cross-encode the whole corpus. |
| Why must you re-embed the entire corpus when you change the embedding model? | Each model+config defines its own vector space; query vectors from a new model and document vectors from the old model live in incompatible geometries, so nearest neighbors become near-random regardless of the new model's MTEB score. A model change (including a fine-tune) means re-embed the whole corpus and rebuild the index, then use one model for both sides. | "The embedding model is a swappable API key" — updating only the query path or reusing the old index; blaming the "worse model" when its higher benchmark refutes that — the problem is the mismatch. |
| When would you truncate embedding dimensions, and when is it unsafe? | Safe only for Matryoshka (MRL)-trained models (e.g. OpenAI `text-embedding-3-*`), which pack importance into leading dims — truncate `vector[:256]` + re-normalize, or use the `dimensions` API param, to shrink storage with minimal quality loss. Unsafe on an ordinary model, which spreads meaning arbitrarily across all dims — slicing destroys quality. | "Bigger embeddings are always better" / "you can always slice a vector to save space" — ignores that non-MRL truncation is destructive and that higher dims cost storage/memory/latency without guaranteed recall gains. |

---

## Applied / Scenario Questions

**Q1:** You must serve top-10 retrieval over **50M** embeddings (d=1536, L2-normalized) at **p95 < 50ms** with **recall@10 ≥ 0.95** on a single well-provisioned box. Queries usually carry a `tenant_id` filter matching ~2% of rows. How do you build and tune the index?

**Strong answer framework:**
- Name the corner first: this is a *high-recall, low-latency* target, so **HNSW** (best speed-recall curve) over IVFFlat; set `M = 32`, `ef_construction = 200` at build, then tune `ef_search` up from 100 until recall@10 ≥ 0.95 measured against exact search, then trim `ef_search` to the smallest value still clearing the target to protect p95.
- Handle the selective filter with a **filter-aware / pre-filter** path — a payload index on `tenant_id` built before ingest (dedicated DB) or per-tenant partitioned/partial indexes / `hnsw.iterative_scan = strict_order` (pgvector) — because naïve post-filtering an `ef_search=40` set against a 2% predicate **starves** the result to ~1 row.
- Fit the index in RAM with `halfvec`/fp16 (~154 GB vs ~307 GB fp32) or binary quantization + re-rank on originals.
- **Tradeoff (latency vs accuracy vs cost vs safety):** IVFFlat is cheaper RAM and faster to build but its worse speed-recall curve makes 0.95@50ms harder; IVFPQ fits memory trivially but lossy codes jeopardize 0.95 recall without a re-rank stage; HNSW spends *memory* to buy the latency-recall target. Multi-tenant isolation is also a *safety* concern — partitioning prevents one tenant's vectors from polluting another's recall and prevents cross-tenant leakage. Hold IVFPQ in reserve for 500M+ vectors where RAM becomes the binding constraint.

**Q2:** A support RAG assistant over 2M help-centre chunks "answers a related-but-wrong article." Recall@50 is already 0.94 but precision@5 is poor. End-to-end p95 must be ≤ 800ms and the LLM call already eats ~500ms, leaving ~300ms for retrieve+rerank. How do you lift precision?

**Strong answer framework:**
- Recall is already high, so the problem is *precision*, not retrieval coverage — add a **two-stage retrieve-then-rerank**: keep hybrid retrieval with **RRF (rank_constant 60)**, set **top-k = 50**, add a distilled **MiniLM-L6 cross-encoder** over those 50, and emit **top-n = 5** to the LLM.
- Keep the BM25 leg in hybrid so exact ticket IDs / error codes (e.g. "E-4021") aren't smeared away by dense retrieval — this is both a quality and a correctness requirement.
- Verify the budget empirically: a MiniLM-L6 cross-encoder scores ~50 pairs in ~60–120ms on the target GPU, comfortably inside the ~300ms slice.
- **Tradeoff (latency vs accuracy vs cost vs safety):** cross-encoding the full 2M corpus is infeasible (millions of forward passes ≫ budget) and a larger/multilingual reranker would blow the 300ms slice; ColBERT late interaction would help precision but forces re-indexing 2M docs into multi-vector storage (index cost) — unjustified when a cross-encoder over 50 candidates already meets target. Choose the smallest top-n that keeps answer quality flat: padding context adds cost, latency, and distraction.

---

## System Design / Architecture Questions

**Q:** Design the retrieval layer for a large **multi-tenant semantic search** system: thousands of tenants, hundreds of millions of chunks total, a p95 latency budget, strict tenant isolation, and a "why can't it find exact SKUs?" complaint from users.

**Approach:**
1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity):** total vector count and per-tenant skew; p95 latency and recall@k SLO; query mix (natural-language vs identifier/code-heavy); language coverage; isolation model (shared index + filter vs per-tenant index); freshness/re-embed cadence; and whether tenants can ever see each other's data (hard isolation requirement).
2. **Propose high-level architecture (topology, retrieval layer, guardrails):** one embedding model chosen by MTEB *retrieval* score plus domain/language/max-seq-length fit, benchmarked on an in-domain eval set (multilingual model if tenants span languages). Hybrid retrieval — dense over an HNSW index + BM25/sparse leg for exact SKUs/codes — fused with RRF, then a cross-encoder rerank of top-k → top-n. Tenant isolation via **filter-aware pre-filter** (payload index on `tenant_id`) or per-tenant partitions so filtering never starves and cross-tenant vectors never mix. Scale with **sharding** for capacity (fan-out + merge; note per-shard-local recall) and **replication** per shard for QPS/HA. Store vectors as `halfvec` or IVFPQ-compressed for the largest shards.
3. **Justify choices and name tradeoffs explicitly (cost, latency, complexity, security):** HNSW spends memory for the best latency-recall curve; IVFPQ trades recall for RAM at extreme scale (mitigate with re-rank). Shared-index + metadata filter is cheaper/simpler but leans entirely on filter correctness for isolation; per-tenant indexes are stronger isolation and better noisy-neighbor behavior at higher operational cost. Hybrid + rerank adds latency and a reranker cost line but is the only way to satisfy both semantic recall and exact-SKU precision. Re-embedding on any model change is a first-class operational cost (rebuild the whole corpus + index) and must be planned, not treated as a config toggle.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:
- **Cosine similarity / dot product / L2-normalization** — say "if vectors are normalized, cosine and dot product rank identically, so we use dot product for speed" to show you understand the precondition, not the buzzword.
- **ANN / recall@k** — frame retrieval quality as recall@k measured against exact search, not "it returns the closest."
- **HNSW `M` / `ef_construction` / `ef_search`** — name which are build-time vs the query-time recall dial; signals you know what needs a rebuild.
- **IVF `nlist` / `nprobe`, coarse quantizer** — "`nlist ≈ √n`, raise `nprobe` for recall"; note IVF must train on representative data.
- **Product quantization / IVFPQ / `nbits`** — for billion-scale, memory-constrained ANN with a re-rank stage to recover recall.
- **Recall × latency × memory triangle** — "name the corner you're prioritizing first, then pick the index."
- **Pre-filter vs post-filter / filter-aware index / payload index** — the vocabulary of avoiding post-filter starvation.
- **RRF / rank_constant / relative-score fusion / `alpha`** — rank-based fusion over incomparable scales.
- **Bi-encoder / cross-encoder / late interaction (ColBERT / MaxSim)** — the accuracy/latency/index-cost spectrum of scoring.
- **MTEB (retrieval subset)** — model selection grounded in benchmark *plus* domain/language/seq-length fit.
- **Matryoshka (MRL) / `dimensions`** — dimension truncation done safely.
- **Asymmetric search / query & passage prompts** — RAG retrieval as a query↔passage problem.

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:
- **"Just use cosine everywhere"** — ignores normalization and that the metric must match how the model was trained; a mismatch fails silently.
- **"ANN gives exact results"** — ANN is approximate *by design*; the tell is not knowing to measure recall@k against brute force.
- **"Reranking is always worth it"** — a reranker only reorders what retrieval returned; if recall@k is low it can't add the gold doc, and it always costs latency.
- **"Bigger embeddings are always better"** — more dims cost storage/memory/latency without guaranteed recall gains, and you can't safely slice a non-Matryoshka vector.
- **"Vector search replaces keyword search"** — dense retrieval systematically loses exact identifiers/SKUs/codes; that's why the BM25 leg exists.
- **"Just swap in a better embedding model"** — without re-embedding the corpus this destroys retrieval; treating the model as a hot-swappable API key is a red flag.
- **"Add the BM25 and cosine scores together"** — incomparable scales; the larger-magnitude signal dominates.
- **"Crank all the index knobs to be safe"** — reveals no mental model of the recall × latency × memory triangle.

---

## STAR Answer Frame

**Situation:** A production RAG support assistant was returning "related-but-wrong" articles; users especially complained it couldn't find articles by exact error codes, and offline precision@5 was poor despite retrieval feeling "roughly right."
**Task:** I owned retrieval quality and had to raise precision@5 without breaking the 800ms end-to-end p95 budget (the LLM call already consumed ~500ms) and without regressing exact-identifier matches.
**Action:** I first measured recall@50 against exact search and found it was already 0.94, so I diagnosed a *precision* problem, not a coverage one — meaning a reranker could actually help. I kept a hybrid retriever (dense HNSW + BM25 for exact codes) fused with RRF at rank_constant 60, set top-k = 50, added a distilled MiniLM-L6 cross-encoder reranking those 50, and emitted top-n = 5. I benchmarked the reranker at ~60–120ms for 50 pairs to confirm it fit the ~300ms retrieve+rerank slice, and rejected a larger reranker and full-corpus reranking as budget-infeasible.
**Result:** Precision@5 rose substantially while p95 stayed within the 800ms budget; exact-error-code queries recovered because the BM25 leg preserved literal-token matches, and the "wrong article" complaints dropped. The key lever was diagnosing recall vs precision *before* adding cost, not reflexively swapping the embedding model.

---

## Red Flags Interviewers Watch For

- Treating the vector DB as an exact `argmin` oracle — no mention of recall@k, no plan to measure it against brute-force/exact search.
- Reaching for a reranker or a bigger embedding model before checking whether recall@k is the actual bottleneck (fixing precision when the problem is coverage, or vice versa).
- Not knowing which HNSW/IVF knobs require a rebuild (`M`, `ef_construction`, `nlist`) vs which tune at query time (`ef_search`, `nprobe`).
- Ignoring metadata-filter interaction — proposing a selective `WHERE` on an ANN index without addressing post-filter starvation via pre-filter/filter-aware indexes.
- Proposing an embedding-model swap or fine-tune without stating that the entire corpus must be re-embedded and the index rebuilt.
- Adding raw BM25 and cosine scores, or otherwise not recognizing incomparable score scales.
- Suggesting dense-only retrieval for identifier/SKU/code-heavy queries, unaware that dense smears rare exact tokens.
- Truncating embedding dimensions of a non-Matryoshka model to "save space," or claiming higher dimensionality is unconditionally better.
- Confusing sharding (capacity) with replication (throughput/HA), or forgetting that merged top-k across shards is still approximate.
- No explicit prioritization among recall, latency, and memory — treating them as independently maximizable rather than a triangle.
