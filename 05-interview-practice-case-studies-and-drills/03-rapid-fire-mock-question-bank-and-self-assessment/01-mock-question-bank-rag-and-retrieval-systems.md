# Mock Question Bank — RAG and Retrieval Systems

**Section:** Interview Practice → Rapid-Fire Mock Question Bank & Self-Assessment | **Use:** rapid-fire self-drill

---

## How to Use This Bank

Run this as a timed drill: read each question, say the answer *aloud* (or scribble it) **before** you expand the `<details>` block — no peeking. Target ~30 seconds per Rapid-Fire question, ~90 seconds per Applied/Analysis question, and ~45 seconds per MCQ. Score yourself in the scorecard at the bottom, then re-drill only the rows you marked △ or ✗ against the linked section 02 chapters.

---

## Rapid-Fire Round (Recall)

**R1. What is an embedding, in one sentence?**

<details><summary>Answer</summary>

A fixed-length vector of floats produced by an encoder that positions text by *meaning*, so semantically similar text lands nearby and you can retrieve by nearest-neighbour search instead of keyword matching. Not a keyword list — meaning is spread across all dimensions, so "car" and "automobile" land close despite sharing no letters.

</details>

**R2. When are cosine similarity and dot product interchangeable?**

<details><summary>Answer</summary>

When the vectors are **L2-normalized** (length 1). Then cosine reduces to a dot product and both give identical rankings — so you use the dot product because it's faster. On *unnormalized* vectors they can rank differently, because dot product also rewards magnitude while cosine only measures angle.

</details>

**R3. What does `recall@10 = 0.9` tell you about an ANN index?**

<details><summary>Answer</summary>

On average the approximate index returns 9 of the 10 *truly* closest vectors per query; 1 is missed because ANN inspects only a subset. It is about **set overlap** with the true top-k, not distance accuracy or similarity score. You measure it by comparing against exact/brute-force search on the same queries.

</details>

**R4. Which metric family scores the *retriever* and which scores the *generator*?**

<details><summary>Answer</summary>

Retriever: **context precision, context recall, hit rate, MRR, NDCG** (was the right evidence fetched and ranked high?). Generator: **faithfulness/groundedness and answer relevancy** (is the answer entailed by the retrieved context, and does it address the question?). Faithfulness measures consistency with the context — *not* real-world truth.

</details>

**R5. Why fuse hybrid results with Reciprocal Rank Fusion instead of adding the two scores?**

<details><summary>Answer</summary>

BM25 scores are unbounded and corpus-dependent; cosine similarities sit in ~[-1, 1] — incomparable scales, so a naive sum lets BM25 drown the cosine signal. RRF sums `1/(k + rank)` (default `k = 60`), using **rank not score**, so it's scale-free and needs no tuning. A document ranked high by *both* retrievers floats to the top.

</details>

**R6. What is the core difference between a bi-encoder and a cross-encoder?**

<details><summary>Answer</summary>

A **bi-encoder** encodes query and document *separately*, so document vectors are precomputed and indexed — fast, million-scale retrieval, but the query never attends to the document. A **cross-encoder** feeds query+document *together* through one transformer (full cross-attention), giving higher accuracy but requiring one forward pass *per pair at query time* with no precompute — so you only rerank tens–hundreds of candidates.

</details>

**R7. In the canonical RAG pipeline, which stages are index-time vs query-time?**

<details><summary>Answer</summary>

**Index-time (offline, once per corpus update):** load → chunk → embed → store. **Query-time (per request):** embed query → retrieve top-k → augment prompt → generate. Expensive work (semantic chunking, LLM metadata extraction) belongs at index-time where cost amortises; chunking is *not* a per-query step.

</details>

**R8. What is HyDE, and what does it embed?**

<details><summary>Answer</summary>

HyDE (Hypothetical Document Embeddings) has the LLM write a *fake ideal answer* to the query, then embeds **that hypothetical answer** and retrieves against it — because an answer embeds closer to real answer-chunks than a question does. It differs from plain query rewriting, which reformulates and embeds a better *question*.

</details>

---

## Applied Round (Scenario)

**A1. Users can't find documents by exact error code "E-4021" with your pure dense retriever. What do you change?**

<details><summary>Answer</summary>

Add a **lexical (BM25/sparse) leg and fuse** with RRF (hybrid search). Dense embeddings smear rare exact tokens into a semantic neighbourhood and lose them; BM25 matches the literal token. **Tradeoff named:** you accept the operational cost of maintaining a second (inverted) index and a fusion step to recover exact-match recall that dense retrieval structurally cannot provide. Swapping to a bigger embedding model would not reliably fix identifier recall.

</details>

**A2. HNSW `recall@10` is 0.82, target is 0.95, and the index is already built. First knob?**

<details><summary>Answer</summary>

Raise `hnsw.ef_search` (e.g. 40 → 100 → 200) — it enlarges the query-time candidate list, recovering more true neighbours, and needs **no rebuild**. **Tradeoff named:** higher `ef_search` buys recall at the cost of query latency, so raise it until recall clears 0.95, then trim to the smallest value still meeting the SLO to protect p95. Changing `M`/`ef_construction` would help the ceiling but forces a full rebuild.

</details>

**A3. A `WHERE tenant_id = 42` filter (matches ~2% of rows) keeps returning only 1–3 results from an HNSW index instead of top-10. Diagnose and fix.**

<details><summary>Answer</summary>

**Post-filter starvation:** HNSW returns `ef_search` (default 40) candidates first, then the 2%-selective predicate discards ~98%, leaving ~1. **Tradeoff named:** you make the search filter-aware (pre-filter) instead of post-filter, trading a little index-build/config complexity for correct result counts. Fix in pgvector: `hnsw.iterative_scan = strict_order` and/or partitioned/partial indexes per tenant. Fix in a dedicated DB (Qdrant): a payload index on `tenant_id` built *before* ingest so the graph is filter-aware.

</details>

**A4. Retrieval is fetching long, vaguely-relevant chunks and the model misses the exact fact on an API-reference corpus. Single change first?**

<details><summary>Answer</summary>

Reduce `chunk_size` and use **structure-aware splitting** so each API entry is its own chunk. One vector must summarise a whole chunk; a long multi-topic chunk yields a blurry embedding that matches weakly, so smaller entry-aligned chunks produce sharp embeddings. **Tradeoff named:** smaller match units risk losing surrounding context, so pair with **small-to-big (parent-document) retrieval** — match on the small child, return the larger parent for context. Raising `top-k` first just adds noisy long chunks.

</details>

**A5. p95 end-to-end budget is 800 ms, the LLM eats ~500 ms, and precision@5 is poor though recall@50 is 0.94. Design the retrieve/rerank stage.**

<details><summary>Answer</summary>

Keep **hybrid retrieval + RRF** for recall, set **top-k = 50**, add a small **MiniLM cross-encoder reranker** over those 50, output **top-n = 5**. Recall@50 = 0.94 means the gold doc is rarely dropped, and a distilled cross-encoder scores ~50 pairs inside the ~300 ms slice left. **Tradeoff named:** reranking buys precision at ~50–120 ms of latency, so you cap the candidate count (never rerank the corpus) and pick the smallest reranker that meets the budget.

</details>

**A6. Compliance QA assistant must never fabricate a policy clause but answer confidently when covered. What layers do you add?**

<details><summary>Answer</summary>

Layer them: (1) **retrieval score gate** — abstain if the top chunk is below threshold; (2) **grounded prompt** at temperature 0 ("answer only from context, else say not covered"); (3) **citation enforcement** with a deterministic validator dropping citations to chunk IDs not in `retrieved_contexts`; (4) a cheap **faithfulness check** (HHEM classifier, threshold ≥0.9) routing failures to abstention. **Tradeoff named:** you trade *coverage* (some false abstentions) for *safety* (near-zero fabrication) — the safe default is "I don't know." No single layer suffices: grounded prompting alone still hallucinates on a retrieval miss.

</details>

---

## Analysis / Trade-off Round

**T1. You must serve 500M fp32 vectors but the HNSW index won't fit one node's RAM and you can't add nodes yet. IVFPQ vs sharding?**

<details><summary>Answer</summary>

**IVFPQ** compresses each vector into `M` product-quantization codes (e.g. from 4·d bytes to M bytes), shrinking the index to fit one node — trading **recall** (PQ is lossy), which you mitigate with a re-rank stage on finer/original vectors (IVFPQR). **Sharding** keeps full precision but splits vectors across nodes — trading **operational cost/complexity** (fan-out/merge, per-shard-local recall), and it's ruled out by "can't add nodes." Under the single-node constraint, IVFPQ + re-ranking is the pragmatic pick. Lowering `ef_search` is a distractor — it cuts latency, not the binding RAM footprint.

</details>

**T2. Pick the #1 English-only MTEB retrieval model or a slightly lower-ranked multilingual model for an EN/ES/DE support KB. Justify.**

<details><summary>Answer</summary>

Pick the **multilingual model**. A leaderboard rank is necessary but not sufficient: an English-only model strands ES/DE text in a poorly-aligned region of its space, so cross-lingual queries fail regardless of its English score. Domain/language fit dominates a marginal benchmark edge. **What flips it:** if the corpus and queries were truly all English (or you translate everything to English up front), the higher-ranked English model wins. Always benchmark finalists on your own eval set with `mteb` rather than trusting the aggregate.

</details>

**T3. A reranker was added but precision@5 barely moved; offline recall@50 is 0.71. Explain why, and what to fix first.**

<details><summary>Answer</summary>

A reranker can only **reorder what retrieval returned** — at recall@50 = 0.71 the gold document is absent ~29% of the time, so no reranking can surface it. Fix **stage 1 first**: hybrid (BM25 + dense) fusion, better chunking/embeddings, raise top-k until recall@k ≈ 0.95. The tempting wrong move — a bigger reranker — spends latency reordering a candidate set that structurally lacks the answer. Recall is the ceiling; precision work only pays off once the right doc is *in* the candidates.

</details>

**T4. "Faithfulness is 1.0, so the answer is correct." Why is this reasoning flawed?**

<details><summary>Answer</summary>

Faithfulness only means every claim is **entailed by the retrieved context** — it deliberately ignores real-world truth. If the retrieved source is wrong or stale, a perfectly faithful answer (1.0) is still factually incorrect. Truth additionally requires **source quality** and **context recall** (the right evidence was fetched). This is why you never track faithfulness alone: you can have perfect faithfulness over an irrelevant/wrong source while the actual answer was never retrieved.

</details>

**T5. A stakeholder wants to replace working naive top-k RAG (meeting recall + faithfulness targets, p95 900 ms) with a full agentic CRAG pipeline "to be state of the art." Recommend.**

<details><summary>Answer</summary>

Recommend **against it**. CRAG's benefit — grade-and-re-search, adaptive retrieval — only pays off against a *measured* failure, and the current system already meets its targets. Adding grading + agent loops means 2–3× the LLM calls, higher and more variable latency (blowing the 900 ms p95), higher cost, and more failure surface, for **no accuracy gain** on this workload. Every advanced pattern is an extra LLM round-trip; add on evidence, always cap loops with `max_iterations`, and revisit only if evaluation surfaces a specific failure (multi-hop, answering-from-irrelevant). "State of the art" is not a requirement.

</details>

---

## Multiple-Choice Rapid Check

**MC1. Which TWO are valid, safe ways to shrink a vector index while keeping retrieval quality acceptable?**
- A. Truncate a Matryoshka-trained model's vectors to 256 dims and re-normalize.
- B. Slice the first 256 dims off an ordinary (non-MRL) 1024-dim model's vectors.
- C. Request a shorter embedding via the OpenAI `dimensions` parameter on a `-3` model.
- D. Switch the index metric from cosine to Euclidean.
- E. Store the vectors as plain text instead of floats.

<details><summary>Answer</summary>

**A and C.** Both exploit Matryoshka Representation Learning: `text-embedding-3-*` and other MRL models pack the most important information into leading dimensions, so truncating (+ re-normalizing) or requesting fewer `dimensions` shrinks storage with minimal quality loss. **B is the trap** — an ordinary model spreads meaning arbitrarily across all dimensions, so slicing destroys quality and re-normalization won't save it. D changes ranking behaviour, not size (and does nothing on normalized vectors). E breaks numeric search entirely.

</details>

**MC2. An IVFFlat index has low recall. Which single action most reliably improves it at query time?**
- A. Decrease `nprobe`
- B. Increase `nprobe`
- C. Set `nlist = 1`
- D. Build the index on an empty table then insert data

<details><summary>Answer</summary>

**B.** Raising `nprobe` scans more cells, so the true neighbour's cell is more likely included (higher recall, higher latency). **C is the tempting distractor** — one cell = effectively exact search, "so perfect recall" — but it abandons the speed gain the index exists for and isn't how you tune IVF. A lowers recall. D is the classic IVF anti-pattern: k-means trains on no data and places centroids badly — always build *after* loading representative data.

</details>

**MC3. Which statement about the retrieve-then-rerank pattern is correct?**
- A. A cross-encoder should score the entire corpus for best accuracy.
- B. Reranking can surface a gold document even if it wasn't in the retrieved top-k.
- C. Stage 1 optimises recall (wide top-k); stage 2 optimises precision (narrow top-n).
- D. More reranked chunks (top-n) sent to the LLM always improve answer quality.

<details><summary>Answer</summary>

**C.** Stage 1 (cheap, high-recall retriever) fetches a generous top-k so it doesn't drop the answer; stage 2 (slow, high-precision cross-encoder) reorders and trims to a small top-n. **A is wrong** — cross-encoding has no precompute and one forward pass per pair, so scoring the whole corpus can't meet any latency budget. **B is the key misconception** — a reranker only reorders what retrieval returned; if the gold doc isn't in top-k it can't appear. **D is wrong** — excess context adds cost, latency, and distraction; pick the smallest top-n that keeps quality flat.

</details>

**MC4. Which TWO are correct about corrective/self-RAG grading loops?**
- A. The grader adds at least one extra LLM call per retrieval, roughly increasing per-turn cost.
- B. Grading eliminates the need to bound retry iterations.
- C. Grading directly reduces answering-from-irrelevant-context, a common hallucination source.
- D. Grading improves recall by generating more query variants.
- E. Grading replaces the need for a reranker in all cases.

<details><summary>Answer</summary>

**A and C.** The grader is a separate LLM invocation per retrieval, roughly doubling per-turn cost (A); refusing to generate from graded-irrelevant docs is exactly what stops confident-but-wrong answers (C). **B is the tempting distractor and is wrong** — grading *creates* a re-search loop that must still be bounded with `max_iterations`, or it runs forever when the corpus can't answer. D describes multi-query expansion, not grading. E is false — grading judges relevance to decide re-search; a reranker still *orders* retained docs.

</details>

**MC5. Which is a genuine advantage of RAG over fine-tuning on the same corpus?**
- A. It eliminates the possibility of hallucination entirely.
- B. Knowledge can be updated by re-indexing without retraining the model.
- C. It removes the need to choose an embedding model.
- D. It guarantees lower query latency than a bare LLM call.

<details><summary>Answer</summary>

**B.** RAG updates knowledge by changing the index, not the weights — plus answers can cite the source chunk that produced them. **A is wrong** — RAG *reduces* but never *eliminates* hallucination; the model can still ignore or misread retrieved text, which is why grounding + verification + abstention exist. **C is wrong** — RAG *requires* an embedding model for chunks and queries. **D is the most tempting distractor but false** — RAG adds a retrieval step, so a single RAG call is generally *higher* latency than a bare LLM call, not lower.

</details>

---

## Self-Assessment Scorecard

| Topic area | Can I explain it cold? (✓/△/✗) | Where to review |
|---|---|---|
| Embeddings, distance metrics, MTEB, Matryoshka, asymmetric search | | `02-llm-serving-and-rag-architecture/02-embeddings-and-vector-search/01-embedding-models-and-vector-representations.md` |
| ANN indexes (HNSW/IVF/IVFPQ), recall×latency×memory, filtering, sharding | | `02-llm-serving-and-rag-architecture/02-embeddings-and-vector-search/02-vector-databases-and-similarity-search-indexes.md` |
| Hybrid search, RRF, cross-encoder reranking, ColBERT, top-k/top-n | | `02-llm-serving-and-rag-architecture/02-embeddings-and-vector-search/03-hybrid-search-and-reranking-strategies.md` |
| RAG pipeline stages + chunking (fixed/recursive/semantic, small-to-big) | | `02-llm-serving-and-rag-architecture/03-rag-pipeline-design-patterns/01-rag-architecture-fundamentals-and-chunking-strategies.md` |
| Retrieval quality metrics, faithfulness, grounded prompting, abstention | | `02-llm-serving-and-rag-architecture/03-rag-pipeline-design-patterns/02-retrieval-quality-and-zero-hallucination-design.md` |
| Advanced/agentic RAG (query transformation, routing, CRAG, bounded loops) | | `02-llm-serving-and-rag-architecture/03-rag-pipeline-design-patterns/03-advanced-rag-patterns-and-agentic-retrieval.md` |

---

## Further Reading

- [pgvector — README (HNSW & IVFFlat indexing, filtering, iterative scans, scaling)](https://github.com/pgvector/pgvector) — *verified 2026-07-29* — `m`/`ef_construction`/`ef_search`, `lists`/`probes`, post-filter behaviour, binary quantization + re-ranking, replicas/sharding.
- [FAISS — Faiss indexes (index families, IVF cell-probe, PQ, HNSW parameters)](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes) — *verified 2026-07-29* — `IndexFlat`, `IndexHNSWFlat`, `IndexIVFFlat`, `IndexIVFPQ`, the `nlist = C·√n` heuristic, and PQ `M`/`nbits`.
- [Retrieve & Re-Rank — Sentence Transformers](https://sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html) — *verified 2026-07-29* — the canonical two-stage bi-encoder-retrieve then cross-encoder-rerank pipeline.
- [Faithfulness — Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — *verified 2026-07-29* — claim-decomposition definition and formula for groundedness, plus the cheap HHEM-2.1 classifier variant for production.
- [Build a custom RAG agent with LangGraph (agentic RAG + document grading loop)](https://docs.langchain.com/oss/python/langgraph/agentic-rag) — *verified 2026-07-29* — retrieval-as-a-tool, `grade_documents` conditional edge, and rewrite/re-retrieve loop.
