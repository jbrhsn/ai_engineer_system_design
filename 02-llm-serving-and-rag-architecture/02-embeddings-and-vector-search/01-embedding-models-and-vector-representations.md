# Embedding Models and Vector Representations

**Section:** LLM Serving & RAG Architecture → Embeddings & Vector Search | **Est. time:** 2.5 hrs | **Interview relevance:** High — the embedding model is the single decision that silently caps a RAG system's retrieval ceiling; interviewers probe it to see if you understand distance metrics, model selection, and re-embedding cost.

---

## TL;DR

An embedding model is a transformer encoder that maps text to a fixed-length vector so that semantically similar text lands close together in a high-dimensional space, letting you find "meaning-similar" passages by nearest-neighbour search instead of keyword matching. The design decisions that matter in an interview are: the distance metric (and whether vectors are normalized), the dimensionality (fixed vs Matryoshka-truncatable), symmetric vs asymmetric search, and choosing the model via a benchmark like MTEB against your actual domain, language, and max-sequence-length needs. The trap that sinks production systems is swapping the embedding model without re-embedding the whole corpus — queries and documents encoded by different models live in incompatible spaces and retrieval silently collapses. **The one thing to remember: an embedding is a coordinate for meaning, and every vector in one index must come from the exact same model configuration or the geometry is meaningless.**

---

## ELI5 — Explain It Like I'm 5

Imagine a giant map of a city where every shop is placed not by its street address but by what it *sells* — all the bakeries cluster in one neighbourhood, all the shoe shops in another, and a shop that sells both pastries and coffee sits between the bakery district and the café district. An embedding model is the mapmaker: you hand it a description of a shop and it hands back the map coordinates for that shop, and shops with similar meanings always land near each other. When you want "somewhere to buy a birthday cake," you don't scan every shop name for the word "cake" — you go to the coordinates for "sweet celebration food" and grab whatever is nearby. The common misconception is that embeddings are just fancy keyword lists; they are not — the coordinates capture meaning, so "car" and "automobile" land on nearly the same spot even though they share no letters, and a keyword search would miss that entirely. The catch: if two mapmakers draw the same city with different conventions, the coordinate "5th and Main" points to different places on each map — so a query drawn on one map can never be trusted to find shops placed on the other.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain how a transformer encoder turns text into a dense vector and contrast dense vs sparse representations.
- [ ] Choose the correct similarity/distance metric (cosine, dot product, Euclidean) given whether vectors are normalized and what "similar" should mean.
- [ ] Select an embedding model using MTEB plus domain fit, multilingual coverage, dimensionality, and max sequence length under a latency/cost budget.
- [ ] Design asymmetric (query≠document) vs symmetric search and apply Matryoshka truncation to trade accuracy for storage.
- [ ] Diagnose why swapping an embedding model without re-embedding the corpus destroys retrieval, and design the migration correctly.

---

## Visual Overview

### Text → Encoder → Vector

```
"How do I reset my password?"
          │
          ▼
   ┌──────────────┐    tokenize
   │  Tokenizer   │  ──► [CLS] how do i reset ... [SEP]
   └──────────────┘
          │ token ids
          ▼
   ┌──────────────┐
   │ Transformer  │  self-attention over all tokens
   │   Encoder    │  ──► one contextual vector per token
   └──────────────┘
          │ token vectors
          ▼
   ┌──────────────┐
   │   Pooling    │  mean / CLS pooling ──► single vector
   └──────────────┘
          │
          ▼
   [0.12, -0.04, 0.88, ... ]   ← fixed-length dense embedding (e.g. 1024 floats)
```

### Points in a 2-D Semantic Space (cosine = angle)

```
        ▲ dim 2
        │        • "automobile"
        │       /  small angle θ ──► high cosine similarity
        │      / θ
        │     •  "car"
        │
        │                         • "banana"
        │                              (large angle ──► low similarity)
        └───────────────────────────────────────► dim 1

Cosine looks at the ANGLE between vectors, not their length.
```

### Symmetric vs Asymmetric Search

```
SYMMETRIC (query and corpus are the same kind of text)
  "duplicate question?"  ──►[ same model ]──► compare ──► "similar question"
  Use case: paraphrase / STS / dedup. One encoder, one prompt.

ASYMMETRIC (short query ↔ long document)
  short query   ──►[ model + query prompt   ]──►  q-vector ┐
                                                            ├──► compare
  long passage  ──►[ model + document prompt ]──►  d-vector ┘
  Use case: RAG retrieval. Model trained so a QUESTION lands
  near its ANSWER passage, even though they read very differently.
```

### Model-Swap Failure (before / after)

```
CORRECT — one model for everything
  corpus ──►[ model A ]──► index (A-space)
  query  ──►[ model A ]──► search (A-space)   ✓ same geometry

BROKEN — swapped query model, kept old index
  corpus ──►[ model A ]──► index (A-space)     (never re-embedded)
  query  ──►[ model B ]──► search (B-space)   ✗ different geometry
                                               ──► near-random results
```

---

## Key Concepts

### Embedding Generation (transformer encoder + pooling)

**What it is:** The process of turning a variable-length string into a single fixed-length vector of floats that encodes its meaning.

**How it works mechanistically:** The text is tokenized into sub-word tokens, which are passed through a transformer encoder (typically a BERT-family or LLM-derived encoder). Self-attention lets each token's output vector absorb context from every other token, so "bank" in "river bank" and "bank" in "savings bank" get different vectors. The per-token output vectors are then collapsed into one sentence vector by a pooling step — usually mean pooling over all token vectors, or taking the `[CLS]` token's vector. The model is trained with a contrastive objective (e.g. `MultipleNegativesRankingLoss`) so that vectors of related texts are pulled together and unrelated ones pushed apart.

**Where it appears in real systems:** `SentenceTransformer("model-name").encode(texts)` in Sentence-Transformers, or `client.embeddings.create(model="text-embedding-3-small", input=text)` in the OpenAI API. The pooling strategy is a configured module in the model's `modules.json`; you rarely set it yourself but it determines whether the model even produces usable sentence vectors.

### Dense vs Sparse Representations

**What it is:** Dense embeddings are low-dimensional vectors (a few hundred to a few thousand floats) where every dimension carries some signal; sparse representations are very high-dimensional vectors (vocabulary-sized) that are mostly zeros, with non-zero weights on specific terms.

**How it works mechanistically:** A dense model distributes meaning across all dimensions — no single dimension means "the word cake." A sparse model (classic BM25, or learned sparse like SPLADE) assigns weight to actual vocabulary terms, so it captures exact-term/lexical matches. Dense captures semantic similarity ("car"≈"automobile") but can miss rare exact tokens (a product SKU, a legal citation); sparse nails exact matches but misses paraphrase. This is why hybrid search combines both.

**Where it appears in real systems:** Dense vectors go into an ANN index (HNSW/IVF) in a vector DB; sparse vectors go into an inverted index. Sentence-Transformers exposes both `SentenceTransformer` (dense) and `SparseEncoder` (SPLADE). This chapter focuses on dense; hybrid combination is covered in `03-hybrid-search-and-reranking-strategies.md`.

### Distance / Similarity Metrics (cosine vs dot product vs Euclidean)

**What it is:** The function that turns two vectors into a single "how close?" number, which drives the nearest-neighbour ranking.

**How it works mechanistically:**
- **Cosine similarity** measures the *angle* between vectors, ignoring magnitude — it answers "do these point the same direction in meaning-space?" Range −1 to 1, higher = more similar.
- **Dot product** multiplies angle *and* magnitude, so longer vectors score higher; it rewards both direction and "confidence/length."
- **Euclidean (L2) distance** measures straight-line distance between the tips of the vectors; lower = more similar.

The key identity: **if all vectors are L2-normalized (length 1), cosine similarity, dot product, and Euclidean-distance ranking are equivalent** — cosine reduces to a dot product, and Euclidean produces the identical ordering. OpenAI embeddings are returned pre-normalized, so cosine and dot product give the same rankings and dot product is just faster.

**Where it appears in real systems:** It is the index's `metric`/`distance` field — e.g. `metric="cosine"` or `metric="dot"` or `metric="euclidean"` when you create a collection in Qdrant/Pinecone/pgvector. Choosing the wrong one relative to how the model was trained is a top production bug (see Implementation anti-pattern).

### Dimensionality and Matryoshka / Truncatable Embeddings

**What it is:** Dimensionality is the vector length (e.g. 384, 768, 1024, 1536, 3072). Matryoshka Representation Learning (MRL) trains a model so that the *prefix* of the vector (e.g. first 256 of 1536) is itself a usable, high-quality embedding.

**How it works mechanistically:** A normal embedding must be used at its full trained length — chopping it destroys quality because meaning is spread arbitrarily across all dimensions. An MRL-trained model uses a nested loss that forces the most important information into the earliest dimensions, so you can truncate `vector[:256]` and still get strong retrieval, then re-normalize. This lets one stored model serve multiple accuracy/cost tiers. OpenAI's `text-embedding-3-*` were trained this way: a `text-embedding-3-large` truncated to 256 dims still outperforms the older 1536-dim `ada-002`.

**Where it appears in real systems:** The OpenAI `dimensions` API parameter (`client.embeddings.create(..., dimensions=256)`); in Sentence-Transformers, `SentenceTransformer(model, truncate_dim=256)` or slicing `embeddings[:, :256]` for a Matryoshka model. Higher dims → better recall but more storage, memory, and slower search; lower dims → cheaper index, some accuracy loss.

### Model Selection via MTEB, Domain, Language, Sequence Length

**What it is:** The decision procedure for picking which embedding model to deploy.

**How it works mechanistically:** MTEB (Massive Text Embedding Benchmark) scores models across dozens of tasks (retrieval, classification, clustering, STS) and many languages; its retrieval subset is the number that predicts RAG quality. But a leaderboard rank is *necessary, not sufficient* — you also weigh (1) **domain fit**: a general model may underperform a domain-tuned one on legal/medical/code text; (2) **multilingual coverage**: pick a multilingual model if queries and docs span languages, or the two languages won't align in vector space; (3) **max sequence length**: text longer than the model's max token limit is silently truncated, so your chunk size must fit; (4) **cost/latency**: dimensionality and whether it's hosted-API vs self-hosted.

**Where it appears in real systems:** The MTEB leaderboard (Hugging Face Space) and the `mteb` Python package (`mteb.evaluate(model, tasks=...)`) let you re-run the benchmark on *your* tasks. In code, model choice is the string in `SentenceTransformer("...")` or the `model` field in the embeddings API call. Max sequence length surfaces as `model.max_seq_length`.

### Asymmetric vs Symmetric Search

**What it is:** Whether the two things you compare are the same *kind* of text (symmetric) or different kinds — typically a short query vs a long passage (asymmetric).

**How it works mechanistically:** In symmetric search (duplicate-question detection, STS, clustering) both sides are similar-length, similarly-phrased text, so one encoding path works. In asymmetric search (RAG retrieval), a short question like "how do I reset my password?" must match a long help article that never repeats the question's wording. Models built for asymmetric search are trained on (query, relevant-passage) pairs and often expect a **prompt/instruction prefix** that tells the model which role the text plays (e.g. a `query:` prefix vs a `passage:` prefix), producing vectors placed so questions land near their answers.

**Where it appears in real systems:** Sentence-Transformers documents this explicitly ("Symmetric vs. Asymmetric Semantic Search") and supports per-role prompts via the `prompt_name`/`prompt` argument to `encode()`, or a `Router` module. Forgetting the query prefix on a model that expects one quietly degrades recall.

### Fine-Tuning / Adapting Embeddings

**What it is:** Continuing to train (or lightly adapting) a base embedding model on your own (query, positive, negative) data to specialize it to your domain.

**How it works mechanistically:** You feed pairs/triples of your domain text and optimize a contrastive loss (e.g. `MultipleNegativesRankingLoss`) so your domain's relevant pairs move closer. Options range from full fine-tuning to parameter-efficient PEFT/LoRA adapters, and unsupervised domain adaptation (e.g. GPL / generative pseudo-labeling) when you have no labels. This can substantially lift retrieval on jargon-heavy corpora a general model handles poorly.

**Where it appears in real systems:** `SentenceTransformerTrainer` with a loss and a dataset of pairs; Sentence-Transformers' training overview and PEFT-adapter docs. Note the operational cost: a fine-tuned model is a *new model*, so the entire corpus must be re-embedded with it before it's used (see the model-swap pitfall).

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Embedding model | Which encoder produces every vector | Pick the top MTEB *retrieval* performer that (a) covers your languages, (b) fits your domain, (c) has a max seq length ≥ your chunk size; benchmark 2–3 finalists on your own eval set with `mteb`. |
| Embedding dimension | Vector length → recall vs storage/latency | Start at the model's native dim; if index size or query latency is the constraint and the model is Matryoshka-trained, truncate to 256/512 and re-check recall — do not truncate a non-MRL model. |
| Distance metric | How similarity is scored/ranked | Use **cosine** unless the model docs say otherwise; if vectors are L2-normalized, switch to **dot product** for speed (identical ranking). Never use Euclidean on unnormalized vectors expecting cosine behaviour. |
| Normalization | Whether vectors are scaled to length 1 | Normalize when the index metric is cosine/dot and you want magnitude ignored; keep OpenAI vectors as-is (already normalized). If you truncate a Matryoshka vector, re-normalize afterward. |
| Max input tokens (`max_seq_length`) | Longest text encoded before truncation | Set your chunk size below this limit; if docs exceed it, chunk first — silent truncation drops the tail of long passages. |
| Query/document prompt | Role prefix for asymmetric models | For asymmetric models that define prompts, always pass the correct `query` vs `passage`/`document` prompt on each side; omit only for symmetric models that define none. |
| Batch size (`encode(batch_size=...)`) | Throughput vs memory during (re-)embedding | Raise until GPU/host memory is ~80% used to embed a large corpus fast; lower it if you hit OOM during a full re-embed. |

### Worked Example: Requirement → Decision

**Given:** You are building a customer-support RAG assistant for a SaaS product. The knowledge base is ~120k help-centre articles in English, Spanish, and German. Users type short natural-language questions in any of those languages. Budget: p95 retrieval latency < 150 ms, self-hosted (no per-token API bill), and the vector index must fit comfortably on a single node.

**Step 1 — Identify the goal:** Retrieve the top-k help articles whose *meaning* answers a short multilingual question — this is asymmetric, cross-lingual, retrieval-optimized semantic search.

**Step 2 — Define inputs:** Short user queries (often < 30 tokens) in EN/ES/DE; a corpus of help articles chunked to ~400 tokens each; a labelled eval set of ~200 (query, correct-article) pairs sampled from real tickets.

**Step 3 — Define outputs:** For each query, a ranked list of chunk vectors' nearest neighbours, feeding the downstream reranker + generator.

**Step 4 — Apply constraints:** Must be multilingual (a monolingual English model would strand ES/DE queries in a misaligned region of space); must be asymmetric-capable; must self-host within the latency and index-size budget; chunk size (~400 tokens) must be ≤ the model's `max_seq_length`.

**Step 5 — Select the approach:** Shortlist 2–3 top MTEB *multilingual retrieval* models, then run `mteb` on the 200-pair eval set and measure recall@k + latency. Choose the best-recall model whose native dimension keeps the 120k×dim index within node memory; if it's Matryoshka-trained and the index is too big, truncate to 512 dims and confirm recall barely drops. Use cosine similarity with normalized vectors (dot product at query time for speed), and apply the model's `query`/`passage` prompts. Rationale vs alternatives: a general-English top-ranked model would beat it on the English MTEB score but fail ES/DE alignment; a hosted API would simplify ops but violates the "no per-token bill / self-hosted" constraint and adds network latency against the 150 ms budget.

---

## Implementation

```python
# Scenario: Build a small semantic search index for help articles and query it.
# Constraint: query and document must be encoded by the SAME model config, and we
# rank with cosine similarity, so we let the library normalize + score for us.
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")  # one model for index AND query

corpus = [
    "To reset your password, open Settings and click 'Reset password'.",
    "Billing invoices are emailed on the first of each month.",
    "Enable two-factor authentication under Security settings.",
]
corpus_emb = model.encode(corpus, normalize_embeddings=True)   # index vectors

query = "how do I change my login password?"
query_emb = model.encode(query, normalize_embeddings=True)     # same model

# cosine == dot product because both sides are normalized
scores = query_emb @ corpus_emb.T
best = scores.argmax()
print(corpus[best])   # ──► the password-reset article
```

```python
# Anti-pattern: created a COSINE index but stored UNNORMALIZED vectors, and later
# "upgraded" the query model without re-embedding the corpus. Both break retrieval.
index = VectorIndex(metric="cosine")
corpus_emb = SentenceTransformer("all-MiniLM-L6-v2").encode(corpus)   # NOT normalized
index.add(corpus_emb)

# months later, someone swaps only the query model to a "better" one:
query_emb = SentenceTransformer("all-mpnet-base-v2").encode(query)    # DIFFERENT model
results = index.search(query_emb)   # ✗ near-random: different space + metric mismatch

# Correct approach:
# 1) Normalize consistently so the cosine index actually gets unit vectors.
# 2) Use ONE model for corpus and query; to change models, RE-EMBED the whole corpus.
MODEL = "all-mpnet-base-v2"                       # single source of truth
encoder = SentenceTransformer(MODEL)
corpus_emb = encoder.encode(corpus, normalize_embeddings=True)   # re-embed everything
index = VectorIndex(metric="cosine")
index.add(corpus_emb)
query_emb = encoder.encode(query, normalize_embeddings=True)     # same MODEL
results = index.search(query_emb)   # ✓ same geometry, correct metric
```

**What breaks in the anti-pattern:** (1) A cosine index expects magnitude to be ignored; feeding unnormalized vectors distorts scores. (2) Encoding the corpus with `MiniLM` and the query with `mpnet` places them in two unrelated coordinate systems — the nearest neighbours are effectively random. The fix is one model everywhere, consistent normalization, and a full corpus re-embed on any model change.

---

## Common Pitfalls & Misconceptions

- **Changing the embedding model without re-embedding the corpus** — teams treat the embedding model like a swappable API key and update only the query path (or upgrade the model and reuse the old index). Each model defines its own vector space, so query and document vectors must come from the *identical* model+config; any model change requires re-embedding the entire corpus and rebuilding the index before serving.
- **"Embeddings are just keyword vectors"** — beginners assume a dimension corresponds to a word, so they expect exact-term behaviour. Dense embeddings distribute meaning across all dimensions and capture paraphrase/synonymy; exact-token matching is the job of sparse/lexical search, which is why hybrid search exists.
- **Wrong distance metric for the model** — people default to Euclidean or dot product without checking how the model was trained or whether vectors are normalized. Match the metric the model was trained/documented for; cosine is the safe default, and dot product only equals cosine when vectors are L2-normalized.
- **Truncating a non-Matryoshka embedding to save space** — since MRL models advertise truncation, people assume any embedding can be sliced. Only MRL-trained models pack importance into early dimensions; slicing an ordinary embedding discards meaning arbitrarily and re-normalization won't save it.
- **Ignoring max sequence length** — beginners send full documents and assume the model reads all of it. Text beyond `max_seq_length` is silently truncated, so long chunks lose their tail; chunk to fit the model's limit before encoding.
- **Skipping the query/passage prompt on asymmetric models** — the model looks like it "just works" without the prefix, so people omit it. Models trained with role prompts expect them; dropping the `query:`/`passage:` prefix quietly lowers recall.

---

## Key Definitions

| Term | Definition |
|---|---|
| Embedding | A fixed-length vector of floats produced by an encoder that positions text by meaning so similar text is nearby. |
| Dense representation | A low-dimensional vector where meaning is spread across all dimensions (semantic similarity). |
| Sparse representation | A high-dimensional, mostly-zero vector with weights on actual vocabulary terms (lexical/exact matching). |
| Cosine similarity | Similarity based on the angle between two vectors, ignoring their magnitude; range −1 to 1. |
| Dot product | Similarity combining angle and magnitude; equals cosine ranking when vectors are L2-normalized. |
| L2 normalization | Scaling a vector to length 1 so magnitude no longer affects similarity. |
| Matryoshka embedding (MRL) | An embedding trained so leading dimensions form a usable shorter embedding, enabling truncation with minimal quality loss. |
| MTEB | Massive Text Embedding Benchmark — a multi-task, multilingual leaderboard/toolkit for comparing embedding models. |
| Symmetric search | Comparing text of the same kind/length (e.g. question↔question, dedup). |
| Asymmetric search | Comparing different kinds of text, typically short query↔long passage (RAG retrieval). |
| Max sequence length | The maximum token count a model encodes; longer input is truncated. |
| Embedding drift (model swap) | Retrieval breakdown caused by query and corpus vectors coming from different models/configs. |

---

## Summary / Quick Recall

- An embedding is a coordinate for meaning; nearest neighbours = semantically similar text.
- Dense = semantic (spread across dims); sparse = lexical (vocab terms); hybrid combines them.
- Cosine ignores magnitude; if vectors are normalized, cosine ≈ dot product ≈ Euclidean ranking — dot product is just faster.
- Pick the model by MTEB *retrieval* score **and** domain, language, and max-seq-length fit — then benchmark finalists on your own data.
- Matryoshka models let you truncate dimensions to save storage; ordinary embeddings cannot be sliced.
- RAG is asymmetric — use query/passage prompts on models that define them.
- Any model change means re-embed the entire corpus; query and corpus must share one model+config.

---

## Self-Check Questions

1. What does cosine similarity measure between two embedding vectors, and how does it differ from dot product?

   <details><summary>Answer</summary>

   Cosine similarity measures the **angle** between the two vectors, ignoring their magnitude — it answers "do they point the same direction in meaning-space?" Dot product combines angle **and** magnitude, so longer vectors score higher. The tempting wrong answer is "they're always the same": they only produce identical rankings when the vectors are L2-normalized (length 1), in which case cosine reduces to a dot product. On unnormalized vectors they can rank differently.

   </details>

2. You deploy a RAG system where users ask short questions against long documentation pages, using a model whose docs specify a `query:` and a `passage:` prompt. How should you encode each side, and what happens if you skip the prompts?

   <details><summary>Answer</summary>

   Encode queries with the `query:` prompt and documents with the `passage:` prompt (asymmetric search) — the model was trained so a question lands near its answer passage only when told each text's role. If you skip the prefixes, the vectors are placed as if the text played no special role, so questions no longer align with their answer passages and recall quietly drops. The wrong answer "prompts are optional cosmetics" fails because the prompt is part of the training contract for asymmetric models.

   </details>

3. **Which TWO** of the following are valid, safe ways to reduce the storage footprint of your vector index while keeping retrieval quality acceptable?
   - A. Truncate the vectors of a Matryoshka-trained model to 256 dims and re-normalize.
   - B. Slice the first 256 dims off an ordinary (non-MRL) 1024-dim model's vectors.
   - C. Use the OpenAI `dimensions` parameter to request a shorter embedding from a `-3` model.
   - D. Switch the index metric from cosine to Euclidean.
   - E. Store the vectors as plain text instead of floats.

   <details><summary>Answer</summary>

   **A and C.** Both exploit Matryoshka Representation Learning: the `text-embedding-3-*` models and other MRL models pack the most important information into the leading dimensions, so truncating (and re-normalizing) or requesting fewer `dimensions` keeps quality high while shrinking storage. **B** is the trap — an ordinary model spreads meaning arbitrarily across all dimensions, so slicing destroys quality. **D** changes ranking behaviour, not storage size, and on normalized vectors doesn't even change results. **E** doesn't reduce meaningful size and breaks numeric search entirely.

   </details>

4. A colleague reports that after "upgrading to a better embedding model," retrieval quality collapsed to near-random even though the model scores higher on MTEB. What is the most likely cause and the fix?

   <details><summary>Answer</summary>

   They almost certainly changed the model on the **query path only** (or reused the old index) without re-embedding the corpus. Each model defines its own vector space, so query vectors from the new model and document vectors from the old model are in incompatible geometries — nearest neighbours become meaningless regardless of MTEB score. The fix: re-embed the *entire* corpus with the new model, rebuild the index, and use that single model for both queries and documents. The tempting-but-wrong diagnosis "the new model is actually worse" is refuted by its higher benchmark score; the problem is the mismatch, not the model.

   </details>

5. You must choose between (a) the #1 English-only model on the MTEB retrieval leaderboard and (b) a slightly lower-ranked multilingual model, for a support KB with English, Spanish, and German queries and documents. Which do you pick and why, and what would change your answer?

   <details><summary>Answer</summary>

   Pick the **multilingual model (b)**. A leaderboard rank is necessary but not sufficient: an English-only model places Spanish and German text in a poorly-aligned region of its space, so cross-lingual queries fail no matter how high its English retrieval score is. Domain/language fit dominates a marginal benchmark edge. What would change the answer: if the corpus and queries were actually all English (or you translate everything to English up front), the higher-ranked English model becomes the better choice. The wrong answer "always take the #1 MTEB model" ignores that the benchmark aggregate doesn't reflect your specific language mix — which is why you benchmark finalists on your own eval set.

   </details>

---

## Further Reading

- [Vector embeddings — OpenAI API guide](https://platform.openai.com/docs/guides/embeddings) — *verified 2026-07-29* — Official guide covering how to get embeddings, model dimensions, the `dimensions` (Matryoshka) parameter, and the "which distance function should I use?" guidance.
- [Semantic Textual Similarity — Sentence Transformers](https://www.sbert.net/docs/sentence_transformer/usage/semantic_textual_similarity.html) — *verified 2026-07-29* — Official docs on similarity calculation (cosine/dot) and encoding sentence pairs.
- [Semantic Search (Symmetric vs. Asymmetric) — Sentence Transformers](https://sbert.net/examples/sentence_transformer/applications/semantic-search/README.html) — *verified 2026-07-29* — Official explanation of symmetric vs asymmetric search, prompts, and ANN integration.
- [MTEB — Massive Text Embedding Benchmark (GitHub)](https://github.com/embeddings-benchmark/mteb) — *verified 2026-07-29* — Official repo and toolkit for evaluating and selecting embedding models across tasks, languages, and modalities.
- [MTEB Leaderboard (Hugging Face Space)](https://huggingface.co/spaces/mteb/leaderboard) — *verified 2026-07-29* — The interactive leaderboard used to compare embedding models by task and language.
