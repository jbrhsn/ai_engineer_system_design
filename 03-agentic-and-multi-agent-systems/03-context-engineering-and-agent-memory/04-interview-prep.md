# Context Engineering & Agent Memory — Interview Prep

**Section:** Agentic and Multi-Agent Systems | **Role target:** Applied AI Engineer / AI Systems Engineer (agentic AI, RAG, multi-agent production systems)

---

## Core Conceptual Questions

These test whether you understand the fundamentals.

| Question | Key points to cover | Common weak-answer trap |
|---|---|---|
| What is context engineering, and how is it broader than prompt engineering? | Prompt engineering optimizes the *wording* of a mostly-static instruction. Context engineering is the discipline of curating the **entire token payload** that enters the model on every turn — system prompt, tool definitions, retrieved documents, conversation history, memory, and tool results — treating the context window as a **finite, degrading resource** to be budgeted. It spans four moves: **write** (persist state outside the window), **select** (pull in only what's relevant, just-in-time), **compress** (summarize/compact), and **isolate** (split context across sub-agents/scratchpads). It's a systems/state-management problem, not a copywriting problem. | Saying context engineering is "just writing better prompts" or "prompt engineering with more examples" — collapses a state/budget-management discipline into wording, and shows no awareness of the window as a scarce resource. |
| What is "context rot" / "lost-in-the-middle," and why does it change how you fill the window? | Model performance **degrades as the context grows**, and information placed in the **middle** of a long context is recalled less reliably than content at the start or end. So more tokens ≠ better answers — a bloated window adds distractor noise, raises cost/latency, and can *lower* accuracy. Implication: curate aggressively, put the highest-signal content at the edges, and stop treating "fits in the window" as "the model will use it well." | "Just use a bigger context window / a long-context model and dump everything in" — ignores that recall degrades with length and mid-context, and that cost/latency scale with tokens. Treats the window as free memory. |
| Compare compaction (summarization) vs. trimming/pruning vs. note-taking for managing a long conversation. When do you reach for each? | **Trimming/pruning** removes messages by rule (e.g. keep last N turns / token budget via `trim_messages`, or drop specific messages via `RemoveMessage`) — cheap, lossy, no model call, good for stale chatter. **Compaction/summarization** replaces a span of history with a model-generated summary (a summarization node/middleware) — preserves gist at a token cost and an extra model call, but **loses fidelity** and can drop details the agent later needs. **Note-taking/scratchpad** writes durable facts/goals to **external state** so they survive outside the window and can be re-selected on demand — highest fidelity for what you choose to persist, but you must decide what to write and when to read it back. | Treating them as interchangeable, or "just summarize everything when it gets long" — ignores that summarization is lossy and can compress away the actual task goal, and that trimming and note-taking are cheaper/higher-fidelity for different needs. |
| Distinguish short-term (thread-scoped) from long-term (cross-thread) agent memory, and how each is implemented. | **Short-term / thread-scoped**: the state of one conversation, persisted by a **checkpointer** keyed by `thread_id` — survives interruptions/resume within that thread, bounded by the context window and eventually compacted/trimmed. **Long-term / cross-thread**: durable knowledge that outlives any single thread, held in a **Store** (`BaseStore` / `InMemoryStore`) organized by **namespaces** (e.g. per-user), written/read via `put`/`get`/`search`, often with **semantic search** over embeddings for retrieval. Short-term is "what happened in this chat"; long-term is "what I know about this user/task across all chats." | Conflating the two ("memory is just the message history"), or thinking a bigger checkpointer gives cross-session memory — thread state doesn't carry across threads without an explicit Store. |
| Name the memory *types* (semantic / episodic / procedural) and the hot-path vs. background writing trade-off. | **Semantic** = facts ("user prefers metric units"); **episodic** = past experiences/events ("last session we debugged the auth flow"); **procedural** = how-to/behavioral rules the agent follows (often refined system-prompt instructions). **Hot-path writing** commits memory synchronously during the turn — immediately available but adds latency and can write noise. **Background writing** extracts/consolidates memory asynchronously after the turn — no user-facing latency, but there's a **staleness window** where new facts aren't yet retrievable. You also need **selection/retrieval** discipline (only surface relevant memories) plus **privacy and staleness** handling (consent, TTL/expiry, updating superseded facts). | Listing "memory" as one undifferentiated bucket, or "just store everything the user ever says" — ignores memory types, retrieval relevance, write cost, and privacy/staleness; persistent memory is neither free nor automatically correct. |

---

## Applied / Scenario Questions

### Q1 — A long-running LangGraph support agent degrades over long sessions: answers get vaguer, latency and token cost climb, and it sometimes "forgets" the customer's original request.

**Strong answer framework:**
- **Name the root cause as context, not the model:** the window is filling with full tool results and stale turns; **context rot / lost-in-the-middle** means the original request (now buried mid-context) is recalled poorly while cost/latency scale with the token count. This is a budgeting failure.
- **Layer the four moves rather than one blunt fix:** **trim** stale chatter with a token-budgeted `trim_messages` policy (cheap, no model call); **compact** older spans via a summarization node once a threshold is crossed; **write** the durable anchors — the customer's goal, key IDs, decisions — to a **scratchpad / external state** so they survive compaction and can be re-selected; **offload/truncate** large raw tool results (store full payload externally, keep a short reference in-context).
- **Trade-off framing — fidelity vs. tokens:** compaction buys token headroom but is **lossy**; guard against summarizing away the goal by pinning it in note-taking (highest fidelity for what matters) and only summarizing the low-signal middle. Trimming is cheapest but irreversibly drops detail; the scratchpad recovers what trimming/compaction would lose.
- **Retrieval-noise awareness:** don't re-inject everything you saved — **select just-in-time** only the notes/memories relevant to the current turn, or you re-create the bloat you were fixing.
- **Verify:** set an explicit per-turn token budget, log context composition (system / tools / history / retrieved / results), and eval task-success on a replay set of long sessions before/after to confirm the goal-forgetting stops without a quality regression.

### Q2 — Product wants the assistant to "remember users across sessions" — preferences, past issues, and how the user likes answers formatted. What do you build, and what are the risks?

**Strong answer framework:**
- **This is long-term, cross-thread memory — separate it from thread state:** thread-scoped checkpointer state won't carry across sessions. Add a **Store** with a **per-user namespace**; write **semantic** facts (preferences), **episodic** records (past issues), and optionally **procedural** rules (preferred answer format) via `put`, retrieve via `get`/`search` (semantic search for the fuzzy cases).
- **Choose the write path deliberately:** prefer **background/asynchronous** consolidation after the turn so you don't add hot-path latency, and so a model can dedupe/extract clean memories rather than dumping raw utterances — accept the **staleness window** (a just-stated fact may not be retrievable next turn) or do a targeted hot-path write for facts needed immediately.
- **Retrieve selectively, not exhaustively:** inject only the top-relevant memories for the current turn; flooding context with every stored memory reintroduces **retrieval noise** and context rot, and costs tokens/latency every call.
- **Trade-off framing — cost/latency/privacy vs. recall:** more persisted memory + longer retrieved context improves personalization but raises token cost, latency, and **privacy exposure**. Persistent memory is **not free**: handle consent, PII minimization, deletion/right-to-be-forgotten, TTL/expiry, and **staleness** (update or supersede facts when they change, so the agent doesn't act on an outdated preference).
- **Verify:** measure personalization lift against added token/latency cost, and test that stale/superseded facts are correctly overwritten and that deletion actually purges the namespace.

---

## System Design / Architecture Questions

### Q — Design a long-running personal-assistant agent that remembers users across many sessions without context overflow.

**Approach:**

1. **Clarify requirements.** How long do sessions run and how many turns; latency budget (interactive); how much personalization is expected (preferences, history recall); data sensitivity (PII, consent, deletion/retention requirements); is resume-after-interruption needed within a session; roughly how many users (namespace/store scale); acceptable staleness for newly-learned facts; hallucination tolerance for recalled "memories."
2. **High-level architecture.**
   - **Short-term / thread memory:** a **checkpointer** keyed by `thread_id` holds the live conversation state so a session is durable and resumable (maps cleanly to a Postgres-backed checkpointer on a FastAPI/PostgreSQL stack).
   - **Context-budget controller on the hot path:** enforce a per-turn token budget — **trim** stale turns (`trim_messages` / `RemoveMessage`), **compact** older spans through a summarization node once a threshold trips, and **offload** large tool results to external storage keeping only a short reference in-context.
   - **Scratchpad / note-taking:** pin the session's durable anchors (current goal, key entities/IDs, open decisions) to external state so compaction/trimming never erases them; re-select them each turn.
   - **Long-term / cross-thread memory:** a **Store** (`BaseStore`) with **per-user namespaces**; hold **semantic** (preferences/facts), **episodic** (past sessions/issues), and **procedural** (answer-format rules) memories; write via `put`, retrieve via `get`/`search` with **semantic search** for fuzzy recall.
   - **Memory write path:** **background consolidation** after each turn extracts and dedupes durable facts into the Store (keeps the hot path fast); a small set of must-have-now facts may be written hot-path.
   - **Retrieval/selection layer:** at the start of each turn, select only the top-relevant long-term memories + scratchpad notes for *this* query and inject them near the edges of context (not the middle).
   - **Privacy/governance:** consent flags per namespace, PII minimization, TTL/expiry, supersede-on-change for stale facts, and a hard-delete path that purges a user's namespace.
3. **Justify choices and name trade-offs.**
   - **Two memory tiers (checkpointer + Store):** thread state can't cross sessions; a separate cross-thread Store is the only way to get durable personalization. Trade-off: more moving parts and a retrieval layer to maintain vs. real cross-session memory.
   - **Compaction + note-taking together:** compaction reclaims tokens but is **lossy**; the scratchpad preserves the goal/anchors at high fidelity so we don't summarize away the task. Trade-off: extra model call for summarization and explicit write/read logic vs. bounded context and preserved intent.
   - **Selective just-in-time retrieval over "inject all memory":** avoids **context rot / lost-in-the-middle** and controls token cost/latency. Trade-off: a retrieval miss can drop a relevant memory — mitigate with semantic search + a small recency/importance heuristic.
   - **Background over hot-path writes:** keeps interactive latency low and yields cleaner memories, at the cost of a **staleness window**; hot-path writes reserved for facts needed immediately.
   - **Privacy is a first-class constraint:** persistent memory creates retention, consent, and staleness liabilities — TTL, supersede-on-change, and true deletion are part of the design, not an afterthought.
   - **Verify:** eval personalization lift vs. added token/latency cost on a multi-session replay set; test goal-retention across compaction, retrieval relevance, stale-fact overwrite, and namespace deletion.

---

## Vocabulary That Signals Expertise

Use these terms naturally — don't force them:

- **Context engineering** — when framing the problem as curating the whole token payload and budgeting a finite window, not wording a prompt.
- **Context window as a finite, degrading resource** — when justifying aggressive curation instead of "dump everything in."
- **Context rot / lost-in-the-middle** — when explaining why more tokens (and mid-context placement) can *lower* accuracy.
- **Write / select / compress / isolate** — the four context-management moves, when structuring a strategy.
- **System-prompt altitude** — when discussing pitching instructions at the right level of specificity (not too rigid, not too vague).
- **Tool curation / just-in-time context** — when explaining that you load only the tools and info relevant to the current step.
- **Compaction / summarization loop** — when reclaiming tokens by summarizing older history, and naming its fidelity cost.
- **Trimming / pruning (`trim_messages`, `RemoveMessage`)** — when removing messages by budget/rule cheaply and losslessly-of-model-call.
- **Tool-result truncation / offload** — when keeping a reference in-context and storing the full payload externally.
- **Note-taking / scratchpad to external state** — when persisting goals/anchors so they survive compaction.
- **Thread-scoped (short-term) vs. cross-thread (long-term) memory; checkpointer / `thread_id` vs. Store / namespaces** — when separating session state from durable personalization.
- **Semantic / episodic / procedural memory** — when classifying *what* is being remembered.
- **Hot-path vs. background writes** — when trading write latency against a staleness window.
- **Retrieval / selection, semantic search over a Store** — when surfacing only relevant memories per turn.
- **Privacy / staleness (consent, TTL, supersede-on-change, deletion)** — when treating persistent memory as a governed, non-free resource.

---

## Vocabulary That Signals Weakness

Avoid these — they signal outdated or shallow understanding:

- **"Just use a bigger context window / long-context model and put everything in."** — Red flag: ignores context rot, lost-in-the-middle, and that cost/latency scale with tokens; treats the window as free memory.
- **"Context engineering is just prompt engineering / writing better prompts."** — Red flag: collapses a state- and budget-management discipline into wording; no notion of write/select/compress/isolate.
- **"Just summarize it when it gets long."** — Red flag: ignores that summarization is **lossy** and can compress away the actual task goal; no mention of trimming or note-taking as higher-fidelity/cheaper alternatives.
- **"Keep the whole history so it never forgets."** — Red flag: unbounded history guarantees context overflow, rot, and runaway cost; shows no budgeting instinct.
- **"Memory is just the message history."** — Red flag: conflates thread-scoped state with cross-thread long-term memory; doesn't know a Store/namespaces are needed for cross-session recall.
- **"Store everything the user ever says as memory."** — Red flag: no memory-type distinction, no retrieval relevance, ignores write cost, privacy, and staleness.
- **"Persistent memory is basically free."** — Red flag: ignores token/latency cost of retrieval, and the consent/PII/retention/staleness governance persistent memory demands.

---

## STAR Answer Frame

**Situation:** A production LangGraph assistant I owned on a FastAPI/PostgreSQL stack handled long, multi-turn sessions. Over long conversations answers drifted vaguer, per-session token cost climbed, and the agent would lose track of the user's original request — classic context bloat and rot.

**Task:** Keep long sessions coherent and bounded in cost/latency, and add durable cross-session personalization, without summarizing away the task goal or creating a privacy liability.

**Action:** I instrumented per-turn context composition (system / tools / history / retrieved / tool-results) and found raw tool payloads and stale turns dominating the window. I (1) added a per-turn token budget with `trim_messages`-style pruning of stale turns; (2) introduced a summarization/compaction node for older spans once a threshold tripped; (3) **pinned the goal and key entities to a scratchpad in external state** so compaction never erased intent, re-selecting them each turn; (4) offloaded large tool results to storage, keeping only a short in-context reference; and (5) added a cross-thread **Store** with per-user namespaces for semantic/episodic/procedural memories, written by a **background** consolidation step after each turn and retrieved selectively via semantic search — with TTL, supersede-on-change, and a namespace-delete path for privacy.

**Result:** Long-session token cost dropped by roughly half and the goal-forgetting behavior stopped, while task-success on a long-session replay set held flat within noise. Personalization measurably improved repeat-user satisfaction, and hot-path latency was unaffected because memory writes ran in the background — bounded context with durable, governed memory.

---

## Red Flags Interviewers Watch For

Specific to context engineering and agent memory:

- **No context budget** — proposing to just append everything to the window with no per-turn token budget or curation strategy; the clearest sign of no long-running-agent experience.
- **Unbounded history growth** — relying on "keep all history" with no trimming/compaction, guaranteeing overflow, context rot, and runaway cost.
- **Summarizing away the goal** — compacting the whole history indiscriminately with nothing pinned (no scratchpad), so the agent loses the task it was working on.
- **One undifferentiated "memory"** — no distinction between thread-scoped state and cross-thread Store, or between semantic/episodic/procedural memory, or between hot-path and background writes.
- **Storing everything as memory** — treating every utterance as durable memory with no relevance selection, so retrieval floods context and reintroduces rot.
- **Ignoring retrieval noise** — re-injecting all saved notes/memories every turn instead of selecting just-in-time, recreating the bloat the memory system was meant to solve.
- **Ignoring memory privacy/staleness** — no consent, PII minimization, TTL/expiry, supersede-on-change, or deletion path; and acting on stale/superseded facts.
- **No evaluation** — shipping without measuring goal-retention across compaction, retrieval relevance, or personalization lift vs. added cost/latency, so degradations go undetected.
