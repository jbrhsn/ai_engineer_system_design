# Retrieval Quality and Zero-Hallucination Design

**Section:** LLM Serving & RAG Architecture → RAG Pipeline Design Patterns | **Est. time:** 3 hrs | **Interview relevance:** High — "how do you stop your RAG system from making things up?" is the single most common follow-up in an applied-AI system design loop.

---

## TL;DR

Hallucination in RAG is an *ungrounded* output — a claim not supported by the retrieved context — and most of it traces back to a retrieval failure (the evidence was never fetched) rather than the generator inventing freely. You attack it on two independent axes: **retrieval quality** (measured by context precision, context recall, hit rate, MRR/NDCG) and **generation faithfulness** (the answer is entailed by the retrieved context, measured by faithfulness/groundedness). The design toolkit is grounded prompting ("answer only from context, else say you don't know"), citation enforcement, faithfulness self-verification, score-threshold abstention, and guardrails — layered because no single one is sufficient. **The one thing to remember: "zero-hallucination" is a design goal you approach through grounding + verification + abstention, never a guarantee you can literally deliver — so build the system to say "I don't know" instead of guessing.**

---

## ELI5 — Explain It Like I'm 5

Imagine an open-book exam where the teacher gives you a strict rule: you may only write an answer if you can point to the exact sentence in the textbook that proves it, and you must copy the page number next to your answer. If the book doesn't cover the question, you're required to write "not in the book" rather than guessing from memory. A good student first has to *find the right page* (that's retrieval), and then has to *only write what the page actually says* (that's faithful generation). Two different things can go wrong: you flipped to the wrong page (retrieval failure), or you found the right page but wrote something it never said (generation failure). The common misconception is that handing the student a textbook magically stops all wrong answers — it doesn't; a student can still ignore the book and write from memory, or the book might simply not contain the answer, which is exactly why the "not in the book" rule and the page-number requirement matter as much as the book itself.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain what hallucination means specifically in a RAG context and trace how a retrieval failure produces an ungrounded answer
- [ ] Measure retrieval quality using context precision, context recall, hit rate, and MRR/NDCG, and state what each one misses
- [ ] Measure generation faithfulness/groundedness and distinguish it from retrieval quality and from answer relevancy
- [ ] Design a low-hallucination pipeline using grounded prompting, citation enforcement, faithfulness verification, and score-threshold abstention
- [ ] Diagnose whether a bad answer came from a retrieval failure or a generation failure, and pick the correct fix

---

## Visual Overview

### Retrieval Failure vs Generation Failure — Diagnosis Tree

```
A bad / unsupported answer was produced.
│
├── Did retrieval return a chunk that actually contains the answer?
│   │
│   ├── NO ──► RETRIEVAL FAILURE
│   │          (low context recall / evidence never fetched)
│   │          Fix: chunking, embeddings, top-k, hybrid search,
│   │               reranking, query rewriting
│   │
│   └── YES ─► Did the generated answer stay within that chunk?
│              │
│              ├── NO ──► GENERATION FAILURE
│              │           (low faithfulness — model used parametric memory)
│              │           Fix: grounded prompt, citation enforcement,
│              │                faithfulness check, lower temperature
│              │
│              └── YES ─► Not a hallucination — answer is grounded.
│                          (If still "wrong", the SOURCE doc is wrong.)
```

### Grounded Generation + Faithfulness-Check Loop

```
Query ──► Retrieve top-k ──► Score gate ──► Grounded prompt ──► Generate
                                │                                   │
                       score < threshold?                           ▼
                                │                          Faithfulness judge
                                ▼                          (answer ⊆ context?)
                          ABSTAIN                                   │
                     "I don't know"                    ┌────────────┴───────────┐
                                                     PASS                      FAIL
                                                       │                        │
                                                       ▼                        ▼
                                              Return answer + citations   Abstain / retry
```

### Metric Map — What Each Metric Actually Checks

```
                    ┌──────────────────────────────────────────┐
   RETRIEVAL SIDE   │  Query ──► [ retrieved_contexts ]          │
                    │                                            │
   Context Precision│  Are the fetched chunks relevant &         │
                    │  ranked with relevant ones on top?         │
   Context Recall   │  Did we fetch ALL the evidence needed?     │
   Hit Rate / MRR / │  Is a relevant chunk present, and how      │
   NDCG             │  high is it ranked?                        │
                    └──────────────────────────────────────────┘
                                     │  context passed to LLM
                                     ▼
                    ┌──────────────────────────────────────────┐
   GENERATION SIDE  │  [ context ] ──► Answer                    │
                    │                                            │
   Faithfulness /   │  Is every claim in the answer supported    │
   Groundedness     │  by the retrieved context? (NOT truth)     │
   Answer Relevancy │  Does the answer address the question?     │
                    └──────────────────────────────────────────┘
```

---

## Key Concepts

### Hallucination in RAG and how retrieval failure causes it

**What is it?** A hallucination in RAG is a generated statement that is *not supported by the retrieved context* — regardless of whether it happens to be true in the real world. OWASP catalogs this under LLM09:2025 Misinformation, describing hallucination as the model filling gaps with statistical patterns and producing content that "sounds correct but is completely unfounded."

**How does it work mechanistically?** RAG conditions the generator on retrieved passages, but the LLM still has its parametric memory and a strong prior to *answer* rather than refuse. When retrieval fails to surface the needed evidence (low context recall) or buries it under irrelevant chunks (low context precision), the model has no grounding for the specific claim, so it falls back on parametric memory and fabricates. This is why the majority of RAG hallucinations are *downstream symptoms of a retrieval failure*: the generator was never given the sentence it needed. A smaller class is genuine generation failure — the evidence was present but the model over-generated beyond it.

**Where does it appear in real systems?** In production this shows as a confident answer with a citation that, on inspection, doesn't actually contain the cited fact; in evaluation it shows as a high answer-relevancy but low faithfulness score. OWASP's canonical example is the Air Canada chatbot that invented a refund policy — a fabricated "policy detail" a compliance-grade system must never emit.

### Retrieval quality metrics: precision, recall, hit rate, MRR/NDCG

**What is it?** A family of metrics scoring the *retriever*, independent of what the LLM later writes. Context precision measures whether relevant chunks are ranked at the top; context recall measures whether all needed evidence was fetched; hit rate/MRR/NDCG measure whether and how highly a relevant chunk appears in the ranked list.

**How does it work mechanistically?** In Ragas, **Context Precision@K** is the mean of Precision@k weighted by a relevance indicator `v_k`, so a relevant chunk sitting at rank 1 scores higher than the same chunk at rank 5 — it explicitly rewards *ranking*. **Context Recall** breaks the reference answer into claims and computes the fraction of those claims attributable to the retrieved context — it explicitly penalizes *missing* evidence. Hit rate is the binary "was any relevant chunk in top-k?"; MRR is the reciprocal of the rank of the first relevant chunk (rewards putting it first); NDCG discounts relevance logarithmically by position (rewards good ordering of *multiple* graded-relevance results).

**Where does it appear in real systems?** Ragas exposes `ContextPrecision`, `ContextRecall`, and non-LLM/ID-based variants (`IDBasedContextPrecision`, `NonLLMContextRecall`) you run over a labeled eval set; LlamaIndex ships `RetrieverEvaluator` (Usage Pattern — Retrieval) computing hit rate and MRR against expected node IDs. You wire these into a CI eval job that fails the build if recall drops below a floor.

### Faithfulness / groundedness

**What is it?** Faithfulness (a.k.a. groundedness) measures how factually consistent the *response* is with the *retrieved context* — an answer is faithful if every claim it makes can be inferred from the context. It is deliberately **not** a measure of real-world truth.

**How does it work mechanistically?** Ragas computes it by (1) decomposing the answer into atomic claims, (2) checking each claim for entailment against the retrieved context, and (3) dividing supported claims by total claims — the Einstein example scores 0.5 because "born in Germany" is supported but "20th March 1879" is not (context says 14 March). LlamaIndex's `FaithfulnessEvaluator` returns a binary `passing` per response (and can score each source node individually). Ragas can also swap the LLM entailment step for Vectara's HHEM classifier for cheaper production checking.

**Where does it appear in real systems?** As an offline eval metric on your golden set, and as an *online* self-verification gate that runs the same claim-entailment check on live traffic before returning the answer — the model of "I don't know" over "confidently wrong." Note the asymmetry: you can have perfect faithfulness (answer strictly from context) while still being *useless* if context recall was low — that's why you never track faithfulness alone.

### Grounded prompting and abstention

**What is it?** A prompting pattern that instructs the model to answer *only* from the provided context and to explicitly abstain ("I don't know" / "not covered in the provided sources") when the context is insufficient.

**How does it work mechanistically?** The system prompt sets a hard constraint and an escape hatch, converting the model's default "always answer" behavior into "answer-or-abstain." Lowering temperature reduces the sampling variance that lets the model drift off-context. Abstention is what makes "zero-hallucination" *reachable in the limit* — a system that refuses when unsure cannot fabricate, trading coverage for safety.

**Where does it appear in real systems?** As the system message in the generation node of a LangChain/LangGraph chain, and as an OWASP-recommended mitigation (RAG + risk communication + UI design that labels/limits AI output). In LangGraph you implement abstention as a conditional edge: if the faithfulness check or score gate fails, route to a canned "I don't know" node instead of returning generated text.

### Citation / attribution enforcement

**What is it?** Requiring the model to attach, for each claim, a pointer to the specific source chunk (an ID, a quote, or a span) that supports it — and rejecting or flagging any answer whose citations don't check out.

**How does it work mechanistically?** Citations turn faithfulness from a fuzzy judgment into a *verifiable* one: you can programmatically confirm the cited chunk ID exists in the retrieved set and (optionally) that the cited span actually entails the claim. This both nudges the model toward grounding (it has to name its evidence) and gives downstream code a cheap deterministic check before an expensive LLM-judge call.

**Where does it appear in real systems?** As a structured output schema (`{claim, source_id, quote}`) enforced via function-calling/JSON mode, plus a validator that drops citations pointing to chunk IDs not in `retrieved_contexts`. This is the mechanism behind "grounded answers with sources" in enterprise assistants.

### Faithfulness verification / self-verification

**What is it?** An automated second pass — LLM-as-judge or a trained classifier — that checks the generated answer against the retrieved context and returns a groundedness score used to accept, retry, or abstain.

**How does it work mechanistically?** After generation, a judge receives (answer, retrieved_contexts) and applies the same claim-entailment decomposition Ragas uses, emitting a score in [0,1]. You threshold it: pass → return; fail → abstain or regenerate with a stricter prompt. Using a small classifier (HHEM-2.1) instead of a large LLM judge makes the check cheap enough for every request.

**Where does it appear in real systems?** As a post-generation guardrail node in the graph, or the Ragas `Faithfulness` / LlamaIndex `FaithfulnessEvaluator` run online. It is the "verification" half of "grounding + verification" and the reason zero-hallucination is *approached* rather than *guaranteed* — the judge itself is imperfect.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Retrieval score threshold (`min_score`) | Minimum similarity/rerank score a chunk must clear to be passed to the generator | Set it from the eval set: pick the score below which precision collapses; if the top chunk is below it, abstain rather than generate. Start strict for compliance domains, loosen only if recall on the golden set stays high. |
| `top_k` | How many chunks are retrieved and passed as context | Raise until context recall on your eval set plateaus, then stop — more chunks past the plateau lowers precision and invites distraction (noise sensitivity). Typical start 3–5; use reranking to keep only the best few. |
| Faithfulness-judge threshold | Score below which the answer is rejected/abstained | High-stakes (policy, medical, legal) → ≥0.9 and abstain on fail; low-stakes internal tools → ~0.7 with a warning. Never set to 0 (no gate) for user-facing factual answers. |
| Abstention policy | When the system says "I don't know" vs guessing | Abstain if (top score < threshold) OR (faithfulness < threshold) OR (no citation resolves). Prefer false abstentions over false answers whenever a wrong answer carries legal/safety cost. |
| Generation temperature | Sampling randomness of the generator | Set 0–0.2 for grounded factual QA so the model stays on-context; higher temperature increases drift into parametric memory. |
| Reranker on/off | Whether a cross-encoder re-scores top-k before generation | Turn on when context precision is the bottleneck (relevant chunk retrieved but ranked low, hurting MRR/NDCG); it won't help if recall is the problem. |

### Worked Example: Requirement → Decision

**Given:** You must design a customer-facing QA assistant for an insurance company that answers questions about policy terms. It must **never fabricate a policy detail** (an invented clause could create legal liability, per the Air Canada precedent), but it should answer confidently when the policy documents do cover the question.

- **Step 1 — Identify the goal:** Maximize correct, grounded answers about policy terms while driving the fabricated-policy-detail rate as close to zero as the design allows; abstain rather than guess.
- **Step 2 — Define inputs:** User question; a vector index of versioned policy PDFs (chunked with metadata: policy_id, section, effective_date); retrieval scores; a golden eval set of Q→(reference answer, reference chunk IDs).
- **Step 3 — Define outputs:** Either (a) an answer with per-claim citations `{claim, source_id, quote}` that all resolve to retrieved chunks, or (b) an explicit abstention "This isn't covered in the current policy documents — please contact an agent."
- **Step 4 — Apply constraints:** Zero-tolerance for fabricated clauses (legal); answers must cite the exact clause; must reflect the current policy version (data freshness); per-request latency budget allows one cheap verification pass but not multiple large-LLM judge calls.
- **Step 5 — Select the approach:** Layer (1) hybrid retrieval + reranker with a **score threshold** gate; (2) a **grounded prompt** ("answer only from context; if not covered, say so") at temperature 0; (3) **citation enforcement** with a deterministic validator that drops any citation whose `source_id` isn't in `retrieved_contexts`; (4) a **faithfulness check** using the cheap HHEM classifier with a high threshold (≥0.9), routing failures to the abstention node. Rationale vs alternatives: grounded prompting *alone* still hallucinates when retrieval misses, and a faithfulness judge alone can't fix a retrieval miss — layering the score gate (catches "no evidence"), citations (deterministic, cheap), and the verifier (catches subtle over-generation) is what makes the fabrication rate approach zero within the latency budget. A single big-LLM judge per request was rejected on cost/latency.

---

## Implementation

```python
# Scenario: A compliance QA assistant must answer ONLY from the retrieved policy
# clauses and abstain otherwise. This is the grounded prompt + citation schema +
# deterministic citation validation, run at temperature 0 for a low-stakes drift budget.

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

GROUNDED_SYSTEM = """You answer questions about insurance policy terms.
RULES:
1. Use ONLY the information in the <context> block below.
2. For every claim, cite the source_id of the chunk that supports it.
3. If the context does not contain the answer, respond EXACTLY:
   {"answer": "Not covered in the provided policy documents.", "citations": []}
Return JSON: {"answer": str, "citations": [{"claim": str, "source_id": str}]}"""

prompt = ChatPromptTemplate.from_messages([
    ("system", GROUNDED_SYSTEM),
    ("human", "<context>\n{context}\n</context>\n\nQuestion: {question}"),
])
llm = ChatOpenAI(model="gpt-4o", temperature=0)  # low temp keeps it on-context

def validate_citations(answer_json, retrieved_ids: set[str]) -> bool:
    # Deterministic, cheap: reject answers citing chunks we never retrieved.
    cited = {c["source_id"] for c in answer_json["citations"]}
    return bool(cited) and cited.issubset(retrieved_ids)
```

```python
# Anti-pattern: an open-ended prompt with no grounding rule, no abstention path,
# and no citation. It invites the model to answer from parametric memory, so on a
# retrieval MISS it confidently fabricates a policy clause (the Air Canada failure).

bad_prompt = "You are a helpful insurance assistant. Answer the user's question."
# ... retrieval may return junk or nothing, but the model still "helpfully" answers.

# Correct approach: gate on retrieval score, force grounding + abstention, then verify.
from ragas.metrics import Faithfulness  # LLM/HHEM claim-entailment check

def answer_with_guardrails(question, retriever, faith_scorer, min_score=0.35, min_faith=0.9):
    hits = retriever.retrieve(question)               # returns (chunk, score)
    if not hits or hits[0].score < min_score:
        return {"answer": "Not covered in the provided policy documents.", "citations": []}

    context = "\n".join(f"[{h.id}] {h.text}" for h in hits)
    resp = (prompt | llm).invoke({"context": context, "question": question}).content
    ans = parse_json(resp)

    retrieved_ids = {h.id for h in hits}
    if not validate_citations(ans, retrieved_ids):     # deterministic gate first
        return abstain()

    faith = faith_scorer.score(user_input=question,    # then the entailment gate
                               response=ans["answer"],
                               retrieved_contexts=[h.text for h in hits]).value
    return ans if faith >= min_faith else abstain()
# What the anti-pattern breaks: with no score gate the model answers on an empty/irrelevant
# context, and with no abstention or verification a fabricated clause is returned verbatim.
```

---

## Common Pitfalls & Misconceptions

- **"RAG eliminates hallucination"** — Beginners assume that because the model is *given* documents, it must *use* them. In reality the LLM retains its parametric memory and its bias to always answer, so on a retrieval miss it fabricates anyway; RAG only *reduces* hallucination and only when paired with grounded prompting, abstention, and verification.
- **"More context always helps"** — It feels safer to raise top-k so the answer is "surely in there somewhere." But past the recall plateau, extra chunks lower context precision and add distracting/irrelevant text (noise sensitivity), which can *increase* hallucination and cost; raise top-k only until recall plateaus, then rerank down.
- **"Faithfulness is the same as being correct"** — People treat a high faithfulness score as "the answer is right." Faithfulness only means the answer is entailed by the retrieved context; if the retrieved document is wrong or stale, a perfectly faithful answer is still factually wrong — truth also depends on source quality and recall.
- **"Just measure precision" (ignoring recall)** — Teams optimize the metric that's easy to eyeball (are the top chunks relevant?) and forget that a retriever can have perfect precision while *missing* the one chunk that answers the question. Low recall silently causes hallucination because the evidence was never fetched; always track recall alongside precision.
- **"Zero-hallucination is achievable as a guarantee"** — The phrase sounds like a spec you can certify. It isn't: the verifier is itself an imperfect model and sources can be wrong, so treat "zero-hallucination" as a design *direction* reached via grounding + verification + abstention, and build in "I don't know" as the safe default.

---

## Key Definitions

| Term | Definition |
|---|---|
| Hallucination (RAG) | A generated claim not supported by the retrieved context, whether or not it is true in reality. |
| Retrieval failure | The retriever did not surface a chunk containing the needed evidence (typically low context recall). |
| Generation failure | The needed evidence was retrieved, but the answer went beyond/against it (low faithfulness). |
| Context Precision | Ragas metric: mean Precision@k over retrieved chunks, rewarding relevant chunks ranked highest. |
| Context Recall | Ragas metric: fraction of reference-answer claims attributable to the retrieved context (did we fetch all needed evidence?). |
| Hit Rate | Fraction of queries for which at least one relevant chunk appears in the top-k. |
| MRR | Mean Reciprocal Rank — average of 1/(rank of first relevant chunk); rewards ranking the answer first. |
| NDCG | Normalized Discounted Cumulative Gain — ranking metric that discounts relevance by log position; rewards good ordering of graded results. |
| Faithfulness / Groundedness | Fraction of answer claims that can be inferred from the retrieved context; consistency with context, not truth. |
| Answer Relevancy | How well the answer addresses the question (independent of whether it is grounded). |
| Abstention | The system explicitly declining to answer ("I don't know") when context or confidence is insufficient. |
| Citation enforcement | Requiring and programmatically validating that each claim points to a retrieved source chunk. |

---

## Summary / Quick Recall

- Hallucination in RAG = ungrounded output; most of it is a *retrieval* failure surfacing as a generation symptom.
- Measure two axes separately: retrieval quality (precision, recall, hit rate, MRR/NDCG) and generation faithfulness/groundedness.
- Context precision rewards ranking; context recall catches *missing* evidence — never track precision alone.
- Faithfulness ≠ truth: it means the answer is entailed by the context; a faithful answer over a wrong source is still wrong.
- Low-hallucination design = layers: score-gate → grounded prompt (temp 0) → citation enforcement → faithfulness verify → abstain on any failure.
- Diagnose first: was the answer *in* a retrieved chunk? No → fix retrieval; Yes but answer strayed → fix generation.
- "Zero-hallucination" is a direction, not a guarantee — the safe default is "I don't know."

---

## Self-Check Questions

1. In the RAG context, what precisely does it mean for an answer to be "faithful," and how does that differ from the answer being "correct"?

   <details><summary>Answer</summary>

   Faithful means every claim in the answer can be inferred from the *retrieved context* — it measures consistency with the provided evidence, not real-world truth. Correct means the answer matches reality. The two diverge when the retrieved source is itself wrong or stale: the answer can be perfectly faithful (score 1.0) yet factually incorrect. The tempting wrong answer — "faithful means the answer is true" — fails because faithfulness deliberately ignores source quality; you need recall + source correctness for truth.

   </details>

2. Your RAG assistant returned a confident but fabricated policy clause. You inspect the trace and find that none of the top-k retrieved chunks contained the clause. Which failure is this, and which fix is appropriate?

   <details><summary>Answer</summary>

   This is a **retrieval failure** (low context recall — the evidence was never fetched), so the model fell back to parametric memory and fabricated. The right fixes target retrieval: better chunking, stronger embeddings/hybrid search, higher top-k until recall plateaus, reranking, or query rewriting — plus a score-gate/abstention so it says "I don't know" when nothing relevant is found. Tightening the generation prompt or the faithfulness judge alone is the wrong fix here: the model can't ground on evidence it was never given.

   </details>

3. **Which TWO** of the following are correct statements about retrieval metrics?
   - A. Context recall measures whether all the evidence needed to answer was actually retrieved.
   - B. Context precision is unaffected by the *ranking* of chunks; only their presence matters.
   - C. MRR rewards placing the first relevant chunk as high as possible in the ranking.
   - D. High context precision guarantees the answer will be grounded.
   - E. Hit rate measures the fraction of answer claims entailed by the context.

   <details><summary>Answer</summary>

   **A and C.** A is correct — context recall is precisely "did we fetch all needed evidence," computed in Ragas as reference-claims attributable to the context. C is correct — MRR is the mean of 1/(rank of first relevant chunk), so ranking it first maximizes the score. B is wrong: Ragas context precision explicitly weights Precision@k by rank, so an irrelevant chunk at position 1 lowers it. D is wrong: precision only concerns retrieval; the generator can still stray from good context (a generation failure), and high precision with low recall still causes hallucination. E describes faithfulness, not hit rate — hit rate is just "was any relevant chunk in top-k."

   </details>

4. A stakeholder says: "Let's just raise top-k to 20 so the answer is always in the context — that will kill hallucination." Analyze why this can backfire.

   <details><summary>Answer</summary>

   Raising top-k helps recall only until it plateaus; beyond that you add irrelevant chunks that *lower context precision* and introduce distracting text (noise sensitivity), which can increase — not decrease — hallucination, while also raising token cost and latency. The correct approach is to raise top-k only until recall on the eval set plateaus, then use a reranker to keep the few best chunks. The naive assumption — "more context is strictly safer" — ignores that the generator can be misled by irrelevant retrieved text and that precision and recall trade off.

   </details>

5. You have a fixed per-request latency budget that allows exactly one cheap verification step, and a legal requirement to never fabricate. You can add ONE of: (a) a large-LLM faithfulness judge, (b) a deterministic citation validator plus a small-classifier (HHEM) faithfulness check, (c) a second retrieval pass. Which do you choose and why?

   <details><summary>Answer</summary>

   Choose **(b)**. The deterministic citation validator is essentially free and catches the common case (citing a chunk that wasn't retrieved), and the HHEM classifier gives a cheap per-request faithfulness gate that fits the latency budget — together they push the fabrication rate down while abstaining on failure. (a) A large-LLM judge is the most thorough but blows the latency/cost budget for every request. (c) A second retrieval pass improves recall but does nothing to *verify* the generated answer, so it doesn't directly satisfy the "never fabricate" gate — you'd still return unverified text. The layered cheap-verify + abstain design is what makes zero-hallucination *approachable* under the constraint.

   </details>

---

## Further Reading

- [Faithfulness — Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — *verified 2026-07-29* — Claim-decomposition definition and formula for groundedness, plus the cheap HHEM-2.1 classifier variant for production.
- [Context Precision — Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/) — *verified 2026-07-29* — Precision@k formula and how ranking of relevant chunks affects the score.
- [Context Recall — Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — *verified 2026-07-29* — Reference-claim attribution formula for measuring missing-evidence (recall) failures.
- [List of available metrics — Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) — *verified 2026-07-29* — Full RAG metric set including Response Groundedness and Noise Sensitivity.
- [Usage Pattern (Response Evaluation) — LlamaIndex](https://docs.llamaindex.ai/en/stable/module_guides/evaluating/usage_pattern/) — *verified 2026-07-29* — `FaithfulnessEvaluator` (hallucination) and `RelevancyEvaluator` for online/offline answer checking.
- [LLM09:2025 Misinformation — OWASP GenAI Security Project](https://genai.owasp.org/llmrisk/llm092025-misinformation/) — *verified 2026-07-29* — Authoritative definition of hallucination-driven misinformation and mitigation strategies (RAG, verification, human oversight).
