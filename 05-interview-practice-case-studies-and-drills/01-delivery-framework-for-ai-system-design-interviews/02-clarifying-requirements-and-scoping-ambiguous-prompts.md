# Clarifying Requirements and Scoping Ambiguous Prompts

**Section:** Interview Practice — Delivery Framework for AI System Design Interviews | **Est. time:** 2 hrs | **Interview relevance:** High — the first 5–7 minutes of every AI system design interview decide whether you design the *right* system; a mis-scoped prompt guarantees a wrong answer no matter how good the architecture.

---

## TL;DR

An AI system design prompt is deliberately vague ("design a chatbot for our docs") because the interviewer is testing whether you can turn ambiguity into a *bounded, prioritized, quantified* problem before you touch architecture. Your job in the opening minutes is to ask targeted AI-specific clarifying questions (scale/QPS, p95 latency, hallucination tolerance, data sensitivity, freshness, cost ceiling, users), convert the answers into functional and non-functional requirements, estimate scale to the degree that it changes the design, state explicit assumptions to close remaining gaps, and name the hard part early. **The one thing to remember: the prompt is the *start* of a negotiation about scope, not a spec to implement — the candidate who clarifies drives the design, and the candidate who assumes silently builds the wrong system.**

---

## ELI5 — Explain It Like I'm 5

Imagine a client walks into an architect's office and says "build me a house." A bad architect starts drawing a two-storey suburban home immediately — and gets it wrong when it turns out the client wanted a mountain cabin for one person with no running water. A good architect asks first: how many people live here, what's the budget, is it on a cliff or a flat lot, do you need it heated in winter, when do you need to move in? Each answer *removes* a whole category of designs that would have been wrong. Only after the questions does the architect pick up a pencil. The most common mistake is thinking the sentence "build me a house" already told you what to build — it didn't; it told you the client doesn't yet know how to describe what they need, and it's your job to find out by asking.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Convert a vague one-line AI prompt into a prioritized list of functional and non-functional requirements.
- [ ] Ask the specific clarifying questions that change an *AI* system's design (hallucination tolerance, freshness, data sensitivity, latency budget, cost ceiling) rather than generic ones.
- [ ] Estimate scale (users, QPS, corpus size, token volume, embedding count) only to the depth that influences a design decision.
- [ ] State explicit bounding assumptions when the interviewer defers, so scope stays fixed.
- [ ] Identify the "hard part" of an agentic/RAG problem in the first few minutes and steer the interview toward it.

---

## Visual Overview

### Vague Prompt → Scoped Requirements Funnel

```
"Build an AI assistant for our support docs"   (ambiguous, unbounded)
                 │
                 ▼
   ┌────────────────────────────────────┐
   │  Targeted clarifying questions      │
   │  who? scale? p95? hallucination?    │
   │  data sensitivity? freshness? cost? │
   └────────────────────────────────────┘
                 │
                 ▼
   Functional reqs (top 3)  +  Non-functional reqs (top 3–5, quantified)
                 │
                 ▼
   Explicit assumptions close remaining gaps
                 │
                 ▼
        Bounded problem  ──►  name the HARD PART  ──►  design
```

### Functional vs Non-Functional Split (for AI systems)

```
FUNCTIONAL ("users should be able to...")   NON-FUNCTIONAL ("system should be...")
├─ answer questions from the docs           ├─ p95 latency < 3s end-to-end
├─ cite the source passage                  ├─ hallucination rate < X% (grounded-only)
├─ say "I don't know" when unsure           ├─ handle 50 QPS peak
└─ escalate to a human on failure           ├─ no PII leaves the VPC (residency)
                                            ├─ docs freshness ≤ 24h
                                            └─ cost ≤ $0.02 / query
```

### Back-of-Envelope Scale Estimation Flow

```
Users (DAU) ──► queries/user/day ──► queries/day
                                        │
                        ÷ 86,400 s ──► average QPS
                                        │
                        × peak factor (~3x) ──► peak QPS  (drives serving/replicas)

Corpus docs × avg tokens/doc ──► total tokens
             ÷ chunk size ──► # chunks ──► # embeddings  (drives vector DB sizing)
```

### "Is a calculation worth doing?" Decision Tree

```
Will the number change a design decision?
├── Yes ──► do the math now (e.g. shard the vector index? one heap or many?)
└── No  ──► state "assume large distributed scale" and move on
```

---

## Key Concepts

### Functional vs Non-Functional Requirements for AI Systems

**What it is.** Functional requirements are the "users should be able to…" statements — the core features. Non-functional requirements are the "the system should be…" statements about quality attributes (latency, availability, accuracy, security), ideally quantified. Per Hello Interview's delivery framework, you split requirements into exactly these two buckets and prioritize the top 3 functional and top 3–5 non-functional.

**How it works under the hood.** For AI systems, non-functional requirements do more work than in classic system design, because *correctness itself is probabilistic*. "The system should be low latency" is meaningless; "feed renders in under 200ms" is useful. The AI analogue is that "the assistant should be accurate" is meaningless — you must quantify a *hallucination tolerance* and specify whether answers must be grounded in retrieved context. The functional bucket for an agentic/RAG system almost always includes non-obvious features like "cite sources," "abstain when unsure," and "escalate to human," which candidates forget because the prompt only mentions "answer questions."

**Where it appears in a real interview.** You write two labelled bullet lists on the board in the first 5 minutes. A strong RAG example: *Functional* — (1) answer questions grounded in the docs, (2) return citations, (3) say "I don't know" when retrieval is weak. *Non-functional* — (1) p95 < 3s, (2) ≤ 50 QPS peak, (3) grounded-hallucination rate under target, (4) no customer PII leaves the tenant boundary. The lists become the checklist your entire design is graded against.

### The AI-Specific Clarifying Question Set

**What it is.** A reusable battery of targeted questions that surface the constraints unique to AI/ML systems — the ones that flip the architecture. Hello Interview's ML framework explicitly says to clarify *who the users are, their pain points, the current solution, scale (users/QPS), real-time vs batch inference, and latency/privacy constraints* before doing anything else.

**How it works under the hood.** Each question is a *design fork*, not small talk. "What's the hallucination tolerance?" forks between pure-generation and retrieval-grounded-with-abstention. "Does data leave our VPC?" forks between a hosted API model and a self-hosted open-weights model. "How fresh must answers be?" forks between a nightly batch re-index and a streaming/incremental index. Asking a generic question ("how many users?") without connecting it to a fork wastes the interviewer's time; connecting it ("if it's >100 QPS I'll need to cache embeddings and possibly shard the vector index") shows senior signal.

**Where it appears in a real interview.** The AI-specific set to keep in your head: **Users** (who, internal vs external, how many languages?); **Scale/QPS** (DAU, queries/user, peak factor); **Latency** (p95 budget end-to-end, streaming tokens acceptable?); **Accuracy** (hallucination tolerance, must answers be grounded + cited?); **Data sensitivity** (PII, residency, can data go to a third-party API?); **Freshness** (how stale can retrieved content be?); **Cost** (ceiling per query / per month); **Online vs offline** (real-time chat vs batch summarization). This maps directly to the OWASP GenAI risks you must design against — e.g. Sensitive Information Disclosure (LLM02) and Misinformation (LLM09).

### Scale Estimation (Only When It Changes the Design)

**What it is.** Back-of-the-envelope math on users, QPS, corpus size, token volume, and embedding count. Hello Interview is explicit that this is *often unnecessary* — do the calculation only when it will directly influence a design decision.

**How it works under the hood.** The chain for a RAG system: DAU × queries/user/day = queries/day; ÷ 86,400 = average QPS; × ~3 peak factor = peak QPS (this decides replica count and whether you need an inference queue). Separately, corpus_docs × avg_tokens/doc = total tokens; ÷ chunk_size (e.g. 512 tokens) = number of chunks = number of embeddings; × vector_dim × 4 bytes ≈ raw index size (this decides single-node vs sharded vector DB, and whether you need quantization). Token volume × price/1K tokens = cost, which you check against the cost ceiling.

**Where it appears in a real interview.** You do this at the whiteboard *at the moment the number matters* — e.g. "10M docs × 800 tokens ÷ 512 ≈ 15.6M chunks; at 1536-dim float32 that's ~96GB raw, so a single node won't hold it in RAM — I'll shard the index or use product quantization." That single calculation justifies a whole architectural branch. Estimating storage just to conclude "so, it's a lot" gives the interviewer no signal.

### Stating Explicit Assumptions to Bound Scope

**What it is.** Declaring a concrete value or constraint yourself when the interviewer won't commit, so the problem stays finite. "I'll assume English-only for v1 and note multilingual as a follow-up."

**How it works under the hood.** Interviewers often deflect ("what do you think?") to see if you can bound the space. An explicit assumption converts an open variable into a fixed one you can design against, and — crucially — it is *reversible on request*: if the interviewer disagrees, they'll correct you, which is itself valuable signal. Silent assumptions are the failure mode; they look identical to explicit ones on the whiteboard but the interviewer never got to veto them, so you may spend 20 minutes designing for the wrong constraint.

**Where it appears in a real interview.** Verbalized and written: "**Assumption:** ~50k internal employees, ~5 queries/day each, English-only, docs refreshed nightly, answers must be grounded and cited, ~$500/day budget. Push back if any of these are off." This one sentence bounds users, QPS, language, freshness, accuracy, and cost simultaneously.

### Finding the Hard Part Early

**What it is.** Identifying, in the first few minutes, the single technical challenge that will dominate the rest of the interview — the thing "teams work on for years."

**How it works under the hood.** Every AI prompt has a load-bearing difficulty hidden by the innocent phrasing. "Chatbot for our docs" → the hard part is usually *retrieval quality and grounding* (avoiding confident wrong answers), not the chat UI. "Agent that books travel" → the hard part is *reliable tool use and error recovery / bounded agency* (OWASP LLM06 Excessive Agency), not the LLM call. Naming it early lets you allocate time to it in deep dives and signals staff-level problem navigation.

**Where it appears in a real interview.** After stating requirements you say: "The interesting part here isn't serving the LLM — it's keeping answers grounded so we don't hallucinate policy that doesn't exist. I'll design the retrieval + citation + abstention path as the core and treat the chat surface as thin." That sentence sets the agenda for the remaining 30 minutes.

### Key Parameters / Configuration Knobs

Frame these as the **requirement dimensions to pin down** — each answer changes the design.

| Parameter (requirement dimension) | What it controls | Decision rule |
|---|---|---|
| Peak QPS | Serving replicas, batching, need for an inference queue | If peak > ~10 QPS, add embedding cache + horizontal LLM replicas; if < 1 QPS, single node is fine — don't over-build. |
| p95 latency budget (end-to-end) | Model size, reranking depth, streaming, sync vs async | If p95 < 2s, stream tokens and skip a heavy cross-encoder rerank; if p95 can be 10s+, allow multi-hop agentic retrieval. |
| Hallucination tolerance | Grounding strictness, abstention, human-in-the-loop | If tolerance is near-zero (legal/medical), force retrieval-grounded answers with citations + "I don't know" fallback + human review; if high (brainstorming), allow free generation. |
| Data sensitivity / residency (PII) | Hosted API vs self-hosted model, VPC boundary, redaction | If PII must not leave the tenant/region, self-host open-weights or use a VPC-deployed model; else a hosted API is acceptable. Maps to OWASP LLM02. |
| Freshness (max staleness of content) | Re-index cadence: batch vs incremental/streaming | If staleness must be < minutes, use incremental/streaming indexing; if daily is fine, a nightly batch re-embed is cheaper and simpler. |
| Cost ceiling (per query / per month) | Model tier, context length, caching, retrieval top-k | If budget is tight, use a smaller model + aggressive caching + smaller top-k; if generous, allow larger models and reranking. Guards against OWASP LLM10 Unbounded Consumption. |
| Languages supported | Embedding model choice, eval coverage, tokenization cost | If multilingual, pick a multilingual embedding model and expand eval sets; if English-only for v1, state it as an explicit assumption and defer. |
| Online vs offline (real-time vs batch) | Whole serving architecture | If real-time chat, optimize for low-latency single-query serving; if batch (e.g. nightly doc summarization), optimize for throughput and cost, not latency. |

### Worked Example: Requirement → Decision

**Given:** The interviewer says only: *"Build an AI assistant for our support docs."* Nothing else.

**Step 1 — Identify the goal.** Reduce support ticket volume by letting users self-serve accurate answers from existing documentation. Note the *business* objective (deflect tickets) may differ from a naive ML objective ("answer questions") — a confidently wrong answer is worse than "I don't know" because it erodes trust and can create liability.

**Step 2 — Define inputs (via clarifying questions).** Ask and record: *Who?* → external customers, ~200k MAU, English-only for v1. *Corpus?* → ~20k public help articles, ~1k tokens each, updated a few times/week. *Latency?* → p95 < 3s, streaming acceptable. *Accuracy?* → low hallucination tolerance; answers must cite the source article and abstain if unsure. *Data sensitivity?* → docs are public, but user *questions* may contain PII, so questions must not be logged to a third party unredacted. *Cost?* → target under ~$0.03/query. *Freshness?* → daily re-index is acceptable.

**Step 3 — Define outputs.** A grounded natural-language answer + citation link(s) to the source article(s), or an explicit "I couldn't find this — here's how to reach support" fallback. This forces the functional requirements: answer, cite, abstain, escalate.

**Step 4 — Apply constraints.** Estimate scale where it matters: 200k MAU × ~0.5 queries/day ≈ 100k queries/day ≈ ~1.2 avg QPS, ~4 QPS peak — modest, so no sharding of the LLM tier; a cache is enough. Corpus: 20k docs × 1k tokens ÷ 512 ≈ 40k chunks — tiny, fits a single-node vector index in memory, no sharding needed. Low hallucination tolerance + public docs + PII-in-questions → retrieval-grounded generation with citations and abstention, PII redaction before any third-party call, daily batch re-index.

**Step 5 — Select the approach + rationale.** Choose **RAG (retrieve → rerank → grounded generate with citation + abstention)** over a fine-tuned model (docs change weekly; RAG re-indexes cheaply and gives citations for free) and over pure generation (unacceptable hallucination risk). Name the hard part: *grounding quality and abstention*, not serving scale. Deep dives will target retrieval quality and the "I don't know" threshold — everything else is a thin wrapper.

---

## Implementation

```text
# Scenario: You have 5 minutes at the start of a RAG interview and need a
# repeatable battery of AI-specific clarifying questions so you never freeze
# on a vague prompt. This checklist maps each question to the design fork it
# resolves — ask them in this order, skip any the prompt already answered.

CLARIFYING CHECKLIST  (each Q → the decision it changes)
  USERS       Who uses it? internal/external? how many? languages?
              → embedding model, eval coverage, auth boundary
  SCALE/QPS   DAU × queries/user/day → QPS; peak factor?
              → replicas, caching, queue, index sharding
  LATENCY     p95 end-to-end budget? streaming ok?
              → model size, rerank depth, sync vs async
  ACCURACY    Hallucination tolerance? must answers be grounded + cited?
              → RAG-with-abstention vs free generation; human-in-the-loop
  DATA        PII? residency/region? can data hit a 3rd-party API?
              → hosted API vs self-hosted; redaction (OWASP LLM02)
  FRESHNESS   Max staleness of content?
              → nightly batch re-index vs incremental/streaming
  COST        Ceiling per query / per month?
              → model tier, top-k, caching (OWASP LLM10)
  MODE        Real-time chat or batch job?
              → whole serving architecture
```

```markdown
<!-- Scenario: You've asked your questions and now need to lock scope on the
     whiteboard in a form the interviewer can veto. This requirements table is
     the artifact your whole design is graded against — top 3 functional,
     top 3-5 non-functional, quantified, plus explicit assumptions. -->

## Requirements — "AI assistant for support docs"

### Functional (users should be able to...)
1. Ask a question and get an answer grounded in the help docs
2. See citation link(s) to the source article(s)
3. Get an honest "I don't know / contact support" when retrieval is weak

### Non-functional (system should be...)
- p95 latency < 3s end-to-end (token streaming allowed)
- ~4 QPS peak (100k queries/day)
- Grounded-only answers; hallucination rate below target; must abstain
- User questions redacted of PII before any 3rd-party model call
- Docs freshness: daily re-index acceptable
- Cost < $0.03 / query

### Explicit assumptions (push back if wrong)
- 200k MAU, ~0.5 queries/user/day, English-only for v1
- 20k public articles (~1k tokens each) — corpus fits one vector node
- HARD PART = grounding + abstention quality, not serving scale
```

```text
# Anti-pattern: Silently assuming requirements and asking only generic
# questions, then diving into architecture. This is the #1 reason mid-level
# candidates fail — they design the wrong system confidently.

Interviewer: "Build an AI assistant for our support docs."
Candidate:   "How many users?"          # generic, not tied to a fork
Interviewer: "A lot."
Candidate:   "Ok, so it's at scale. I'll use an LLM with a vector DB..."
             # jumps straight to architecture; never established
             # hallucination tolerance, PII handling, freshness, or cost.
# What breaks: 20 minutes later the interviewer reveals answers must never
# hallucinate policy and user data can't leave the VPC. The candidate's
# hosted-API, free-generation design is now wrong and must be rebuilt with
# no time left. The scope was never negotiated, so the design was a guess.

# Correct approach: Tie each question to the design decision it resolves,
# state assumptions out loud so they can be vetoed, and name the hard part.

Candidate: "Before I design — who are the users and can their data leave our
            infra? That decides hosted vs self-hosted. What's the tolerance
            for a wrong answer? If it's low I'll ground every answer in
            retrieval and force abstention. How fresh must answers be — that
            picks batch vs streaming indexing. I'll assume 200k external MAU,
            English v1, PII must stay in-VPC, near-zero hallucination
            tolerance, daily freshness — stop me if any of that's off.
            Given that, the hard part is grounding, so I'll center the design
            on retrieval quality + citations + abstention."
# Now every subsequent decision is justified by a stated, vetoed constraint.
```

---

## Common Pitfalls & Misconceptions

- **Treating the prompt as a spec** — Beginners read "chatbot for our docs" as a finished requirement because it's phrased like an instruction. The correct mental model is that the vagueness is *intentional*: the prompt is the opening of a scoping negotiation, and clarifying is the graded skill, not obedience.
- **Asking generic questions with no fork attached** — Candidates ask "how many users?" because it feels like the "right" system-design question, but they don't connect it to a decision. The fix: only ask a question if you can state what design choice its answer will change ("if >10 QPS I'll cache embeddings"); otherwise you're burning time interviewers evaluate as noise.
- **Skipping accuracy/hallucination requirements entirely** — In classic system design there's no "correctness tolerance," so candidates carry that habit over and never quantify it. But for AI systems, hallucination tolerance is the *most* design-shaping non-functional requirement — it decides RAG-with-abstention vs free generation and whether you need human-in-the-loop.
- **Doing capacity math that changes nothing** — Beginners calculate DAU, storage, and QPS out of ritual and conclude "it's a lot." Do the math *only* when the number forks the design (e.g. does the vector index fit one node?); otherwise say "assume large distributed scale" and move on, per the delivery framework.
- **Making assumptions silently** — Candidates assume English-only or public data in their head and design accordingly, so the interviewer never gets to correct a wrong assumption. Always verbalize and write assumptions with an explicit "push back if this is off," making them reversible.

---

## Key Definitions

| Term | Definition |
|---|---|
| Functional requirement | A "users should be able to…" statement describing a core feature the AI system must deliver (e.g. answer, cite, abstain, escalate). |
| Non-functional requirement | A "the system should be…" statement about a quality attribute (latency, availability, hallucination rate, cost), quantified wherever possible. |
| Hallucination tolerance | The acceptable rate/severity of confidently wrong or ungrounded answers; the single most design-shaping accuracy requirement for generative AI systems. |
| Grounding | Constraining generated answers to information present in retrieved context, so the model cannot invent facts; enables citations and abstention. |
| Freshness (staleness bound) | The maximum acceptable age of the content backing an answer; determines batch vs incremental/streaming indexing. |
| Data residency | A constraint on where data may be physically processed/stored (region, VPC, tenant boundary); decides hosted-API vs self-hosted model. |
| Peak QPS | Highest expected queries per second (average QPS × peak factor); drives replica count, caching, and whether the vector index must shard. |
| Explicit assumption | A concrete value the candidate declares to bound an open variable, verbalized so the interviewer can veto it. |
| The "hard part" | The single load-bearing technical challenge a prompt hides (e.g. grounding for RAG, bounded agency for agents) that should dominate deep dives. |

---

## Summary / Quick Recall

- The vague prompt is a scoping negotiation, not a spec — clarifying is the graded skill.
- Split requirements into functional ("users should be able to…") and non-functional ("system should be…"), prioritize top 3 / top 3–5, and quantify.
- For AI, hallucination tolerance, data sensitivity, and freshness are the requirements that fork the architecture — never skip them.
- Ask a question only if you can name the design decision its answer changes.
- Do scale math only when the number changes a decision (e.g. does the vector index fit one node?); otherwise assume large scale and move on.
- State assumptions out loud and in writing so they're reversible; silent assumptions are the classic failure.
- Name the hard part early (grounding for RAG, tool reliability/bounded agency for agents) to set the deep-dive agenda.

---

## Self-Check Questions

1. In the requirements phase of an AI system design interview, what is the difference between a functional and a non-functional requirement, and give one AI-specific example of each?

   <details><summary>Answer</summary>

   A functional requirement is a "users should be able to…" statement (a feature), e.g. "users should get a citation to the source doc." A non-functional requirement is a "the system should be…" statement about a quality attribute, ideally quantified, e.g. "grounded-hallucination rate below target" or "p95 < 3s." The tempting error is to list only functional features and treat accuracy as implied — for AI systems, quantifying hallucination tolerance (a non-functional requirement) is what actually shapes the architecture, so omitting it is a serious miss.

   </details>

2. The interviewer says "design a chatbot for our internal HR docs" and nothing else. You have time for only three clarifying questions before designing. Which three do you ask and why?

   <details><summary>Answer</summary>

   Strong picks: (1) *hallucination tolerance / must answers be grounded + cited?* — forks RAG-with-abstention vs free generation; HR answers being wrong is high-risk. (2) *Can the data (docs + user questions) leave our infra / go to a hosted API?* — internal HR data implies PII and residency limits, forking hosted vs self-hosted (OWASP LLM02). (3) *How fresh must answers be?* — forks nightly batch vs streaming re-index. "How many users?" is weaker here because an internal HR tool is almost certainly low-QPS, so the answer rarely changes the design — asking it wastes one of your three slots.

   </details>

3. **Which TWO** of the following back-of-the-envelope calculations are worth doing during the interview because they directly change a design decision?
   - A. Total marketing spend implied by the user base
   - B. Number of embedding chunks vs a single vector node's memory capacity
   - C. Peak QPS vs the throughput of one model replica
   - D. The exact floating-point precision of cosine similarity
   - E. Total lines of code in the ingestion pipeline

   <details><summary>Answer</summary>

   **B and C.** B decides whether the vector index fits on one node or must be sharded/quantized — a real architectural fork. C decides how many replicas and whether you need an inference queue or caching. Both change the design, which is the delivery framework's test for whether a calculation is worth doing. A is a business/finance figure irrelevant to the system design. D is a fixed implementation detail that doesn't fork the architecture. E (the tempting one, because it sounds like "estimation") measures effort, not a design constraint, and changes nothing about the system's shape.

   </details>

4. A candidate assumes the support docs are public and designs a hosted-API RAG system. Twenty minutes in, the interviewer notes the docs contain confidential internal pricing. What did the candidate do wrong, and how should assumptions have been handled?

   <details><summary>Answer</summary>

   The candidate made a *silent* assumption about data sensitivity — the single most consequential AI-specific requirement for the hosted-vs-self-hosted fork. Because it was never verbalized, the interviewer couldn't veto it, and the whole design now needs rebuilding under time pressure. The correct handling is to state assumptions explicitly and in writing ("assuming docs are public and can go to a third-party API — push back if not"), making the assumption reversible before it drives 20 minutes of design. This maps to OWASP LLM02 Sensitive Information Disclosure. The subtle wrong answer — "the candidate should have known HR docs are private" — misses the point: the fix isn't domain omniscience, it's *surfacing* the assumption so it can be corrected.

   </details>

5. You're given "build an agent that books business travel for employees." Compare naming the hard part as (a) "serving the LLM at scale" vs (b) "reliable tool use and bounded agency," and justify which better positions the rest of the interview.

   <details><summary>Answer</summary>

   (b) is the correct hard part. A travel-booking agent is low-QPS (employees book occasionally), so LLM serving scale is not load-bearing — choosing (a) sends the deep dive toward a non-problem. The genuine difficulty is that the agent takes *actions* (searching, holding, purchasing) via tools, where a wrong or hallucinated tool call spends real money or books the wrong flight — this is OWASP LLM06 Excessive Agency, and it demands guardrails: confirmation steps, permission scoping, idempotency, and human-in-the-loop for high-stakes actions. Naming (b) early lets you allocate deep-dive time to tool reliability and error recovery, which is exactly the staff-level problem navigation interviewers reward; naming (a) signals you pattern-matched to generic scaling and missed what makes *this* problem interesting.

   </details>

---

## Further Reading

- [System Design Delivery Framework — Hello Interview (System Design in a Hurry)](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery) — *verified 2026-07-29* — Canonical functional vs non-functional requirements split, the non-functional checklist, and guidance on when capacity estimation is worth doing.
- [ML System Design Delivery Framework — Hello Interview](https://www.hellointerview.com/learn/ml-system-design/in-a-hurry/delivery) — *verified 2026-07-29* — Problem-framing phase: clarifying an ambiguous prompt, targeted questions, and turning a business objective into a scoped ML objective.
- [ML System Design in a Hurry: Introduction — Hello Interview](https://www.hellointerview.com/learn/ml-system-design) — *verified 2026-07-29* — Interview assessment rubric, including "Problem Navigation" — framing a vague business goal as a measurable ML problem.
- [Building Effective Agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — *verified 2026-07-29* — Grounds the "when is complexity warranted" and agentic-vs-workflow scoping decisions that clarifying questions must surface.
- [OWASP Top 10 for LLM Applications (2025) — OWASP GenAI Security Project](https://genai.owasp.org/llm-top-10/) — *verified 2026-07-29* — The risk taxonomy (LLM02 Sensitive Information Disclosure, LLM06 Excessive Agency, LLM09 Misinformation, LLM10 Unbounded Consumption) behind the data-sensitivity, agency, hallucination, and cost clarifying dimensions.
