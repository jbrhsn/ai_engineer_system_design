# Agent Memory Architectures — Short-Term vs Long-Term

**Section:** 03 Agentic & Multi-Agent Systems → Context Engineering & Agent Memory | **Est. time:** 3 hrs | **Interview relevance:** High — "how does your agent remember things?" is a near-guaranteed follow-up, and the crisp answer is *two separate systems* (thread-scoped state vs cross-thread store), not "we keep the whole history."

---

## TL;DR

Agents have two distinct memory systems that beginners collapse into one. **Short-term (working / thread-scoped) memory** is the conversation state *within one thread*, held in the graph state and persisted by a **checkpointer** keyed by `thread_id`; it is bounded by the context window, which is exactly why you compact/trim it (chapter 02). **Long-term memory** is data that must survive *across* threads and sessions — user facts, past examples, learned instructions — persisted in a separate **Store** keyed by a `namespace` tuple, written deliberately and *retrieved on demand* rather than always resident. Long-term memory splits by *type* (semantic = facts, episodic = experiences/examples, procedural = how-to/instructions) and by *when you write it* (hot path during the turn vs background after). The hard part is not storage but **selection**: you must retrieve the *right* memories into the limited context window, which ties memory straight back to context engineering. **The one thing to remember: short-term memory is thread-scoped state behind a checkpointer; long-term memory is cross-thread data in a store that you must deliberately *write* and *selectively retrieve* — the model does not automatically remember anything past the context window.**

---

## ELI5 — Explain It Like I'm 5

Imagine a receptionist who meets clients all day. While one client is standing at the desk, the receptionist keeps everything about *this* conversation in their head — what was just said, what the client asked for two sentences ago. That's short-term memory: it's fast and effortless, but the moment the client walks away (the session ends) it's gone, and if the conversation runs too long the receptionist can only hold so much in their head at once. So the receptionist also keeps a **notebook** with one page per client, filed by name. After a client leaves, they jot down the useful things — "prefers morning appointments," "allergic to peanuts" — and next time that client returns, they flip to that client's page and read just the relevant lines back into their head before speaking. That notebook is long-term memory: it survives across every visit, but nothing on the page helps unless the receptionist *chooses to open the right page and copy the right lines* into their head. The common misconception is thinking the receptionist "just remembers everyone forever" automatically — they don't; the head has limited room, and everything durable only survives because it was deliberately written to the notebook and deliberately looked up again.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Distinguish short-term (thread-scoped, checkpointer-backed) memory from long-term (cross-thread, store-backed) memory and state which persistence primitive backs each.
- [ ] Explain the semantic / episodic / procedural taxonomy of long-term memory and give a concrete agent example of each.
- [ ] Implement long-term memory in LangGraph using a `BaseStore`/`InMemoryStore` with namespaces, `put`/`get`/`search`, and semantic search over memories.
- [ ] Decide when to write memories "in the hot path" versus "in the background," and articulate the latency/quality trade-off.
- [ ] Diagnose the memory *retrieval/selection* problem and connect it to context engineering, plus reason about privacy and staleness of persistent user memory.

---

## Visual Overview

### Short-Term (thread-scoped) vs Long-Term (cross-thread)

```
SHORT-TERM  (one conversation)              LONG-TERM  (across all conversations)
──────────────────────────────             ─────────────────────────────────────
 thread_id = "chat-1"                        namespace = ("user-42", "memories")
      │                                             │
      ▼                                             ▼
 ┌──────────────────┐                        ┌───────────────────────────┐
 │ graph STATE      │                        │  STORE  (BaseStore)        │
 │  messages[...]   │                        │  key → {value dict}        │
 │  files, artifacts│                        │  "likes window seats"      │
 └──────────────────┘                        │  "allergic to peanuts"     │
      │  persisted by                         │  "prefers terse answers"  │
      ▼                                        └───────────────────────────┘
 ┌──────────────────┐                              ▲            ▲
 │ CHECKPOINTER     │  scope: ONE thread           │            │
 │ (InMemorySaver / │  bounded by context window   thread-1   thread-2 ... (any thread,
 │  PostgresSaver)  │  → trim/compact (ch 02)      reads      same user_id can read it)
 └──────────────────┘
```

### The Long-Term Memory Taxonomy (CoALA mapping)

```
LONG-TERM MEMORY
├── SEMANTIC   → facts about the user/world      e.g. "user is vegetarian", "org uses GCP"
│               stored as a profile (1 doc, updated) OR a collection (many docs)
├── EPISODIC   → past experiences / examples     e.g. a successful past task trajectory
│               usually surfaced as few-shot examples selected by similarity
└── PROCEDURAL → learned instructions / how-to   e.g. an updated system prompt / rules
                usually a reflection loop rewrites the agent's own instructions
```

### Hot-Path vs Background Memory Writing

```
HOT PATH  (during the turn)                 BACKGROUND  (after / async)
─────────────────────────────              ──────────────────────────────
user msg ─► agent reasons ─► decides        user msg ─► agent answers ─► done
           to save memory ─► store.put                        │
           ─► THEN answers user                               ▼ (separate process, later)
                                                       summarize thread ─► store.put
 + memory available immediately             + zero added latency on the turn
 + user-visible ("I'll remember that")      + agent focuses on the task
 − adds latency & tool-choice complexity    − new memories lag; must decide trigger timing
```

### Write → Store → Later Retrieve → Select-Into-Context

```
   TURN A (thread-1)                    TURN B (thread-2, days later)
   ─────────────────                    ─────────────────────────────
   extract memory                       store.search(ns, query=user_msg, limit=k)
        │                                        │
        ▼                                        ▼  (semantic top-k, NOT everything)
   store.put(ns, key, value) ──────────►  selected memories
                                                 │
                                                 ▼
                                    inject into system prompt (bounded context)
                                                 │
                                                 ▼
                                    model.invoke([system+memories, *messages])
```

---

## Key Concepts

### Short-Term / Thread-Scoped (Working) Memory

**What it is.** Short-term memory is the state of a single ongoing conversation — the running message list plus any per-conversation data (uploaded files, retrieved docs, generated artifacts) — that lets the agent remember what was said earlier *in this same thread*.

**How it works mechanistically.** LangGraph manages short-term memory as part of the graph's **state**, and persists that state to a database via a **checkpointer** that saves a snapshot (checkpoint) at each super-step, organized under a **`thread_id`**. On each invocation you pass `{"configurable": {"thread_id": ...}}`; the checkpointer loads the thread's latest checkpoint at the start of the step and writes an updated one after, so follow-up messages sent to the same thread see the prior turns. Because this state is fed into the model each turn, it is **bounded by the context window** — which is precisely why long conversations require the trimming/summarization/compaction techniques from chapter 02, and why the checkpointer is the same primitive that enables human-in-the-loop and durable execution (section 02, chapter 03).

**Where it appears in real systems.** In LangGraph you enable it by compiling with a checkpointer: `graph = builder.compile(checkpointer=InMemorySaver())` for dev, or `PostgresSaver` / `SqliteSaver` / `RedisSaver` in production (`InMemorySaver` loses everything on process restart). You manage its size with `trim_messages`, `RemoveMessage`, or summarization inside a node. The conversation continuity you feel in ChatGPT is thread-scoped memory.

### Long-Term / Cross-Thread (Persistent) Memory

**What it is.** Long-term memory is application-defined data that must persist *across* threads and sessions — user preferences, accumulated facts, shared knowledge — recallable at any time from any thread, not just the one that created it.

**How it works mechanistically.** LangGraph persists long-term memory in a **Store** (interface `BaseStore`; `InMemoryStore` for dev, `PostgresStore`/`RedisStore`/etc. for prod), *separate* from the checkpointer. Memories are JSON documents organized under a **namespace** — a tuple of strings like `(user_id, "memories")` (a "folder") — and a **key** (a "filename"). You write with `store.put(namespace, key, value)`, fetch a specific item with `store.get(namespace, key)`, and enumerate/query a namespace with `store.search(namespace_prefix, query=..., limit=...)`. Crucially, unlike short-term state, long-term memory is **not automatically in context** — it lives outside the graph state and only enters a prompt when you explicitly retrieve it and inject it, which is the whole point: you keep the context window small and pull in only what's relevant.

**Where it appears in real systems.** Compile with `graph = builder.compile(store=store)` (often alongside a checkpointer). Inside a node you access it via the injected `Runtime` — `runtime.store.search((user_id, "memories"), query=..., limit=3)` — or via a `store: BaseStore` parameter. Namespaces are commonly scoped by `user_id` (per-user memory) or an org ID (shared memory); `store.search` supports prefix matching, `filter=` on content, and `limit`/`offset` pagination.

### The Memory Type Taxonomy — Semantic, Episodic, Procedural

**What it is.** A classification (borrowed from human psychology and mapped to agents by the CoALA paper) of *what kind* of thing a long-term memory holds: **semantic** = facts, **episodic** = experiences, **procedural** = instructions/rules.

**How it works mechanistically.** *Semantic* memory stores facts about a user or the world; it's managed either as a single continuously-updated **profile** (one JSON doc you overwrite each time — simple but error-prone as it grows) or a **collection** of many small docs (higher recall, but the model must decide when to insert vs update vs delete). *Episodic* memory stores past experiences and is typically surfaced as **few-shot examples**: relevant past input→output trajectories are selected by similarity and shown to "program" the model by example. *Procedural* memory stores the rules for performing tasks; agents rarely change weights or code, but they commonly **rewrite their own system prompt** via a reflection/meta-prompting loop that feeds the current prompt plus feedback back to the model to produce improved instructions. (Note: "semantic memory" the *type* is different from "semantic search" the *retrieval technique* — you can retrieve any memory type via semantic search.)

**Where it appears in real systems.** All three live in the same `Store` as namespaced JSON docs. ChatGPT's "save to memory" feature is semantic-profile/collection memory; few-shot example selection from a store or a LangSmith dataset is episodic; a "reflection" node that reads `store.get(("instructions",), ...)`, has the LLM revise it, and `store.put`s the new prompt back is procedural memory.

### Writing Memories — Hot Path vs Background

**What it is.** The two timing strategies for *when* the agent commits something to long-term memory: **in the hot path** (synchronously, during the user's turn) or **in the background** (asynchronously, in a separate process after the turn).

**How it works mechanistically.** *Hot path*: the agent (often via a dedicated `save_memory` tool it can choose to call) decides what to remember and writes it before responding. Pro: the memory is immediately available and the write can be made transparent to the user; con: reasoning about what to save adds latency and forces the agent to multitask between answering and memory management. *Background*: a separate task (triggered on a timer, a cron, after N messages, or manually) reads the finished conversation and extracts/summarizes memories to write. Pro: zero added latency on the response path and cleaner separation of concerns; con: new memories lag (other threads won't see them until the job runs) and you must design the trigger.

**Where it appears in real systems.** Hot-path is a `store.put` inside your `call_model` node (e.g. the docs' pattern of checking `if "remember" in last_message` then `runtime.store.aput(...)`), or ChatGPT's `save_memories` tool. Background is a separate LangGraph run / worker (the "memory-service" pattern) that consumes threads and writes to the same store.

### The Retrieval / Selection Problem (ties to context engineering)

**What it is.** The core difficulty of long-term memory: having stored many memories, you must **select the right subset** into the context window on each turn — because you cannot (and should not) inject them all.

**How it works mechanistically.** Storage is cheap; the context window is not. If you dump every memory into the prompt you reintroduce the exact overflow, cost, and "lost in the middle" distraction problems that motivated compaction. So retrieval typically uses **semantic search**: configure the store with an embedding model (`index={"embed": ..., "dims": ..., "fields": [...]}`), then `store.search(namespace, query=<current user message>, limit=k)` returns the top-*k* memories ranked by vector similarity to the query. You then format only those *k* into the system prompt. This is context engineering applied to memory: the *select* step (choosing what enters the limited window) is the same discipline from chapter 01, and getting `k` and the query right is what separates "personalized" from "noisy."

**Where it appears in real systems.** `InMemoryStore(index={"embed": init_embeddings("openai:text-embedding-3-small"), "dims": 1536})`, then `runtime.store.asearch(ns, query=state["messages"][-1].content, limit=3)`. Without the `index` config, `search` still works but only does insertion-order/`filter` listing, not similarity ranking.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Checkpointer backend (`InMemorySaver` vs `PostgresSaver`/`SqliteSaver`/`RedisSaver`) | Where short-term thread state is persisted and whether it survives restarts | Use `InMemorySaver` only for tests/notebooks; use a DB-backed saver in production — `InMemorySaver` loses all threads on process restart. |
| Store backend (`InMemoryStore` vs `PostgresStore`/`RedisStore`) | Where long-term cross-thread memory is persisted | `InMemoryStore` for dev; a DB-backed `BaseStore` implementation for production so memories survive and scale beyond one process. |
| Namespace scoping (e.g. `(user_id, "memories")` vs `(org_id, ...)`) | The isolation boundary of a memory — who can read it | Namespace per `user_id` for personal memory; per `org_id`/`team` for shared knowledge. Never share one namespace across users you must keep isolated (privacy leak). |
| `index` (embed model + `dims` + `fields`) on the store | Whether/what supports semantic search | Set it when you'll retrieve by *meaning* over a growing collection; `fields=["specific_field"]` (or `index=False` on a `put`) to embed only what's searchable and cut embedding cost/noise. |
| `limit` (top-k) on `store.search` | How many memories get retrieved into context per turn | Start small (3–5); raise only if recall is demonstrably low — every retrieved memory is input tokens and dilutes the context (the selection problem). |
| Write timing (hot path vs background) | When memories are committed | Hot path when the memory must be usable *this session* or shown to the user; background when response latency is critical and slight staleness is acceptable. |
| Memory TTL / staleness policy | How long a memory is trusted before refresh/expiry | Expire or timestamp-check volatile facts (job title, current project); keep stable facts (allergies, name) but always let newer statements override older ones. |

### Worked Example: Requirement → Decision

**Given:** You're building a personal travel-planning assistant. Users chat with it over many separate sessions across weeks. It must remember durable preferences ("I'm vegetarian," "I prefer aisle seats," "budget hotels only") so a brand-new conversation next month already knows them, *without* the user restating everything and *without* the assistant re-reading every past chat. Response latency matters (users expect a snappy chat), and the preferences are personal data.

- **Step 1 — Identify the goal:** Persist per-user preferences so they're available in *future, separate* conversations and used to personalize answers.
- **Step 2 — Define inputs:** The current user message; a per-user memory namespace `(user_id, "memories")`; an embedding model for semantic retrieval; a chat model.
- **Step 3 — Define outputs:** (a) new/updated preference memories written to the store; (b) on each turn, the top-k relevant preferences injected into the system prompt so the reply reflects them.
- **Step 4 — Apply constraints:** Cross-session survival ⇒ this is **long-term**, not short-term (short-term dies with the thread). Latency-sensitive ⇒ prefer **background** writing (or a lightweight hot-path save) so extraction doesn't slow replies. Personal data ⇒ **namespace per `user_id`** for isolation, plus a staleness rule so newer statements override older ones. Bounded context ⇒ retrieve **top-k via semantic search**, never all memories.
- **Step 5 — Select the approach:** Use a **`Store` (long-term memory), namespaced by `user_id`, holding semantic (profile/collection) memories**, written in the **background** and retrieved with `store.search(query=<user message>, limit=3)` injected into the system prompt. *Rationale vs alternatives:* a checkpointer alone (short-term) fails the core requirement because a new thread starts empty; stuffing all history into one never-trimmed thread (see anti-pattern below) overflows the context and costs a fortune; a store with selective semantic retrieval gives cross-session recall at bounded context cost.

---

## Implementation

```python
# Scenario: A personal assistant must remember a user's preferences ACROSS separate
# sessions. Short-term (checkpointer) state dies with the thread, so we add a long-term
# Store: write the memory (hot path here, for immediacy) and, on every turn, retrieve
# only the top-k relevant memories via semantic search and inject them into the prompt.
# APIs verified against LangGraph add-memory / stores docs (docs.langchain.com).
import uuid
from dataclasses import dataclass
from langchain.chat_models import init_chat_model
from langchain.embeddings import init_embeddings
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore
from langgraph.runtime import Runtime

model = init_chat_model("openai:gpt-4o-mini", temperature=0)

# Store with semantic search enabled so retrieval ranks memories by meaning, not recency.
store = InMemoryStore(index={"embed": init_embeddings("openai:text-embedding-3-small"),
                             "dims": 1536})
checkpointer = InMemorySaver()  # short-term (per-thread) memory, separate concern

@dataclass
class Context:
    user_id: str

def call_model(state: MessagesState, runtime: Runtime[Context]):
    ns = (runtime.context.user_id, "memories")           # per-user namespace = isolation
    # SELECT: pull only the top-3 relevant memories into the bounded context window.
    hits = runtime.store.search(ns, query=state["messages"][-1].content, limit=3)
    prefs = "\n".join(h.value["fact"] for h in hits)
    system = f"You are a travel assistant. Known user preferences:\n{prefs}"

    # WRITE (hot path): capture a durable preference immediately if the user states one.
    last = state["messages"][-1].content
    if "prefer" in last.lower() or "i am" in last.lower():
        runtime.store.put(ns, str(uuid.uuid4()), {"fact": last})

    response = model.invoke([{"role": "system", "content": system}, *state["messages"]])
    return {"messages": response}

builder = StateGraph(MessagesState, context_schema=Context)
builder.add_node(call_model)
builder.add_edge(START, "call_model")
graph = builder.compile(checkpointer=checkpointer, store=store)

# Same user_id in a brand-new thread still recalls the stored preferences.
graph.invoke({"messages": [{"role": "user", "content": "I prefer aisle seats"}]},
             {"configurable": {"thread_id": "t1"}}, context=Context(user_id="u-42"))
graph.invoke({"messages": [{"role": "user", "content": "Book me a flight to Rome"}]},
             {"configurable": {"thread_id": "t2"}}, context=Context(user_id="u-42"))
```

```python
# Anti-pattern: "preserve cross-session memory by NEVER trimming the thread" — keep one
# eternal thread_id per user and stuff all history into short-term state forever.
# This conflates short-term and long-term memory. The message list grows without bound,
# so every turn re-sends the entire history: context OVERFLOWS, cost/latency climb each
# turn, and the model gets "lost in the middle" among stale chatter.
def call_model_bad(state: MessagesState):
    # No trimming, no store — the whole growing history rides in the context window.
    return {"messages": model.invoke(state["messages"])}
# eternal_thread = {"configurable": {"thread_id": f"user-{user_id}-forever"}}  # never resets

# Correct approach: keep short-term memory BOUNDED (trim/summarize per chapter 02) AND put
# durable facts in a long-term Store, retrieving only top-k relevant ones per turn.
from langchain_core.messages.utils import trim_messages, count_tokens_approximately

def call_model_good(state: MessagesState, runtime: Runtime[Context]):
    # 1) Bound short-term memory so the window can't overflow.
    recent = trim_messages(state["messages"], strategy="last",
                           token_counter=count_tokens_approximately, max_tokens=2000,
                           start_on="human", end_on=("human", "tool"))
    # 2) Pull durable knowledge from long-term memory selectively (not all of it).
    ns = (runtime.context.user_id, "memories")
    hits = runtime.store.search(ns, query=state["messages"][-1].content, limit=3)
    prefs = "\n".join(h.value["fact"] for h in hits)
    system = f"You are a travel assistant. Known user preferences:\n{prefs}"
    return {"messages": model.invoke([{"role": "system", "content": system}, *recent])}
# What breaks without this: the never-trimmed thread eventually exceeds the context
# window (hard error) or degrades quality and cost long before that. Splitting the two
# memory systems keeps context bounded while durable facts still survive across sessions.
```

---

## Common Pitfalls & Misconceptions

- **"The model just remembers everything from before"** — beginners assume the LLM has persistent memory because a chat *feels* continuous. In reality the model is stateless per call; continuity within a thread comes from re-sending state (short-term, bounded by the window), and anything across threads only survives because you *wrote it to a store and retrieved it back*.
- **Using short-term memory for cross-session recall** — because thread state "remembers" earlier turns, people assume it persists forever. A thread is scoped to one `thread_id`; a new session/thread starts empty, so durable facts belong in a cross-thread `Store`, not in an ever-growing thread.
- **Never trimming the thread to "keep memory"** — trimming feels like "forgetting," so beginners disable it to be safe. Short-term memory is *supposed* to be bounded; unbounded threads overflow the context window and inflate cost/latency — durable knowledge goes to long-term memory, not an infinite message list.
- **Storing every message as a "memory"** — "more memory = better recall" seems intuitive, so people `put` every turn into the store. Retrieval then returns noise and the top-k fills with trivia; write *selective, summarized* memories (facts/preferences), so search surfaces signal.
- **Injecting all memories into the prompt** — once memories exist, dumping them all in looks thorough. That reintroduces overflow and "lost-in-the-middle" distraction; retrieve only the top-k relevant via `store.search(query=..., limit=k)` — memory is a *selection* problem, not a *storage* problem.
- **Confusing "semantic memory" with "semantic search"** — the shared word makes people equate them. Semantic memory is a memory *type* (facts); semantic search is a *retrieval technique* (embedding similarity) usable to retrieve any memory type.
- **Ignoring staleness and privacy of user memory** — a stored fact feels permanently true and harmless. Preferences change (stale "current project"), and personal facts are sensitive; timestamp/expire volatile memories, let newer statements override older, and isolate per-user namespaces.

---

## Key Definitions

| Term | Definition |
|---|---|
| Short-term (thread-scoped / working) memory | The state of a single conversation (messages + per-thread data), persisted by a checkpointer and bounded by the context window. |
| Long-term (cross-thread / persistent) memory | Application data (facts, examples, instructions) stored outside graph state in a Store, surviving across threads and sessions, retrieved on demand. |
| Checkpointer | LangGraph component (`InMemorySaver`, `PostgresSaver`, …) that saves graph state snapshots per `thread_id`; backs short-term memory, HITL, time travel, and fault tolerance. |
| `thread_id` | The config key (`configurable.thread_id`) that scopes and reloads a conversation's checkpointed state. |
| Store / `BaseStore` | The interface for long-term memory; `InMemoryStore` for dev, DB-backed implementations for prod; exposes `put`/`get`/`search`. |
| Namespace | A tuple of strings (e.g. `(user_id, "memories")`) that scopes a memory like a folder; the isolation boundary and search prefix. |
| Semantic memory | A memory *type*: facts about a user/world, managed as a profile (one updated doc) or a collection (many docs). |
| Episodic memory | A memory *type*: past experiences/examples, usually surfaced as few-shot examples selected by similarity. |
| Procedural memory | A memory *type*: learned rules/instructions, commonly an agent-updated system prompt via a reflection loop. |
| Hot-path writing | Committing a memory synchronously during the turn (immediate availability, added latency). |
| Background writing | Committing memories asynchronously after the turn in a separate process (no turn latency, some staleness). |
| Memory selection/retrieval | Choosing which stored memories enter the bounded context window per turn, typically top-k via semantic search. |

---

## Summary / Quick Recall

- Two systems, not one: **short-term** = thread-scoped state behind a **checkpointer** (`thread_id`), bounded by the context window; **long-term** = cross-thread data in a **Store** (`namespace`), retrieved on demand.
- The model does **not** remember automatically — continuity comes from re-sent state; cross-session recall exists only because you **wrote** to a store and **retrieved** it back.
- Long-term memory has three **types**: semantic (facts), episodic (experiences/few-shot examples), procedural (instructions/updated prompt).
- Write memories **hot path** (immediate, adds latency) or **background** (no turn latency, some staleness) — pick per the latency requirement.
- Storage is easy; **selection is hard** — retrieve top-k relevant memories via `store.search(query=..., limit=k)`, don't dump them all (that re-creates overflow) — this is context engineering applied to memory.
- Keep short-term **bounded** (trim/summarize, ch 02) and put durable facts in long-term; never keep one eternal, never-trimmed thread.
- Mind **privacy** (namespace per user) and **staleness** (expire/override volatile facts) for persistent user memory.

---

## Self-Check Questions

1. Which persistence primitive backs *short-term* (thread-scoped) memory in LangGraph, and which backs *long-term* (cross-thread) memory?

   <details><summary>Answer</summary>

   Short-term memory is backed by a **checkpointer** (e.g. `InMemorySaver`, `PostgresSaver`), which saves the graph *state* snapshot per `thread_id`; long-term memory is backed by a **Store** (`BaseStore`/`InMemoryStore`), which holds JSON docs under a `namespace` and is accessible across threads. The tempting wrong answer is "the checkpointer does both" — but the checkpointer is scoped to a single thread and only persists graph state, so it cannot serve data to a *different* thread/session; that's exactly why the Store is a separate primitive.

   </details>

2. You built an assistant with only a checkpointer. Users complain that when they start a *new* chat, it has forgotten their preferences from last week. What is the fix and why?

   <details><summary>Answer</summary>

   Add **long-term memory via a Store**: write durable preferences to `store.put((user_id, "memories"), ...)` and retrieve them with `store.search(...)` at the start of each turn, injecting the top-k into the system prompt. A new chat is a new `thread_id`, and checkpointer state is thread-scoped, so it necessarily starts empty. The tempting wrong answer is "reuse one permanent `thread_id` per user so the thread never resets" — that stuffs all history into short-term state, overflowing the context window and inflating cost; cross-session recall belongs in a store, not an eternal thread.

   </details>

3. **Which TWO** of the following are correct about long-term agent memory?
   - A. Semantic, episodic, and procedural are three *types* of long-term memory (facts, experiences, instructions).
   - B. Every message in a conversation should be written to the store as its own memory to maximize recall.
   - C. Retrieving memories typically uses semantic search (`store.search(query=..., limit=k)`) to select only the top-k relevant into context.
   - D. "Semantic memory" and "semantic search" are the same thing.
   - E. Memories stored in the Store are automatically present in the model's context on every turn.

   <details><summary>Answer</summary>

   **A and C.** A is the standard taxonomy (CoALA mapping): facts = semantic, experiences/examples = episodic, rules/instructions = procedural. C is correct because memory is a *selection* problem — you retrieve a small top-k by similarity so the context stays bounded. B is the most tempting distractor and is wrong: writing every message produces noisy retrievals; you write selective, summarized memories. D is a common confusion — semantic memory is a *type* (facts), semantic search is a *technique* (embedding similarity). E is wrong: Store data lives *outside* graph state and only enters context when you explicitly retrieve and inject it.

   </details>

4. Your assistant must feel snappy, but it also needs to learn user preferences over time. Compare hot-path versus background memory writing for this case and recommend one.

   <details><summary>Answer</summary>

   **Hot-path** writing commits the memory during the turn (immediately available, can be shown to the user) but the extraction reasoning **adds latency** to the response. **Background** writing runs memory extraction in a separate process after the turn, so it **adds zero latency** to the reply at the cost of some **staleness** (new memories aren't visible until the job runs). Since the requirement is snappiness and preferences accrue *over time* (they needn't be usable the same instant), **background writing is the better default**, optionally with a lightweight hot-path save only for facts the user explicitly asks to remember. Choosing pure hot-path would sacrifice the very latency the requirement prioritizes.

   </details>

5. A teammate says: "Storage is basically free, so let's just retrieve *all* of a user's memories into the prompt every turn — more context is better." Evaluate this under real constraints.

   <details><summary>Answer</summary>

   It's wrong because the **context window is the scarce resource, not storage**. Injecting all memories reintroduces the exact problems compaction was built to solve: you can overflow the window (a hard error), you pay for every token each turn, and the model gets **"lost in the middle,"** degrading quality as irrelevant memories dilute the signal. The correct approach is **selective retrieval** — `store.search(query=<current message>, limit=k)` with a small k — so only the most relevant memories enter the bounded context. "More context is better" fails precisely because relevance-per-token, not raw volume, drives answer quality; memory is a selection/context-engineering problem.

   </details>

---

## Further Reading

- [Memory overview (concepts) — short-term vs long-term, semantic/episodic/procedural, hot-path vs background](https://docs.langchain.com/oss/python/concepts/memory) — *verified 2026-07-29* — LangGraph's conceptual guide to memory recall scope, the CoALA memory-type taxonomy, and memory-writing strategies.
- [Persistence — checkpointers vs stores](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29* — Explains that checkpointers give short-term thread-scoped memory and stores give long-term cross-thread memory, with the comparison table.
- [Checkpointers (short-term / thread-scoped state)](https://docs.langchain.com/oss/python/langgraph/checkpointers) — *verified 2026-07-29* — Threads, `thread_id`, checkpoints/super-steps, and the checkpointer backends behind short-term memory.
- [Stores (long-term / cross-thread memory)](https://docs.langchain.com/oss/python/langgraph/stores) — *verified 2026-07-29* — `BaseStore`/`InMemoryStore`, namespaces, `put`/`get`/`search`, and configuring semantic search over memories.
- [Add and manage memory (how-to)](https://docs.langchain.com/oss/python/langgraph/add-memory) — *verified 2026-07-29* — Working code for adding short-term (checkpointer) and long-term (store) memory, semantic search, and managing short-term memory (trim/delete/summarize).
