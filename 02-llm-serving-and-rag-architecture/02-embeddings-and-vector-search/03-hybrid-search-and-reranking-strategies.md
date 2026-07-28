# Hybrid Search and Reranking Strategies

**Section:** LLM Serving & RAG Architecture → Embeddings & Vector Search | **Est. time:** 3 hrs | **Interview relevance:** High — "how would you improve retrieval quality?" is the single most common RAG follow-up, and hybrid+rerank is the expected senior answer.

---

## TL;DR

Dense vector search and lexical (BM25) search fail in complementary ways: dense misses exact terms, codes, and rare tokens; lexical misses paraphrase and synonymy. Hybrid search fuses both result lists — usually with Reciprocal Rank Fusion (RRF), which combines ranks rather than incomparable scores — and a two-stage *retrieve-then-rerank* pipeline then applies a slow-but-accurate cross-encoder to only the top handful of candidates to sharpen precision within a latency budget. **The one thing to remember: retrieval decides what the model *can* see (recall, cheap, wide top-k), reranking decides what it *does* see (precision, expensive, narrow top-n) — get recall first with hybrid, then buy precision with a reranker.**

---

## ELI5 — Explain It Like I'm 5

Imagine you send two scouts to find the best restaurant in a city. One scout has a phone book and finds every place whose *name* contains the word you said — great when you know the exact name, useless when you only remember "that cozy noodle spot." The other scout wanders around by *vibe* and finds places that *feel* right — great for the noodle spot, but it walks right past the place literally called "Noodle Spot" because the name never registered. Each scout hands you their own ranked shortlist, and because their lists were scored on totally different scales you can't just add the numbers — instead you reward places that *both* scouts ranked near the top. Then, and only then, a single expert judge personally visits the top ten survivors and re-ranks them carefully, because you can't afford to send the judge to all ten thousand restaurants. The common misconception is that the vibe scout (vector search) alone is always best — but it silently loses exact matches like product codes and names, which is exactly why you keep the phone-book scout around.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the complementary failure modes of lexical (BM25/sparse) vs dense vector retrieval and justify when hybrid beats either alone.
- [ ] Implement rank-based fusion (RRF) and explain why it is preferred over naive weighted score addition across incomparable score scales.
- [ ] Design a two-stage retrieve-then-rerank pipeline with concrete top-k / top-n values that fit a stated p95 latency budget.
- [ ] Compare bi-encoders, cross-encoders, and ColBERT-style late interaction on the accuracy/latency/index-cost axes.
- [ ] Decide when a reranker is worth its cost and diagnose when it is being misused (e.g. reranking the whole corpus).

---

## Visual Overview

### Two-Stage Retrieve → Fuse → Rerank Pipeline

```
Query
  │
  ├──► BM25 / sparse retriever ──► top-k (e.g. 50) ─┐
  │                                                 ├──► RRF fuse ──► top-k' (e.g. 50)
  └──► dense (ANN) retriever   ──► top-k (e.g. 50) ─┘         │
                                                              ▼
                                              Cross-encoder rerank (score all 50)
                                                              │
                                                              ▼
                                                     top-n (e.g. 5) ──► LLM context
```

### BM25 vs Dense: Complementary Hits

```
Query: "error code E-4021 in payment retry"

BM25 finds ───► docs containing literal "E-4021", "retry"   (exact-token wins)
                but MISSES ─► "transaction reattempt failure 4021"

Dense finds ──► docs about "payment reattempt / retry logic" (semantic wins)
                but MISSES ─► the exact string "E-4021" (rare token, weak in embedding)

Hybrid  ─────► keeps BOTH → the doc with the exact code AND the paraphrased doc
```

### Bi-Encoder vs Cross-Encoder Scoring

```
BI-ENCODER (retrieval — fast, precomputable)
  query ──► [encoder] ──► q-vector ─┐
                                     ├─► cosine sim ──► score
  doc   ──► [encoder] ──► d-vector ─┘   (doc vectors precomputed & indexed)

CROSS-ENCODER (rerank — slow, no precompute)
  [query + doc] together ──► [encoder] ──► relevance score
                                          (must run once PER query-doc PAIR at query time)
```

### When-to-Rerank Decision Tree

```
Is retrieval recall already high but precision@n low?
├── No  ──► fix retrieval first (hybrid, better embeddings, chunking) — reranking can't add what wasn't retrieved
└── Yes ──► Do you have a p95 latency budget for +50–300ms?
            ├── Yes ──► add cross-encoder rerank over top-k candidates
            └── No  ──► use ColBERT late-interaction or a distilled/smaller reranker, or skip
```

---

## Key Concepts

### Lexical (BM25 / sparse) vs Dense vector search

**What it is.** Lexical retrieval scores documents by term overlap with the query (BM25 is the standard bag-of-words ranking function); sparse-vector methods like SPLADE learn term weights but still operate in a high-dimensional vocabulary space. Dense retrieval encodes query and document into fixed low-dimensional embeddings and ranks by vector similarity (cosine / dot product).

**How it works mechanistically.** BM25 rewards documents where query terms are frequent (term frequency) but discounts terms that appear everywhere (inverse document frequency), with a length-normalisation term — so it excels at *exact* token matches: identifiers, error codes, product SKUs, names, acronyms. It has zero notion of meaning, so "car" and "automobile" are unrelated tokens. Dense retrieval maps text into a learned semantic manifold where paraphrases land near each other, so it captures synonymy and intent — but rare tokens and exact strings get "smeared" into the nearest semantic neighbourhood and can be lost entirely. Their failure modes are therefore complementary, which is the whole justification for hybrid.

**Where it appears in real systems.** In Elasticsearch, lexical is a `standard` retriever with a BM25 `term`/`match` query and dense is a `knn` retriever; in Weaviate, BM25F powers the keyword half of `query.hybrid(...)` and HNSW powers the vector half. Sparse learned retrieval appears as Elastic's ELSER (`sparse_vector` field + `inference_id`) or Sentence-Transformers `SparseEncoder` (SPLADE) models.

### Hybrid fusion: RRF, weighted score fusion, convex combination

**What it is.** Fusion is the step that merges two (or more) ranked result lists into one. The three common recipes are Reciprocal Rank Fusion (rank-based), weighted/relative score fusion, and convex combination (a normalised weighted sum).

**How it works mechanistically.** RRF ignores raw scores entirely and sums `1 / (k + rank)` across the lists for each document (Elastic's default rank constant `k = 60`); a document ranked #1 in both lists dominates one ranked #1 in only one list. Because it uses *rank*, not *score*, it sidesteps the fatal problem that BM25 scores (unbounded, corpus-dependent) and cosine similarities (bounded ~[-1,1]) live on incomparable scales. Weighted score fusion (Weaviate's `RelativeScoreFusion`, the default since v1.24) first min-max normalises each list's scores into a comparable range, then blends them by a weight `alpha` (`alpha=1` pure vector, `alpha=0` pure keyword). Convex combination is the general form `α·norm(dense) + (1-α)·norm(sparse)` and requires you to pick and tune both the normaliser and `α`.

**Where it appears in real systems.** Elastic exposes RRF as an `rrf` retriever with `rank_constant` and `rank_window_size`; Weaviate exposes `alpha` and `HybridFusion.RELATIVE_SCORE` / `Ranked` on `query.hybrid`. RRF's selling point per Elastic's docs is that it "requires no tuning" and beats either query alone — which is why it is the safe default in interviews.

### Cross-encoder reranking (vs bi-encoders)

**What it is.** A reranker re-scores an already-retrieved candidate set for relevance to the query. A cross-encoder is the accurate variant: it feeds the query and a document *together* through a single transformer and outputs one relevance score.

**How it works mechanistically.** A bi-encoder (used for retrieval) encodes query and document *separately*, so document vectors can be precomputed and indexed — enabling million-scale search — but the query never "sees" the document during encoding, capping accuracy. A cross-encoder applies full self-attention across the concatenated query+document, so every query token can attend to every document token; this captures fine-grained relevance that separate encoding cannot, at the cost of one full forward pass *per query-document pair at query time* with no precomputation possible. This is exactly why you can retrieve over 10M docs with a bi-encoder but can only rerank tens to low hundreds with a cross-encoder inside a latency budget.

**Where it appears in real systems.** Sentence-Transformers exposes `CrossEncoder(model).predict([(query, doc), ...])` and `.rank(query, docs, top_k=n)`; MS MARCO cross-encoders (e.g. `cross-encoder/ms-marco-MiniLM-L6-v2`) are the standard reranker checkpoints. Managed rerank endpoints (hosted reranker APIs) wrap the same pattern behind a single call.

### The two-stage retrieve-then-rerank pattern

**What it is.** A pipeline that uses a cheap, high-recall retriever to fetch a wide candidate set (top-k), then a slow, high-precision reranker to reorder and trim to a narrow final set (top-n) sent to the LLM.

**How it works mechanistically.** Stage 1 optimises *recall* — it must not drop the right document, so k is generous (25–100). Stage 2 optimises *precision* — the cross-encoder promotes truly relevant candidates and demotes lexical/semantic false positives, and you keep only top-n (3–10). The crucial constraint is that **reranking can only reorder what retrieval returned** — if the gold document is not in the top-k, no reranker can rescue it. Hence recall (hybrid, chunking, embedding quality) must be fixed *before* reranking is worth adding.

**Where it appears in real systems.** Sentence-Transformers documents this exact "Retrieve & Re-Rank" pipeline: a bi-encoder retrieves, a cross-encoder re-ranks. In a LangChain RAG chain it is a base retriever wrapped by a `ContextualCompressionRetriever` / reranker; in Weaviate it is `query.near_text(...).rerank(...)`.

### ColBERT / late interaction (the middle ground)

**What it is.** Late interaction stores a *per-token* embedding for every document token (a multi-vector representation) instead of one pooled vector, and scores relevance by a cheap token-level operator at query time.

**How it works mechanistically.** ColBERT encodes documents into token-level vectors offline (like a bi-encoder, so it's precomputable and indexable), then at query time computes, for each query token, its maximum similarity against all document tokens and sums these ("MaxSim"). This recovers much of the token-level interaction a cross-encoder gets — better than a single pooled vector — without the query-time full transformer pass over each pair, so it sits between bi-encoders (fast, less accurate) and cross-encoders (slow, most accurate). The cost moves into storage: many vectors per document inflate the index dramatically.

**Where it appears in real systems.** ColBERT-style multi-vector fields appear in vector DBs supporting "multi-vector" / late-interaction indexes; the operational tell is a config for token-level vectors and a MaxSim scorer rather than a single dense field.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Retrieval `top-k` (candidate count per retriever / fused) | How many candidates enter the reranker | Set k so gold-doc recall@k ≥ ~0.95 on your eval set; typically 25–100. Raise if recall is low, lower if rerank latency blows the budget. |
| Rerank `top-n` (final results) | How many docs reach the LLM | Set to the smallest n that keeps answer quality flat; usually 3–8. More context ≠ better — it adds cost and dilution. |
| RRF `rank_constant` (`k`, default 60) | How much lower-ranked docs influence the fused score | Keep the default 60 unless tuning; *lower* k (e.g. 20) sharply favours top-ranked hits, *higher* k flattens the contribution across ranks. |
| RRF `rank_window_size` | Size of each per-retriever list fused (must ≥ final `size`) | Set ≥ your retrieval top-k; raise for better relevance at a performance cost, but it also bounds pagination consistency. |
| Fusion weight `alpha` (convex/relative score) | Balance between vector (α→1) and keyword (α→0) | Start at 0.5; move toward keyword for identifier/code-heavy queries, toward vector for natural-language/paraphrase queries. |
| Reranker model choice | Accuracy vs latency of stage 2 | Use a small distilled cross-encoder (e.g. MiniLM-L6) under tight budgets; larger/multilingual models only when eval shows the gain justifies the added ms. |

### Worked Example: Requirement → Decision

**Given:** A customer-support RAG assistant over 2M help-centre chunks. Users report the model "answers a related-but-wrong article." Retrieval recall@50 is already 0.94, but precision@5 is poor. Product requires p95 end-to-end ≤ 800ms; the LLM call already consumes ~500ms, leaving ~300ms for retrieval+rerank.

- **Step 1 — Identify the goal.** Lift precision@5 (put the *right* article in the top handful) without breaking the 800ms p95, and without losing exact-match support-ticket IDs.
- **Step 2 — Define inputs.** User query (natural language, sometimes containing an error code); a hybrid retriever (BM25 + dense over HNSW); an offline eval set with labelled gold chunks.
- **Step 3 — Define outputs.** Top-n = 5 reranked chunks passed as LLM context, ordered by cross-encoder relevance.
- **Step 4 — Apply constraints.** ~300ms budget for retrieve+rerank ⇒ cross-encoder can only score a bounded candidate set. Benchmark shows a MiniLM-L6 cross-encoder scores ~50 pairs in ~60–120ms on the target GPU. Recall@50 = 0.94 means top-k=50 rarely drops the gold doc.
- **Step 5 — Select the approach.** Keep hybrid retrieval with **RRF (rank_constant 60)** for recall, set **top-k = 50**, add a **MiniLM-L6 cross-encoder reranker** over those 50, output **top-n = 5**. Rationale vs alternatives: pure dense retrieval loses the exact error-code tickets (fails the identifier requirement); reranking the full corpus is infeasible (2M forward passes ≫ budget); a larger reranker would exceed the 300ms slice; ColBERT would help but requires re-indexing 2M docs into multi-vector storage — not justified when a cross-encoder over 50 candidates already meets the target.

---

## Implementation

```python
# Scenario: A code-search RAG keeps missing exact function names because pure dense
# retrieval smears rare identifiers. Combine BM25 (exact tokens) with dense (semantics)
# and fuse by RANK, so incomparable score scales never get compared directly.
def reciprocal_rank_fusion(result_lists, k=60, top_n=50):
    """result_lists: list of ranked lists of doc_ids (best first), one per retriever."""
    scores = {}
    for ranked in result_lists:
        for rank, doc_id in enumerate(ranked, start=1):   # rank is 1-based
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    fused = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)
    return [doc_id for doc_id, _ in fused[:top_n]]

bm25_hits = ["E4021", "retry-logic", "payments"]      # exact-token winners
dense_hits = ["retry-logic", "reattempt-guide", "E4021"]  # semantic winners
candidates = reciprocal_rank_fusion([bm25_hits, dense_hits], k=60, top_n=50)
# Docs ranked well by BOTH retrievers (E4021, retry-logic) float to the top.
```

```python
# Anti-pattern: naively ADD a BM25 score and a cosine similarity, then also run the
# cross-encoder over the ENTIRE corpus. Two independent failures in one snippet.
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L6-v2")

def broken(query, all_docs):                       # all_docs = the WHOLE corpus
    fused = {d.id: d.bm25 + d.cosine for d in all_docs}   # BM25 ~[0, 30+], cosine ~[-1, 1]
    # BM25's larger magnitude silently dominates; cosine barely matters.
    pairs = [(query, d.text) for d in all_docs]    # millions of query-doc pairs
    scores = reranker.predict(pairs)               # one forward pass PER pair -> minutes/OOM
    return scores

# Correct approach: fuse by RANK (scale-free), then rerank ONLY the top-k candidates.
def fixed(query, retrieve_hybrid, corpus, k=50, n=5):
    candidate_ids = retrieve_hybrid(query, top_k=k)          # RRF over BM25 + dense
    docs = [corpus[i] for i in candidate_ids]                # just k of them
    ranked = reranker.rank(query, [d.text for d in docs], top_k=n)  # k forward passes, bounded
    return [docs[r["corpus_id"]] for r in ranked]
# What breaks in the anti-pattern: (1) adding incomparable scales lets BM25 drown cosine;
# (2) cross-encoding the full corpus has no precompute and cannot meet any latency budget.
```

---

## Common Pitfalls & Misconceptions

- **"Vector search alone is always best."** — Beginners assume semantic embeddings subsume keyword search. Dense retrieval systematically loses exact identifiers, codes, names, and rare tokens; hybrid keeps a BM25 leg precisely to recover those exact-match cases.
- **Adding raw scores across retrievers.** — It looks natural to sum a BM25 score and a cosine similarity. Their scales are incomparable (BM25 unbounded and corpus-dependent, cosine ~[-1,1]), so the larger-magnitude signal dominates; use rank-based RRF or min-max-normalised relative-score fusion instead.
- **Expecting a reranker to fix bad recall.** — People bolt on a cross-encoder hoping it "finds better docs." A reranker only *reorders the candidates retrieval already returned*; if the gold doc isn't in the top-k it cannot appear, so fix recall (hybrid, chunking, embeddings) first.
- **Cross-encoding too many candidates.** — Because a cross-encoder often gives the best relevance, it's tempting to feed it hundreds of docs. Its cost is a full forward pass per pair with no precompute, so latency scales linearly with k — cap k to what the p95 budget allows and lean on stage-1 recall.
- **Treating top-n as "bigger is better."** — Beginners pad the LLM with 20 reranked chunks assuming more context helps. Excess context adds cost, latency, and distraction; choose the smallest top-n that keeps answer quality flat.

---

## Key Definitions

| Term | Definition |
|---|---|
| BM25 | A bag-of-words lexical ranking function scoring term-frequency × inverse-document-frequency with length normalisation; strong on exact-token matches. |
| Sparse retrieval | Retrieval over high-dimensional vocabulary-space vectors with learned or statistical term weights (e.g. SPLADE, ELSER); a learned superset of keyword search. |
| Dense retrieval | Retrieval by similarity between low-dimensional learned embeddings of query and document; captures semantics/paraphrase. |
| Hybrid search | Combining lexical/sparse and dense result lists into one ranking via a fusion method. |
| RRF | Reciprocal Rank Fusion: fuse lists by summing `1/(k+rank)` per document; rank-based, tuning-free, scale-agnostic. |
| Relative score fusion | Fusion that min-max normalises each retriever's scores then blends by weight `alpha`. |
| Bi-encoder | Encodes query and document separately; enables precomputed, indexable vectors for fast large-scale retrieval. |
| Cross-encoder | Encodes query+document jointly through one transformer for a relevance score; most accurate, no precompute, used for reranking a small candidate set. |
| Late interaction (ColBERT) | Stores per-token document vectors and scores via query-token MaxSim; a precomputable middle ground between bi- and cross-encoders. |
| top-k / top-n | Candidate count returned by retrieval (k, wide, recall) vs final count kept after rerank (n, narrow, precision). |

---

## Summary / Quick Recall

- Lexical (BM25) and dense search fail in **complementary** ways → hybrid beats either alone.
- Fuse by **rank (RRF, k=60 default)**, not by adding incomparable raw scores.
- **Retrieve-then-rerank**: cheap wide top-k for recall, expensive narrow top-n for precision.
- **Cross-encoders** are accurate but run one forward pass per query-doc pair at query time → only rerank tens–hundreds, never the corpus.
- A reranker can only reorder what retrieval returned — **fix recall before adding rerank**.
- **ColBERT / late interaction** = precomputable per-token vectors, accuracy between bi- and cross-encoders, at higher index cost.
- Tune `top-k` for recall@k ≥ ~0.95 and pick the smallest `top-n` that keeps answers flat.

---

## Self-Check Questions

1. What is the core mechanical reason a cross-encoder is more accurate but far slower than a bi-encoder?

   <details><summary>Answer</summary>

   A cross-encoder feeds the query and document *together* through one transformer, so full self-attention lets every query token attend to every document token — capturing fine-grained relevance — but this requires a forward pass **per query-document pair at query time** with no precomputation. A bi-encoder encodes each side separately, so document vectors are precomputed/indexed (fast, scalable) but the query never attends to the document during encoding, capping accuracy. The tempting wrong answer — "the cross-encoder just has more parameters" — misses that the real cost is the *joint, per-pair, query-time* computation, not model size.

   </details>

2. Your RAG system retrieves with pure dense vectors and users complain it can't find documents by their exact error code (e.g. "E-4021"). What change do you make and why?

   <details><summary>Answer</summary>

   Add a lexical (BM25/sparse) retriever and fuse the two lists (hybrid search), because dense embeddings smear rare exact tokens like "E-4021" into a semantic neighbourhood and lose them, whereas BM25 matches the literal token. Simply swapping to a bigger embedding model (the tempting distractor) doesn't reliably fix exact-identifier recall — the failure is structural to pooled dense representations, which is exactly what the keyword leg is there to cover.

   </details>

3. **Which TWO** of the following are valid reasons to prefer Reciprocal Rank Fusion over naively adding a BM25 score to a cosine similarity?
   - A. RRF requires no tuning and works even when the two relevance signals are unrelated.
   - B. RRF uses ranks, avoiding the incomparable-scale problem between unbounded BM25 and bounded cosine scores.
   - C. RRF guarantees the cross-encoder can be skipped entirely.
   - D. RRF always returns more documents than either retriever alone.
   - E. RRF converts embeddings into keywords so one index suffices.

   <details><summary>Answer</summary>

   **A and B.** Per Elastic's docs, RRF requires no tuning and combines result sets with different (even unrelated) relevance indicators (A); and because it sums `1/(k+rank)` it operates on rank, sidestepping the fatal mismatch between BM25's unbounded, corpus-dependent scores and cosine's ~[-1,1] range (B). C is wrong — fusion improves recall/ranking but is orthogonal to whether you add a reranker for precision. D is false — fusion produces one merged, truncated list, not necessarily more docs. E is nonsense — RRF fuses two separate result lists; it doesn't merge indexes or convert representations.

   </details>

4. You add a cross-encoder reranker and precision@5 barely improves, yet your offline recall@50 is only 0.71. What is the most likely diagnosis and fix?

   <details><summary>Answer</summary>

   The reranker can only reorder candidates that retrieval returned, and with recall@50 = 0.71 the gold document is absent from the candidate set ~29% of the time — so no reranking can surface it. Fix stage 1 first: add hybrid (BM25 + dense) fusion, improve chunking/embeddings, and/or raise top-k until recall@k is high (~0.95). The tempting wrong move — swapping in a bigger reranker — spends latency to reorder a candidate set that structurally lacks the right document.

   </details>

5. Under a strict p95 latency budget where a cross-encoder over your desired candidate count would be too slow, but a single pooled dense vector is losing token-level nuance, which approach best trades off accuracy, latency, and index cost, and what is the catch?

   <details><summary>Answer</summary>

   ColBERT-style **late interaction**: it precomputes per-token document vectors (indexable like a bi-encoder, so no per-pair query-time transformer pass) and scores via query-token MaxSim, recovering much of the token-level accuracy a single pooled vector loses — placing it between bi- and cross-encoders on accuracy/latency. The catch is **index cost**: storing many vectors per document inflates storage and memory substantially. Choosing a bigger single-vector embedding (the distractor) doesn't recover true token-level interaction, and cross-encoding is what the budget already ruled out.

   </details>

---

## Further Reading

- [Cross-Encoders — Sentence Transformers](https://www.sbert.net/examples/applications/cross-encoder/README.html) — *verified 2026-07-29* — bi-encoder vs cross-encoder mechanics and when to use each.
- [Retrieve & Re-Rank — Sentence Transformers](https://sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html) — *verified 2026-07-29* — the canonical two-stage bi-encoder-retrieve then cross-encoder-rerank pipeline.
- [Reciprocal rank fusion — Elasticsearch Reference](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion) — *verified 2026-07-29* — RRF formula, `rank_constant` (default 60) and `rank_window_size`, worked scoring example.
- [Hybrid search — Weaviate Documentation](https://weaviate.io/developers/weaviate/search/hybrid) — *verified 2026-07-29* — `alpha` weighting and `RelativeScoreFusion` vs `Ranked` fusion for combining BM25F and vector search.
