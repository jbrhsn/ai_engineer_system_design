# Delivery Framework for AI System Design Interviews — Interview Prep

**Section:** Interview Practice — Case Studies & Drills → Delivery Framework for AI System Design Interviews | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

This chapter is about the *meta-skill* of delivering an AI system design interview well — not any one architecture. The questions below therefore test whether you can **scope, sequence, articulate trade-offs, and drive a 45-minute conversation**, using a real RAG/agentic prompt only as the vehicle. Treat every answer as an audition for "would I want this person leading a design review?"

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| How do you structure a 45-minute AI system design answer? | Named phases in order — scoping → high-level → data/model → inference & eval → deep dives → wrap-up; time-box (scoping ~6 min, high-level ~3 min, protect the deep dive ~10–15 min); each phase hands an artifact to the next (objective → diagram → component to deep-dive); announce the phase out loud to signal driving. | "I'd start by drawing the architecture" — jumps to boxes before an agreed objective, so there is nothing for the design to be correct against; also gives no time budget, guaranteeing the deep dive gets squeezed. |
| Which clarifying questions matter most for an *AI* system, and why those over generic ones? | Each question must be a *design fork*, not small talk: hallucination tolerance (RAG-with-abstention vs free generation), data sensitivity/residency (hosted API vs self-host, OWASP LLM02), freshness (batch vs streaming re-index), p95 latency budget, cost ceiling per query (LLM10), online vs batch. Tie each answer to the decision it changes. | Asking "how many users?" with no fork attached, then concluding "so it's at scale." Generic capacity questions rarely move an AI design; skipping hallucination tolerance — the single most design-shaping non-functional requirement — is the fatal omission. |
| How do you articulate a trade-off so it reads as engineering, not guessing? | Three-part statement: **benefit gained + cost sacrificed + the stated constraint that justifies it** ("agentic RAG buys multi-hop accuracy but costs ~15× tokens and latency; I accept it because the SLA is 10s not 2s"). Bonus senior signal: state what you'd do if the constraint changed. Compare 2–3 concrete options against the *actual* budget. | Naming only the benefit ("I'll use a reranker") or hedging "it depends" without finishing the sentence. A bare choice invites a "why not just…?" probe you've left unanswered and gives zero signal you evaluated alternatives. |
| How does AI system design differ from classical system design? | Five concerns move from footnote to first-class: **evaluation** (correctness is probabilistic → needs an eval harness, not an assertion), **hallucination control** (confident-but-wrong is a distinct failure mode → grounding/citations/abstention), **cost-per-token** (agents ~4×, multi-agent ~15× a chat → a first-order scaling variable), the **data flywheel** (logs → labels → model, with feedback-loop bias risk), and **guardrails** (prompt injection LLM01, excessive agency LLM06). Raise them *unprompted*. | Treating eval, hallucination, cost, and security as "nice-to-have add-ons" the way they'd be optional in a CRUD service — reads as "only built demos." Also: assuming "accuracy" is a single implied requirement instead of a quantified tolerance. |
| How do you handle interviewer pushback ("why not strong consistency here?") without losing points? | A probe is a collaboration signal, not a trap — the interviewer wants a specific piece of signal. Two winning responses: **defend** with a reason tied to a requirement, or **concede explicitly** ("good point — the user must see their own upload, let me revise") and update. Absorb hints, thank, continue. | Getting defensive and digging in ("no, my choice is right, moving on"), or silently redrawing without explaining why. Stubbornness and silent capitulation both fail — the first looks like ego, the second hides your reasoning. |
| When do you ask a clarifying question versus state an assumption and move on? | Ask when the answer *changes the design* (real-time vs batch, data residency, hallucination tolerance). State-and-write an explicit, reversible assumption when it only changes a number ("assume ~50k employees, English v1, daily freshness — push back if off"). Silent assumptions are the failure mode because the interviewer never gets to veto them. | "I'll just assume it's at scale" said only in your head — a silent assumption looks identical to an explicit one on the board, but you may design 20 minutes for the wrong constraint before the interviewer corrects you. |

---

## Applied / Scenario Questions

**Q1:** You're 10 minutes into designing a RAG assistant when the interviewer says *"assume infinite scale."* What do you do?

**Strong answer framework:**
- Don't treat it as license to skip estimation — recognize it as a prompt to name the concerns that *change* at scale and design for them explicitly, rather than hand-wave "so it's distributed."
- Convert "infinite scale" into the specific forks it triggers: vector index no longer fits one node → shard or product-quantize; peak QPS forces horizontal LLM replicas + an inference queue + embedding cache; cost-per-token now dominates the bill, so route easy queries to a small model and cap agent-loop iterations.
- Re-anchor to the objective so scale doesn't derail the load-bearing discussion: "At that scale the *hard part* is still grounding quality; scale changes my serving tier and index sharding but not the retrieval-quality-vs-hallucination problem I want to deep-dive."
- **Explicit trade-off:** state the latency-vs-accuracy-vs-cost-vs-safety balance the new constraint forces — "at infinite scale I'd bias toward a smaller/faster model with aggressive caching (protects latency and cost), accept slightly lower single-query accuracy, and keep the abstention guardrail non-negotiable because a confident-wrong policy answer is a safety/compliance harm that doesn't get cheaper at scale."
- Keep driving: announce you'll note sharding + caching on the diagram and then return to the eval/guardrail deep dive, so the scale tangent doesn't consume the highest-signal phase.

**Q2:** Mid-answer you realize your first design (a full multi-agent orchestrator) won't meet the stated p95 latency budget. How do you recover?

**Strong answer framework:**
- Self-correct *out loud* immediately — audible self-correction is a green flag for driving and structured thinking, not a weakness: "I proposed multi-agent orchestration, but at ~25s that breaks our 10s p95 — let me step back and re-pick against the constraint."
- Re-run the option comparison against the *actual* budget instead of defending the sunk choice: naive RAG (~1.5s, cheap, but fails multi-hop), corrective RAG capped at two re-queries (~6s, ~$0.03, fixes multi-hop), full multi-agent (~25s, ~$0.40, over budget).
- Commit to the middle option and say what you're giving up: "corrective RAG is the only option inside both the latency and cost ceiling while lifting accuracy on hard queries; I give up ~4.5s over naive RAG, which the 10s SLA permits."
- **Explicit trade-off:** name latency-vs-accuracy-vs-cost-vs-safety in one breath — "I'm trading peak multi-agent accuracy for a latency and cost profile that meets the SLA, while keeping the grounding/abstention guardrail so the accuracy I do give up never turns into an unsafe confident-wrong answer."
- Frame the recovery as a feature: "In production I'd cap the agent loop and alert if p95 drifts — the same discipline I'm applying live here." This turns a stumble into evidence of judgment.

---

## System Design / Architecture Questions

**Q:** *"Design a RAG assistant that answers employee questions from our internal policy documents."* — but the graded skill here is **how you deliver the answer**, so walk the interviewer through your delivery sequence, not just the boxes.

**Approach:**

1. **Clarify requirements (scope before drawing, ~6 min).** State up front "before I design anything, let me scope so I don't build the wrong thing," then ask only fork-changing questions: who asks (all employees vs a support team)? corpus size and change rate? is a confident-but-wrong policy answer dangerous? real-time chat or batch? can internal data hit a third-party API? Convert answers into a *written, vetoable* requirements artifact — top-3 functional (answer grounded in docs; cite source; abstain/escalate when retrieval is weak) and top-3–5 non-functional, quantified (p95 < 3s; ~4 QPS peak; grounded-only with a hallucination-rate target; PII redacted before any 3rd-party call; daily freshness; cost < $0.03/query). Name the business objective (deflect HR/IT tickets *without* wrong guidance) and the AI objective (retrieve grounded passages → cited answer, else abstain). Close gaps with explicit assumptions ("~50k employees, English v1 — push back if off") and name the **hard part** — grounding + abstention, not serving scale.

2. **Propose high-level architecture (~3 min, rough, don't beautify).** Announce the phase, then sketch the whole lifecycle as a communication aid: ingest → chunk → embed → vector index → retrieve → rerank → grounded generate → cite, with a PII-redaction step before any external model call and an abstention gate before the answer returns. State that the diagram is disposable and you'll return to harden one component. This is where you *drive* — say "next I'll pick one component to go deep on; I think it's retrieval-quality-vs-hallucination — stop me if you'd rather steer elsewhere."

3. **Justify choices and name trade-offs explicitly (the deep dive, ~15 min — where depth is scored).** Commit to concrete choices and defend each as a three-part trade-off tied to the stated constraints: choose RAG over fine-tuning (docs change weekly; RAG re-indexes cheaply and gives citations for free) over pure generation (unacceptable hallucination risk). On the precision/recall axis, bias recall (generous top-k) then rerank to top-5 — "a missed policy clause is a wrong answer; +300ms of reranker is cheap insurance under a 3s budget." Specify the eval harness (context precision/recall + faithfulness offline, thumbs + escalation-rate online) and tie every metric to the business objective. Set the abstention threshold. Raise **guardrails unprompted** — prompt injection via poisoned documents (OWASP LLM01/LLM08) and sensitive-info disclosure (LLM02) — and the **data-flywheel** feedback-loop risk. Wrap up: recap decisions + the single biggest risk (confident-wrong policy answers) + what you'd do next with more time. The explicit **latency-vs-accuracy-vs-cost-vs-safety** balance is stated at each fork, not deferred.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **Functional vs non-functional requirements** — use in scoping to structure the two labelled lists ("users should be able to…" vs "the system should be…, quantified"); signals you scope before you draw.
- **p95 latency budget** — use instead of "low latency"; a quantified percentile ties every serving/rerank/agent-loop decision to a concrete ceiling and shows you think in SLOs.
- **Hallucination tolerance** — use in scoping to quantify accuracy for a generative system; it's the fork between RAG-with-abstention and free generation, and naming it early shows AI-specific maturity.
- **Back-of-envelope estimation** — use only when the number changes a decision ("15.6M chunks at 1536-dim float32 ≈ 96GB, so one node won't hold it — I'll shard"); signals you estimate to a purpose, not by ritual.
- **"The hard part"** — use right after requirements to set the deep-dive agenda ("the interesting part is grounding, not the chat UI"); signals staff-level problem navigation.
- **Defense in depth / guardrails** — use in the deep dive when layering PII redaction + injection filtering + abstention + human-in-the-loop; signals you treat security (OWASP LLM Top 10) as first-class.
- **Driving the whiteboard** — the behaviour, not the phrase: announce the next phase, propose + name the trade-off, draw, pause for a probe, loop; signals you can lead a design review unprompted.
- **Grounding / abstention** — use to describe constraining generation to retrieved context and returning "I don't know"; signals you design against confident-wrong answers, not just for happy-path answers.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"It depends"** (with no dependency named) — hedging to avoid being wrong; reads as not knowing *what* it depends on. Always finish the sentence: "it depends on the latency budget — under 2s I do X, over 10s I do Y."
- **"I'd just use GPT-4 (or the biggest model) for everything"** — signals no cost-per-token awareness and no routing instinct; ignores that a small model for easy queries protects both cost and latency, and that model choice is a trade-off, not a default.
- **Jumping to architecture before scoping** ("So I'll use a vector DB with 512-token chunks and top-k of 5…") — boxes before an agreed objective; the design has nothing to be correct against and triggers red flags on structured thinking.
- **"We can scale later" / "assume it's at scale"** (unqualified) — waves away the exact estimation that forks the design (does the index fit one node? how many replicas?); reads as avoiding the analysis rather than doing it.
- **"The system just needs to be accurate / fast / reliable"** — unquantified quality attributes are meaningless for AI; without a hallucination tolerance and a p95 budget there's no design constraint to reason against.
- **"Eval / security we can add at the end"** — treats first-class AI concerns as QA afterthoughts; for generative systems, raising them unprompted *is* the depth signal, so deferring them reads as demo-only experience.
- **Silence after each box** (waiting to be led) — at senior level the interviewer's silence means *you* pick up the marker; passivity forfeits the "driving the conversation" signal.

---

## STAR Answer Frame

**Situation:** In a design review for a production RAG assistant over internal engineering docs, the team was split — half wanted a full multi-agent orchestrator for "best accuracy" on multi-hop queries, and the meeting was stalling into competing opinions with no shared frame.

**Task:** As the applied AI engineer leading the session, I had to drive the room to a defensible decision under a stated p95 ≤ 10s SLA and a ≤ $0.05/query budget — without letting the loudest preference win.

**Action:** I reset the discussion into the delivery sequence I use in interviews: I re-stated the objective and the two binding non-functional constraints on the board, then put three options into a decision matrix — naive RAG (~1.5s, ~$0.005, fails multi-hop), corrective RAG capped at two re-queries (~6s, ~$0.03, recovers on bad first retrieval), and full multi-agent (~25s, ~$0.40, over both budgets). I articulated each as an explicit trade-off (benefit, cost, the constraint that justified it), showed multi-agent broke both ceilings, and committed to corrective RAG — naming out loud the ~4.5s and ~6× cost I was giving up over naive RAG, which the SLA permitted. I also raised prompt-injection and abstention guardrails unprompted so accuracy gains never became unsafe confident-wrong answers.

**Result:** The team aligned on corrective RAG in the same meeting. In production it held p95 at ~6.3s and ~$0.028/query (inside both budgets), lifted answer faithfulness on multi-hop queries by ~22 points on our offline eval set versus naive RAG, and the capped loop kept token spend ~14× below the multi-agent estimate — the decision framing itself, not just the architecture, was what unblocked the room.

---

## Red Flags Interviewers Watch For

Specific to interview *delivery* — these are the behaviours that cost you the hire signal regardless of how much you know:

- **Poor time management** — burning 15–20 minutes enumerating retrieval features or perfecting the diagram, so the deep dive (where depth is scored) never happens. The clock is part of the exam; overspending early *mathematically* forfeits the highest-signal phase.
- **No trade-offs stated** — asserting choices bare ("I'll use agentic RAG," "self-host it") with no benefit/cost/constraint. Reads as buzzword-reaching; the interviewer can't tell whether you evaluated the alternative or pattern-matched to the fanciest option.
- **Not driving** — going silent after each box and waiting to be asked the next question. At applied/senior level the interviewer expects you to announce the next phase and lead; passivity reads as "can't run a design review."
- **Ignoring the hard part** — sending the deep dive toward a non-problem (e.g. "serving the LLM at scale" for a low-QPS internal tool) instead of the load-bearing difficulty (grounding for RAG, bounded agency for agents). Signals you can't identify what makes *this* problem interesting.
- **Skipping AI-specific concerns** — never mentioning how you'd *measure* faithfulness, stop a confident-wrong answer, cost a query, or defend against prompt injection. Omitting eval/hallucination/cost/guardrails reads as "only built demos."
- **Making silent assumptions** — designing against an unstated constraint (public docs, English-only, hosted API) the interviewer never got to veto, then rebuilding under time pressure when it's revealed. Always verbalize and write assumptions as reversible.
- **Talking over the interviewer** — over-correcting on "be proactive" by narrating non-stop and never pausing; you miss the specific probes that carry their scoring signals. Drive, then pause to invite the probe.
- **Treating pushback as an attack** — getting defensive and digging in on a challenged choice instead of defending with a reason or conceding explicitly and revising. Stubbornness reads as ego; a probe is a collaboration signal.
