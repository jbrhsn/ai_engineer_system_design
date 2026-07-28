# RAG Pipeline Design Patterns — Interview Prep

**Section:** 02 LLM Serving & RAG Architecture → RAG Pipeline Design Patterns | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| Why does chunking matter, and why is it the decision that silently caps retrieval quality? | You retrieve *chunks*, not documents — the chunk boundary *is* the retrieval unit; one vector must summarise the whole chunk, so a bad boundary poisons every future query for that content; index-time chunking is paid once but caps every downstream metric no reranker fully recovers. | "Chunking just splits text so it fits the context window." Misses that the chunk is the atomic embed/retrieve unit and that the boundary — not the size alone — drives precision. |
| Explain the chunk size vs overlap trade-off and how you'd set them. | Large chunks → one blurry vector averaging many topics (low precision, wasted tokens); tiny chunks → sharp vector but lost context (low recall of full answer); `chunk_overlap` (~10–20% of `chunk_size`) is insurance against a fact landing on a boundary; the `1000/200` defaults are a *starting point*, tuned against a retrieval eval set, not a law. | Quoting `chunk_size=1000, chunk_overlap=200` as a universal truth, or claiming "bigger chunks are always safer because the answer is definitely in there." |
| What is the difference between context precision and context recall, and why can you never track only one? | Precision = are fetched chunks relevant *and ranked* with relevant ones on top (Ragas weights Precision@k by rank); recall = did we fetch *all* the evidence needed (fraction of reference-answer claims attributable to retrieved context); a retriever can have perfect precision while missing the one chunk that answers the question — low recall silently causes hallucination. | "Precision and recall are basically the same retrieval quality number." Or optimising only precision because it's easy to eyeball the top chunks. |
| How does faithfulness/groundedness differ from answer relevancy and from correctness? | Faithfulness = every claim in the answer is entailed by the *retrieved context* (consistency with evidence, NOT real-world truth); answer relevancy = does the answer address the question; correctness = matches reality. A faithful answer over a wrong/stale source is still factually wrong; you need recall + source quality for truth. | "A high faithfulness score means the answer is correct." Conflates entailment-with-context with real-world truth. |
| How do you diagnose whether a bad answer came from a retrieval failure or a generation failure, and what fix follows? | Ask: did a top-k chunk actually contain the answer? NO → retrieval failure (low recall) → fix chunking/embeddings/top-k/hybrid search/reranking/query rewriting + score-gate abstention. YES but the answer strayed → generation failure (low faithfulness) → fix grounded prompt, citation enforcement, faithfulness check, lower temperature. | "Just tighten the prompt" for every hallucination — useless when the evidence was never retrieved; you can't ground on a chunk the model never saw. |
| When would you use CRAG / agentic RAG instead of naive top-k RAG, and when would you *not*? | Naive RAG = one fixed loop (embed → top-k → stuff → generate); add machinery only on a *measured* failure: query wording misses docs → query transformation (rewrite/HyDE/multi-query); wrong index → routing/self-querying; answers-from-irrelevant → CRAG/Self-RAG grading loop; dependent lookups → agentic/multi-hop (capped). Each pattern is an extra LLM round-trip costing latency, cost, and failure surface. | "Always use agentic RAG, it's state of the art." Ignores that naive RAG meeting targets should ship, and that every added loop must be justified and bounded. |

---

## Applied / Scenario Questions

**Q1:** You must build a customer-facing QA assistant for an insurance company that answers questions about policy terms. It must **never fabricate a policy detail** (an invented clause is legal liability — the Air Canada refund-policy precedent), but should answer confidently when the documents cover the question. How do you design it?

**Strong answer framework:**
- **Frame the goal as approach-not-guarantee:** "zero-hallucination" is a design direction reached via grounding + verification + abstention, never a certifiable spec — so the safe default is "I don't know."
- **Layer defences, because no single one is sufficient:** (1) hybrid retrieval + reranker with a **score-threshold gate** (abstain if the top chunk is below the score where precision collapses on the eval set); (2) a **grounded prompt** at temperature 0 — "answer only from context; if not covered, say so"; (3) **citation enforcement** with a *deterministic* validator that drops any answer whose `source_id` isn't in `retrieved_contexts`; (4) a **faithfulness check** (cheap HHEM classifier, threshold ≥0.9) routing failures to an abstention node.
- **Chunk for citability:** structure-aware parsing keeps clauses/tables atomic; metadata (`policy_id`, `section`, `effective_date`) powers pre-filtering, tenant isolation, and the current-version freshness requirement.
- **Tradeoff bullet (latency vs accuracy vs cost vs safety):** the deterministic citation validator is essentially free and catches the common failure (citing an un-retrieved chunk); a small-classifier faithfulness check fits a tight per-request latency budget where a large-LLM judge per request would not; you deliberately trade *coverage* (more abstentions) for *safety* (near-zero fabrication) because a wrong answer carries legal cost that a "contact an agent" fallback does not.

---

**Q2:** A stakeholder reports your production RAG system has poor answer quality — users say it "misses obvious answers" and occasionally "makes stuff up with a citation that doesn't say that." Diagnose and fix.

**Strong answer framework:**
- **Refuse to guess — instrument first.** Build/pull a golden eval set (Q → reference answer + reference chunk IDs) and split the problem on two independent axes: retrieval quality (context precision, context recall, hit rate, MRR/NDCG) and generation faithfulness.
- **Run the diagnosis tree per failing example:** did a top-k chunk contain the answer? If NO for the "misses obvious answers" class → retrieval failure (low recall) → try smaller/structure-aware chunks first (blurry embeddings from oversized chunks are the usual culprit), then hybrid search, then raise `top_k` until recall plateaus and rerank down. For the "citation doesn't say that" class → the chunk *was* retrieved but the answer strayed → generation failure → grounded prompt at temp 0, citation enforcement with a validator, faithfulness gate.
- **Don't reach for agentic machinery reflexively:** if the measured failure is answering-from-irrelevant-context, a CRAG `grade_documents` loop (capped at `max_iterations=2`) targets it directly; if it's phrasing mismatch, query transformation; match the fix to the *measured* failure.
- **Tradeoff bullet (latency vs accuracy vs cost vs safety):** raising `top_k` to 20 "so the answer is always in context" backfires — past the recall plateau it lowers precision, adds distracting text (noise sensitivity that can *increase* hallucination), and raises token cost and latency; the disciplined move is raise-until-recall-plateaus then rerank, spending compute only where the eval set proves it buys accuracy.

---

## System Design / Architecture Questions

**Q:** Design a production RAG system for enterprise document QA that returns answers **with citations** and **low hallucination**, over a large, multi-department, versioned corpus.

**Approach:**
1. **Clarify requirements (scale, latency budget, hallucination tolerance, data sensitivity).** How many docs and refresh cadence (drives whether semantic chunking's recurring ingest cost is affordable)? What p95 latency budget (drives how many verification/LLM round-trips you can afford)? What is the cost of a wrong answer vs an abstention (drives abstention thresholds)? Is the corpus multi-tenant / departmental (drives metadata pre-filtering and isolation)? Must answers be auditable (drives citation logging)?
2. **Propose high-level architecture (indexing layer, retrieval layer, guardrails).**
   - *Index-time (offline, amortised):* load → structure-aware/recursive chunking (keep tables/clauses atomic) → contextual chunk headers → embed → store in a vector index with `source`, `section`, `department`, `effective_date` metadata; consider small-to-big (parent-document) so a tight child match returns a full parent section.
   - *Query-time:* embed query → (self-querying to extract metadata filters + query router if multiple indexes) → hybrid retrieval top-k with a **score-threshold gate** → reranker to fix precision/ordering → grounded prompt (temp 0) → generate structured `{claim, source_id, quote}`.
   - *Guardrails:* deterministic citation validator → cheap faithfulness/groundedness check → conditional abstention node ("not covered in the provided documents"); optionally a bounded CRAG `grade_documents` loop (`max_iterations` = 2–3) for the answering-from-irrelevant case.
   - *Evaluation:* CI eval job over the golden set that fails the build if context recall drops below a floor or faithfulness below threshold.
3. **Justify choices and name tradeoffs explicitly (cost, latency, complexity, security).** Structure-aware chunking + small-to-big buys precision *and* context at index-time cost paid once, not per query. The score gate + citation validator are cheap deterministic filters run before any expensive LLM-judge call — layering them is what makes the fabrication rate approach zero within the latency budget. Metadata pre-filtering serves both routing accuracy and cross-tenant isolation (security). The reranker only helps when precision (ranking) is the bottleneck, not recall — so it's added on evidence. An uncapped grading/agent loop is the #1 runaway-cost bug; every loop is bounded and falls back to abstention, making worst-case cost deterministic.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:
- **Chunk overlap** — when explaining insurance against boundary-split facts; signals you know the seam matters, not just the size.
- **Parent-document / small-to-big retrieval** — when someone frames chunk size as a single dial; signals you decouple the *match* unit from the *context* unit.
- **Context precision vs context recall** — when discussing retrieval evaluation; signals you measure the retriever separately from the generator and never track one alone.
- **Faithfulness / groundedness** — when discussing hallucination; signals you distinguish entailment-with-context from real-world truth.
- **Abstention** — when discussing safety; signals you design "I don't know" as the safe default over confidently-wrong.
- **HyDE (Hypothetical Document Embeddings)** — when the question↔answer vocabulary gap is the failure; signals you know *what* gets embedded (a synthetic answer), not just "reword the query."
- **Multi-query expansion** — when recall is short due to phrasing variety; signals you fuse the union across paraphrases and know it multiplies vector-DB QPS.
- **RRF / reranking** — when precision/ordering is the bottleneck; signals you fix ranking (MRR/NDCG) rather than blindly raising top-k.
- **Corrective RAG (CRAG) / Self-RAG** — when the system answers from irrelevant context; signals you insert a grade step and *bound* the loop.
- **Query routing / self-querying** — when the right docs live in another index or behind a metadata filter; signals you route on tool *descriptions* and extract structured filters.
- **Agentic retrieval** — when retrieval should fire only when needed / multi-hop; signals you treat the retriever as a callable tool with a capped loop.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:
- **"RAG eliminates hallucination"** — RAG only *reduces* it, and only paired with grounded prompting, abstention, and verification; the model keeps its parametric memory and its bias to always answer, so it fabricates on a retrieval miss.
- **"Just stuff more docs in the prompt"** — past the recall plateau, extra chunks lower precision and add distracting text (noise sensitivity) that can *increase* hallucination and cost; retrieval selects *which* few passages appear, and more is not strictly safer.
- **"A bigger context window fixes retrieval"** — retrieval quality is set by chunking, embeddings, and ranking; a longer window doesn't make the right chunk get *fetched* or *ranked first*, and dumping the whole corpus dilutes attention.
- **"Always use agentic RAG / it's the state-of-the-art choice"** — every advanced pattern is an extra LLM round-trip with latency, cost, and failure surface; naive RAG that meets recall and faithfulness targets should ship, and machinery is added only on a measured failure.
- **"Faithfulness means the answer is correct"** — faithfulness is entailment with the *retrieved context*, not truth; a faithful answer over a wrong/stale source is still wrong.
- **"We hit 100% precision, retrieval is solved"** — precision without recall means you may be missing the one chunk that answers the question; low recall silently causes hallucination.
- **"Zero-hallucination is guaranteed"** — the verifier is itself an imperfect model and sources can be wrong; it's a design direction, not a certifiable spec.

---

## STAR Answer Frame

**Situation:** A production RAG assistant over a versioned internal policy corpus was returning confident answers that occasionally fabricated clauses — some with citations that, on inspection, didn't contain the cited fact. Compliance flagged it as a legal risk (the Air Canada refund-policy precedent was raised directly).

**Task:** I owned reducing the fabricated-answer rate to as close to zero as the design allowed *without* collapsing answer coverage, under a per-request latency budget that permitted at most one cheap verification pass.

**Action:** I first built a golden eval set (Q → reference answer + reference chunk IDs) and split failures on two axes — retrieval (context precision/recall, MRR) and generation (faithfulness). Diagnosis showed two distinct causes: oversized fixed-size chunks were slicing tables and producing blurry embeddings (a retrieval-recall failure), and an open-ended prompt with no abstention path let the model answer from parametric memory on misses (a generation failure). I re-chunked with a structure-aware parser keeping tables/clauses atomic plus small-to-big retrieval, attached `policy_id`/`section`/`effective_date` metadata, then layered guardrails: a retrieval score-gate, a grounded prompt at temperature 0 emitting `{claim, source_id, quote}`, a *deterministic* citation validator that dropped any citation not in the retrieved set, and a cheap HHEM faithfulness check (threshold ≥0.9) routing failures to an abstention node. All of it was wired into a CI eval job that fails the build if recall drops below a floor.

**Result:** Fabricated-clause incidents on the eval set dropped from ~15% of answered questions to under 1%, context recall on the golden set rose above the CI floor, and the deterministic-plus-classifier verification stayed within the single-pass latency budget where a large-LLM judge per request would have blown it — trading a modest rise in abstentions for a near-elimination of legally risky fabrications.

---

## Red Flags Interviewers Watch For

- **Treating RAG as "paste documents into the prompt"** — betrays that the candidate thinks the value is in generation, not retrieval; they'll under-invest in chunking, embeddings, and ranking, which is where quality is won or lost.
- **Quoting `chunk_size=1000/overlap=200` as universal law** — signals tutorial-depth understanding; a strong candidate ties chunk parameters to document type, query pattern, and a retrieval eval set.
- **Optimising precision while ignoring recall (or vice versa)** — reveals they don't understand that a retriever can be perfectly precise yet miss the answer entirely, and that low recall silently drives hallucination.
- **Conflating faithfulness with correctness** — shows they'd trust a groundedness score as a truth guarantee and miss that a faithful answer over a stale/wrong source is still wrong.
- **Reaching for agentic RAG / GraphRAG unprompted "to be state of the art"** — signals they add complexity without a measured failure to justify the extra latency, cost, and failure surface; senior candidates justify *not* adding machinery.
- **Proposing retrieve/grade/agent loops with no `max_iterations`** — the #1 runaway-cost bug; a query the corpus can't answer loops forever. A strong candidate threads an iteration counter through state and forces an abstention fallback.
- **No abstention path** — designing a system that always answers reveals no appreciation that "I don't know" is the safe default in high-stakes/compliance settings.
- **Blaming the model for misrouting** — reaching for a bigger LLM when a router picks the wrong index, rather than recognising that vague tool/index *descriptions* are almost always the real cause.
- **Claiming "zero-hallucination" as a guarantee** — a spec-level promise no honest design delivers; the verifier and sources are both imperfect, so it's a direction approached via grounding + verification + abstention.
