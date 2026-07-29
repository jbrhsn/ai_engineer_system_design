# LangGraph State Graphs and Orchestration Primitives

**Section:** 03 Agentic & Multi-Agent Systems → Multi-Agent Orchestration with LangGraph | **Est. time:** 3 hrs | **Interview relevance:** High — this is the substrate every multi-agent design in this section stands on; supervisor topologies, hierarchical teams, and human-in-the-loop are all just patterns of nodes, edges, and shared state, so if you can't defend the primitives you can't defend the architecture.

---

## TL;DR

A single tool-calling agent (previous chapter) is a model in a loop; **LangGraph** is the framework for wiring *many* such steps into an explicit, inspectable control-flow graph. You model your system as a **StateGraph**: **nodes** are plain functions that do the work, **edges** decide what runs next, and a typed **State** object is the shared channel every node reads from and writes to. State updates don't blindly overwrite — each key has a **reducer** (e.g. `add_messages`) that says *how* a node's partial update merges into the accumulated value, which is what makes concurrent/parallel writes safe. Execution follows a Pregel-style **super-step** model (parallel branches run in the same super-step; sequential nodes run in separate ones), bounded by a **`recursion_limit`** so a cyclic graph can't run forever. **The one thing to remember: nodes do the work, edges say what's next, and the reducer-typed state is the *only* thing they share — master those three and every supervisor/hierarchical/HITL topology in this section is just a different arrangement of them.**

---

## ELI5 — Explain It Like I'm 5

Imagine a factory floor with several work stations, and a single clipboard that travels the floor holding everything known so far. Each station (a **node**) picks up the clipboard, does one job, writes its result on it, and puts it down; painted arrows on the floor (**edges**) tell the clipboard which station to visit next, and some arrows have a little sign — "if the part is red go left, otherwise go right" — that's a *conditional* edge. The important twist is *how* things get written: for some clipboard fields the new station crosses out the old value and writes its own, but for the running "log of everything done" field there's a rule that says *append, don't overwrite* — that rule is the **reducer**, and it's the only reason two stations working at the same time don't scribble over each other. A common misconception is that this is just a straight assembly line from station 1 to station N; it isn't — arrows can loop back, split into parallel branches, or skip ahead, and the whole point of drawing it as a map (a graph) instead of a straight line is that loops, branches, and merges become things you can *see and control* rather than tangle inside one function.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the StateGraph model — nodes as functions, edges as control flow, and a typed State as the shared channel — and articulate why it generalizes the single-agent loop.
- [ ] Define a state schema with reducers (`add_messages`, `Annotated` custom reducers) and predict how concurrent node updates to the same key merge.
- [ ] Wire normal edges, conditional edges, and the `Command` primitive, and choose correctly between routing-only and update-plus-routing.
- [ ] Trace execution through the Pregel-style super-step model, including how parallel branches run in one super-step and where `START`/`END`/`compile()` fit.
- [ ] Bound a cyclic graph with `recursion_limit` and diagnose the `InvalidUpdateError` that arises when parallel writers share an unreduced key.

---

## Visual Overview

### StateGraph: Nodes, Edges, and the Shared State Channel

```
                    ┌─────────────────────────────────────┐
                    │  STATE (typed shared channel)        │
                    │   messages: [...]  (reducer:         │
                    │                     add_messages)    │
                    │   next: "..."      (default:         │
                    │                     overwrite)       │
                    └─────────────────────────────────────┘
                          ▲ read/write        ▲ read/write
                          │                   │
   START ──► ┌──────────┐ │  conditional  ┌──────────┐ │  ┌──────────┐
             │  node_a  │─┼──── edge ─────►│  node_b  │─┼─►│  node_c  │──► END
             │(function)│ │   (router)  ┌─►│(function)│    │(function)│
             └──────────┘ │             │  └──────────┘    └──────────┘
                          └── else ─────┘
   nodes DO the work  •  edges say WHAT'S NEXT  •  state is the ONLY shared thing
```

### Super-Step / Pregel Execution Model (parallel branches)

```
 super-step 1        super-step 2                 super-step 3
 ───────────         ─────────────────────        ────────────
 ┌───────┐           ┌───────┐   ┌───────┐         ┌────────┐
 │ START │──fan-out─►│ branch│   │ branch│──fan-in─►│ merge  │──► END
 │ node  │        ┌─►│   A   │   │   B   │◄─┐       │  node  │
 └───────┘        │  └───────┘   └───────┘  │       └────────┘
                  │      │           │       │
                  └──────┴───────────┴───────┘
  A and B run IN PARALLEL in the SAME super-step (both got a message this step).
  merge runs in the NEXT super-step, AFTER both A and B have written state.
  A run = a sequence of super-steps; recursion_limit caps how many can occur.
```

### How a Reducer Merges Concurrent Updates to One Key

```
   accumulated state:  messages = [Human("hi")]
                              │
         ┌────────────────────┴────────────────────┐
         ▼ (parallel node A)                        ▼ (parallel node B)
   returns {"messages":[AI("from A")]}    returns {"messages":[AI("from B")]}
         │                                          │
         └───────────────► REDUCER ◄────────────────┘
                    add_messages(left, right)
                              │
                              ▼
        messages = [Human("hi"), AI("from A"), AI("from B")]
   WITHOUT a reducer on this key: two writers in one super-step ──► InvalidUpdateError
```

### Routing Mechanism Decision

```
Does this node need to change state AND pick the next node?
├── Only pick the next node (no state change)
│     ├── always same next node ──► add_edge (normal edge)
│     └── depends on state       ──► add_conditional_edges (router fn)
└── Both update state AND route  ──► return Command(update=..., goto=...)
      (do NOT also add a normal edge from the same node — both would fire)
```

---

## Key Concepts

### The StateGraph Model (nodes / edges / state)

**What it is.** `StateGraph` is LangGraph's main graph class: you parameterize it with a user-defined `State` type, register **nodes** (functions that do work) and **edges** (control flow between nodes), then `compile()` it into a runnable. It is the generalization of the single-agent loop — instead of one hard-coded "call model → run tools → loop" cycle, you draw an arbitrary directed graph that can branch, merge, and cycle.

**How it works mechanistically.** LangGraph models the workflow with message passing (inspired by Google's Pregel). A node receives the current `state`, does a computation or side-effect, and returns a *partial* update (just the keys it changed, not the whole state). When a node finishes it "sends messages" along its outgoing edges to the next node(s); a node becomes active when it receives input on an incoming edge, runs, and emits its update. The mantra from the docs is literal: *nodes do the work, edges tell what to do next* — and both are "nothing more than functions," which can wrap an LLM call or just plain Python.

**Where it appears in real systems.** `from langgraph.graph import StateGraph`; `builder = StateGraph(MyState)`. The state schema is a `TypedDict`, a `dataclass` (for default values), or a Pydantic `BaseModel` (for recursive validation, at a perf cost). This is the exact object you saw in the previous chapter's `StateGraph(MessagesState)` agent loop — the supervisor and hierarchical topologies in chapter 02 are the *same* `StateGraph`, just with more nodes and smarter routing.

### State Schema and Reducers

**What it is.** The **state schema** declares the channels (keys) every node can read and write; a **reducer** is a per-key function that decides how a node's returned update is *applied* to the current value of that key. If no reducer is declared for a key, the default is overwrite (the new value replaces the old).

**How it works mechanistically.** A reducer is a binary function `reducer(left, right)` where `left` is the value currently in state and `right` is the node's returned update; LangGraph calls it for each updated key and stores the return value: `new_value = reducer(current_state[key], node_update[key])`. The default reducer ignores `left` and keeps `right` (overwrite). A custom reducer *combines* them — e.g. `operator.add` on a list appends. You attach one with `Annotated`: `bar: Annotated[list[str], add]`. This is also what makes parallel writes to one key well-defined: when two nodes in the same super-step both update `bar`, the reducer folds both updates in; without a reducer, two concurrent writers to the same key raise `InvalidUpdateError` because "just overwrite" is ambiguous when there are two rights.

**Where it appears in real systems.** `from typing import Annotated; class State(TypedDict): tags: Annotated[list[str], add]`. Any state field that accumulates across nodes (a message log, a list of retrieved docs, per-agent scratch findings) needs an accumulating reducer; scalar fields that only ever hold "the latest value" (a routing decision, a step counter) use the default overwrite.

### `add_messages` and `MessagesState`

**What it is.** `add_messages` is LangGraph's prebuilt reducer for a list-of-messages channel; `MessagesState` is a prebuilt state class with a single `messages` key already annotated with it.

**How it works mechanistically.** A naive list reducer (`operator.add`) would blindly append, which breaks two things: manual edits (human-in-the-loop overwriting a prior message) and de-duplication. `add_messages` is smarter — it tracks message **IDs**: a brand-new message is appended, but an update carrying an existing ID *overwrites* that message in place. It also deserializes raw dicts (`{"role": "user", "content": "..."}`) into proper LangChain `Message` objects on the way in, which is why you can invoke a graph with plain dicts and still read `state["messages"][-1].content`.

**Where it appears in real systems.** `from langgraph.graph.message import add_messages` for the raw reducer, or `from langgraph.graph import MessagesState` and subclass it to add fields: `class State(MessagesState): documents: list[str]`. This is the default backbone of almost every conversational/agentic graph, and it's what the handoff patterns in chapter 02 rely on so that transferring control between agents preserves a valid message history.

### Nodes and Edges (normal + conditional)

**What it is.** A **node** is a function registered with `add_node`; a **normal edge** (`add_edge`) is an unconditional "after A, always go to B"; a **conditional edge** (`add_conditional_edges`) calls a routing function that inspects state and returns the name(s) of the next node(s).

**How it works mechanistically.** `add_node("name", fn)` registers a function taking `(state)` (optionally `config`/`runtime`). `add_edge("a", "b")` hard-wires the transition. `add_conditional_edges("a", router)` runs `router(state)` after node `a`; the router's return value *is* the next node name, or you pass a mapping dict `{"yes": "b", "no": "c"}` to translate the return value to a destination. Critically, a node can have multiple outgoing edges — if so, **all** destinations run in parallel in the next super-step (this is how you fan out). The docs warn: for any one node, pick *one* routing mechanism — don't mix a normal edge and dynamic routing from the same node, or both paths fire.

**Where it appears in real systems.** `builder.add_conditional_edges("call_model", route, {"tools": "tools", END: END})` — exactly the loop-vs-stop router from the single-agent chapter. In a supervisor topology, the supervisor node's conditional edge is what routes to `worker_a`, `worker_b`, or `END`.

### The `Command` Primitive (update + route in one step)

**What it is.** `Command` is a versatile control primitive a node can *return* to **both** update state **and** specify the next node in a single move — a fusion of "return a state update" and "conditional edge."

**How it works mechanistically.** A node returns `Command(update={...}, goto="next_node")`; `update` is applied through the reducers exactly like a normal returned dict, and `goto` navigates like a conditional edge. Because a static router can't reach into and change state, `Command` is the tool when a routing decision and a state write are the same logical act (e.g. "record the answer *and* move to the specialist"). Two rules matter: you must annotate the return type with the reachable nodes — `def n(state) -> Command[Literal["other"]]` — for graph rendering; and `Command` adds a *dynamic* edge on top of static ones, so if you also declared `add_edge("n","b")`, both `b` and `goto` fire. `Command` is also used with `graph=Command.PARENT` to hand off from a subgraph node to a parent-graph node, and with `resume=` as input to `invoke` after an interrupt.

**Where it appears in real systems.** `from langgraph.types import Command`. The multi-agent **handoff** pattern (chapter 02) is built almost entirely on tools/nodes returning `Command(goto="sales_agent", update={...}, graph=Command.PARENT)`; the HITL/durable-execution chapter (03) uses `Command(resume=value)` to continue a paused graph after an `interrupt()`.

### `START`, `END`, and `compile()`

**What it is.** `START` and `END` are virtual nodes: `START` represents where user input enters, `END` is the terminal marker. `compile()` turns the mutable builder into an immutable, runnable graph.

**How it works mechanistically.** `add_edge(START, "first_node")` declares the entry point (equivalent to the older `set_entry_point`); `add_edge("last_node", END)` marks where a path terminates. `compile()` runs structural checks (no orphaned nodes, etc.) and is where you attach runtime concerns like a **checkpointer** (for persistence/HITL — chapter 03) and breakpoints. You *must* compile before invoking; the compiled object exposes `.invoke()`, `.stream()`, and async variants.

**Where it appears in real systems.** `graph = builder.compile()` or `builder.compile(checkpointer=InMemorySaver())`. The presence or absence of a checkpointer at `compile()` time is the dividing line between a stateless single-shot graph and the durable, resumable graphs the HITL chapter needs.

### Super-Steps and the Pregel Execution Model

**What it is.** A **super-step** is one discrete iteration over the graph's active nodes; LangGraph executes in a sequence of super-steps (the Pregel/BSP model). Nodes that run in parallel share one super-step; sequential nodes fall in separate super-steps.

**How it works mechanistically.** At start, all nodes are inactive; a node becomes active when a message arrives on an incoming edge. Active nodes in the current super-step run (in parallel, if there are several), each emitting a state update; at the end of the super-step, updates are applied through the reducers, and nodes with no new incoming messages vote to halt. Execution ends when all nodes are inactive and no messages are in transit. This is why parallel branches' writes to a shared key must go through a reducer — they're applied together at the super-step boundary, so a bare-overwrite key with two writers is a conflict.

**Where it appears in real systems.** You don't call super-steps directly, but they're the unit `recursion_limit` counts, and they're what `stream_mode="updates"` emits one of per step. Understanding them is how you reason about "do these two workers run at once or one after the other?" in a multi-agent graph.

### `recursion_limit` (bounding execution)

**What it is.** `recursion_limit` caps the maximum number of **super-steps** a single graph execution may run; exceeding it raises `GraphRecursionError`. As of v1.0.6 the default is **1000** super-steps.

**How it works mechanistically.** Because LangGraph graphs can contain cycles (an agent loop, a reflect-and-retry loop, a supervisor bouncing between workers), there must be a hard backstop or a graph that never routes to `END` would run forever. `recursion_limit` is that backstop. It is a **standalone top-level `config` key** — `config={"recursion_limit": 25}` — *not* nested inside `configurable`. For graceful handling before the crash, the `RemainingSteps` managed value lets a router proactively divert to a fallback node when steps run low.

**Where it appears in real systems.** `graph.invoke(inputs, config={"recursion_limit": 25})`. In a multi-agent supervisor loop you set this to the smallest value that covers the worst *legitimate* number of hops, so a mis-routing loop between two agents fails fast instead of billing 1000 model calls.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `recursion_limit` (config key) | Max super-steps before `GraphRecursionError` | Set to the smallest value covering your worst *legitimate* path (often 10–30 for a bounded supervisor loop), not the 1000 default; pass it top-level in `config`, never inside `configurable`. |
| Reducer choice per state key | How a node's update merges with the current value | Use `add_messages` for message channels; an accumulating reducer (`operator.add`) for any list many nodes append to or that parallel branches write; leave default overwrite for "latest value only" scalars (routing decision, step count). |
| State schema type (`TypedDict` / `dataclass` / Pydantic) | Validation and defaults for the shared channel | `TypedDict` by default (fastest); `dataclass` when you need default field values; Pydantic `BaseModel` only when you need recursive runtime validation and accept the perf cost. |
| Routing mechanism (`add_edge` vs `add_conditional_edges` vs `Command`) | How the next node is chosen | Normal edge for a fixed transition; conditional edges to route on state *without* changing it; `Command` when the same node must update state *and* route — never combine a static edge with `Command`/dynamic routing on one node. |
| `checkpointer` at `compile()` | Whether state is persisted across steps/turns | Omit for stateless single-shot runs; supply (`InMemorySaver` for dev, `PostgresSaver` for prod) whenever you need HITL interrupts, multi-turn threads, or fault-tolerant resume (chapter 03). |
| `cache_policy` on a node | Whether a node's output is cached by input | Set `CachePolicy(ttl=...)` on expensive, deterministic nodes (e.g. a costly retrieval) whose output is stable for identical inputs; leave off for nodes with side-effects or nondeterministic output. |

### Worked Example: Requirement → Decision

**Given:** You're building a research assistant. On each query it must run *two independent* lookups in parallel — a web search and an internal vector-store search — then a synthesis step must combine both results into one answer. Each searcher appends its findings to a shared `findings` list. Latency matters (running the two searches sequentially would double the wait), and the graph must be guaranteed to terminate.

- **Step 1 — Identify the goal:** Fan out to two searchers concurrently, fan back in to a synthesizer that sees *both* sets of findings, with safe concurrent writes and bounded execution.
- **Step 2 — Define inputs:** A `State` with `query: str` (overwrite), `findings: Annotated[list, add]` (accumulating reducer so both searchers can write it), and `messages` on `add_messages`.
- **Step 3 — Define outputs:** The synthesizer reads the merged `findings` and writes a final `AIMessage` to `messages`.
- **Step 4 — Apply constraints:** The two searches are independent (safe to parallelize) but both write the *same* key (`findings`) in the *same* super-step, so that key MUST have a reducer or it will raise `InvalidUpdateError`; latency budget rules out sequential edges; termination must be guaranteed even though the graph could loop on retries.
- **Step 5 — Select the approach:** From `START`, add edges to **both** `web_search` and `vector_search` (multiple outgoing edges from START → both run in one super-step, i.e. in parallel). Give `findings` an `operator.add` reducer so the two concurrent writes merge instead of colliding. Add edges from both searchers to a single `synthesize` node (fan-in: `synthesize` runs in the *next* super-step, after both writers commit). Compile and invoke with `config={"recursion_limit": 10}`. *Rationale vs alternatives:* sequential edges (`START→web→vector→synthesize`) would be simpler but blow the latency budget; using `Command` here is unnecessary because the searchers route the same way regardless of state (a static fan-out edge is clearer); the reducer — not a lock or a merge node — is the idiomatic LangGraph answer to concurrent writes.

---

## Implementation

```python
# Scenario: Two independent searches must run in PARALLEL and both append to the
# same `findings` list, then a synthesizer combines them. This is the canonical
# fan-out/fan-in multi-agent shape (supervisor topologies build on it). The key
# design point is the reducer that makes concurrent writes to `findings` safe.
# APIs verified against LangGraph Graph API docs (docs.langchain.com).
from operator import add
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class ResearchState(TypedDict):
    query: str                                   # default reducer: overwrite
    findings: Annotated[list[str], add]          # accumulating reducer: append

def web_search(state: ResearchState):
    # returns a PARTIAL update — only the key it changed
    return {"findings": [f"web result for {state['query']}"]}

def vector_search(state: ResearchState):
    return {"findings": [f"internal doc for {state['query']}"]}

def synthesize(state: ResearchState):
    # runs in the NEXT super-step, after BOTH searchers have committed findings
    return {"findings": [f"SUMMARY of {len(state['findings'])} findings"]}

builder = StateGraph(ResearchState)
builder.add_node("web_search", web_search)
builder.add_node("vector_search", vector_search)
builder.add_node("synthesize", synthesize)

# START has TWO outgoing edges -> both searchers run in the SAME super-step (parallel)
builder.add_edge(START, "web_search")
builder.add_edge(START, "vector_search")
# fan-in: synthesize waits until both incoming branches have run
builder.add_edge("web_search", "synthesize")
builder.add_edge("vector_search", "synthesize")
builder.add_edge("synthesize", END)

graph = builder.compile()
result = graph.invoke({"query": "vector DBs", "findings": []},
                      config={"recursion_limit": 10})  # top-level, not in configurable
```

```python
# Anti-pattern: two nodes run in parallel and BOTH write the same state key, but the
# key has NO reducer. At the super-step boundary LangGraph gets two conflicting
# "overwrite" updates for one key and raises InvalidUpdateError (it cannot decide
# which write wins). Beginners hit this the moment they fan out to parallel workers.
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class BrokenState(TypedDict):
    findings: list[str]        # BROKEN: no reducer, yet two parallel nodes write it

def a(state): return {"findings": ["from A"]}
def b(state): return {"findings": ["from B"]}

b1 = StateGraph(BrokenState)
b1.add_node("a", a); b1.add_node("b", b)
b1.add_edge(START, "a"); b1.add_edge(START, "b")   # a and b run in ONE super-step
b1.add_edge("a", END); b1.add_edge("b", END)
# b1.compile().invoke({"findings": []})  ->  InvalidUpdateError: concurrent writes

# Correct approach: give the shared key a reducer so the two updates MERGE instead
# of colliding. The Annotated[..., add] reducer folds both writes at the super-step
# boundary, yielding ["from A", "from B"] (order not guaranteed across the branches).
from operator import add
from typing import Annotated

class FixedState(TypedDict):
    findings: Annotated[list[str], add]           # reducer resolves concurrent writes

def a2(state): return {"findings": ["from A"]}
def b2(state): return {"findings": ["from B"]}

b2g = StateGraph(FixedState)
b2g.add_node("a", a2); b2g.add_node("b", b2)
b2g.add_edge(START, "a"); b2g.add_edge(START, "b")
b2g.add_edge("a", END); b2g.add_edge("b", END)
graph2 = b2g.compile()   # invoke now merges: {"findings": ["from A", "from B"]}
# What breaks without the reducer: any fan-out where branches touch a shared key
# throws InvalidUpdateError at runtime. The reducer is LangGraph's built-in answer
# to "how do concurrent updates merge" — not locks, not a manual merge node.
```

```python
# Scenario: A router node must BOTH record a classification into state AND jump to
# the matching specialist node. That's a state-update + route in one logical act, so
# a conditional edge (route-only) is not enough — return a Command. This is the exact
# shape multi-agent handoffs use. Verified against Graph API `Command` docs.
from typing import Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command

class TriageState(TypedDict):
    request: str
    category: str          # overwrite: only the latest classification matters

def triage(state: TriageState) -> Command[Literal["billing", "tech"]]:
    cat = "billing" if "invoice" in state["request"] else "tech"
    # update state AND route in ONE return — a conditional edge could not set `category`
    return Command(update={"category": cat}, goto=cat)

def billing(state): return {"request": state["request"] + " [handled by billing]"}
def tech(state):    return {"request": state["request"] + " [handled by tech]"}

g = StateGraph(TriageState)
g.add_node("triage", triage)
g.add_node("billing", billing)
g.add_node("tech", tech)
g.add_edge(START, "triage")
# NOTE: no static edge out of `triage` — Command supplies its dynamic edge. Adding a
# normal edge here too would make BOTH the static target and `goto` fire.
g.add_edge("billing", END)
g.add_edge("tech", END)
graph3 = g.compile()
```

---

## Common Pitfalls & Misconceptions

- **Thinking a StateGraph is just a linear chain** — the word "graph" gets skimmed and people mentally model step 1 → step 2 → step N. LangGraph's value *is* the non-linear part: edges can loop back (agent loops), fan out to parallel branches, and merge; drawing it as a graph is what makes those control-flow shapes explicit and inspectable instead of buried in one function's `if`/`while`.
- **Two parallel nodes writing one key with no reducer** — beginners fan out to parallel workers and have each write `results`, expecting the framework to "just merge." Without a reducer on that key LangGraph raises `InvalidUpdateError` at the super-step boundary because two "overwrite" updates for one key are ambiguous; the fix is an accumulating reducer (`Annotated[list, add]` / `add_messages`), which is the *designed* mechanism for concurrent merges.
- **Using `operator.add` for a message channel** — it looks equivalent to `add_messages` and appends fine at first. But `operator.add` blindly concatenates, so a human-in-the-loop edit that re-sends an existing message ID *duplicates* it instead of overwriting; `add_messages` tracks IDs (append new, overwrite existing) and deserializes dicts — always use it for messages.
- **Mixing a normal edge and `Command`/dynamic routing on one node** — it feels safe to add a fallback `add_edge` "just in case." But `Command`'s `goto` is a *dynamic* edge layered on top of static ones, so both the static target and the `goto` target fire, running a node you didn't intend. Pick exactly one routing mechanism per node.
- **Leaving `recursion_limit` at the default 1000** — the happy path terminates in testing so the cap seems irrelevant. A supervisor that mis-routes into a two-agent ping-pong will run up to 1000 super-steps of model calls before crashing; set the limit to the smallest value covering your worst legitimate hop count, and pass it as a top-level `config` key (not inside `configurable`).
- **Nesting `recursion_limit` under `configurable`** — every other user config goes in `configurable`, so people put this there too and it's silently ignored. `recursion_limit` is a standalone top-level key: `config={"recursion_limit": 25}`.

---

## Key Definitions

| Term | Definition |
|---|---|
| StateGraph | LangGraph's main graph class, parameterized by a user-defined State schema; holds nodes and edges and is turned into a runnable by `compile()`. |
| Node | A function (sync or async) registered with `add_node` that reads state, does work, and returns a partial state update. |
| Edge | Control flow between nodes: a normal edge (`add_edge`, fixed) or a conditional edge (`add_conditional_edges`, routes on a function's return). |
| State (channel) | The typed shared data structure every node reads and writes; each key is a channel with its own reducer. |
| Reducer | A per-key binary function `reducer(left, right)` deciding how a node's update (`right`) merges into the current value (`left`); default is overwrite. |
| `add_messages` | Prebuilt reducer for message lists that appends new messages, overwrites by matching ID, and deserializes dicts into Message objects. |
| `MessagesState` | Prebuilt state class with a single `messages` key annotated with `add_messages`; commonly subclassed to add fields. |
| `Command` | A primitive a node returns to update state (`update=`) and route (`goto=`) in one step; also carries `graph=Command.PARENT` and `resume=`. |
| `START` / `END` | Virtual nodes marking where input enters and where a path terminates. |
| `compile()` | Converts the mutable builder into a runnable graph; runs structural checks and attaches runtime concerns like a checkpointer. |
| Super-step | One discrete iteration over active nodes (Pregel/BSP); parallel nodes share a super-step, sequential nodes occupy separate ones. |
| `recursion_limit` | Top-level config capping the number of super-steps per run; exceeding it raises `GraphRecursionError` (default 1000). |
| `InvalidUpdateError` | Raised when multiple concurrent writers update the same state key that has no reducer to merge them. |

---

## Summary / Quick Recall

- A **StateGraph** = **nodes** (functions that do work) + **edges** (what runs next) + a typed **State** (the only shared thing); it generalizes the single-agent loop to arbitrary branch/merge/cycle graphs.
- Each state key has a **reducer**: default is overwrite, `add_messages` for message channels, an accumulating reducer for lists — the reducer is what makes concurrent/parallel writes to one key well-defined.
- Two parallel nodes writing an **unreduced** key raise **`InvalidUpdateError`**; the fix is an `Annotated[..., reducer]`, not a lock.
- Routing: **normal edge** (fixed), **conditional edge** (route on state, no state change), **`Command`** (update state *and* route in one node) — never mix static + dynamic routing on the same node.
- Execution is a sequence of **super-steps** (Pregel): parallel branches run in one super-step, sequential nodes in separate ones; you must `compile()` before invoking.
- **`recursion_limit`** (top-level config, default 1000) bounds super-steps so cyclic graphs terminate; size it to your worst legitimate hop count.
- These primitives are the substrate: supervisor/hierarchical topologies (ch. 02) are more nodes + routing; HITL/durable execution (ch. 03) is `compile(checkpointer=...)` + `Command(resume=...)`.

---

## Self-Check Questions

1. In LangGraph, what is a reducer, and what does the *default* reducer do to a state key that has no reducer declared?

   <details><summary>Answer</summary>

   A reducer is a per-key binary function `reducer(left, right)` — `left` is the value currently in state, `right` is the update a node returned — whose result becomes the new value for that key. The **default** reducer ignores `left` and keeps `right`, i.e. it **overwrites** the key with the latest node's value. The tempting wrong answer is "the default appends/merges updates" — that is only true if you explicitly attach an accumulating reducer (`operator.add`, `add_messages`); with no reducer declared, updates overwrite, which is exactly why two concurrent writers to such a key conflict.

   </details>

2. You fan out from `START` to two nodes that run in parallel, and both return `{"results": [...]}`. Your first run raises `InvalidUpdateError`. What is the cause and the minimal fix?

   <details><summary>Answer</summary>

   Both parallel nodes write the **same key (`results`) in the same super-step**, but the key has **no reducer**, so LangGraph receives two "overwrite" updates for one key and can't decide which wins — hence `InvalidUpdateError`. The minimal fix is to annotate the key with an accumulating reducer so the two updates merge: `results: Annotated[list, operator.add]` (or `add_messages` for a message channel). The tempting wrong answer is "run the two nodes sequentially instead" — that avoids the error but discards the parallelism you wanted; the reducer keeps the concurrency *and* resolves the merge, which is the designed mechanism.

   </details>

3. **Which TWO** of the following require returning a `Command` from a node rather than using a plain conditional edge?
   - A. A router node that only inspects `state["category"]` and sends control to `billing` or `tech` without changing state.
   - B. A node that must record a classification into `state["category"]` *and* route to the matching specialist in one step.
   - C. A handoff tool/node that updates `messages` and navigates to a node in the *parent* graph.
   - D. A node that always transitions to the same next node regardless of state.
   - E. Splitting from `START` into two parallel branches.

   <details><summary>Answer</summary>

   **B and C.** B needs `Command` because a conditional edge can *route* but cannot *write* state — updating `category` and choosing the destination is one logical act, which is exactly `Command(update=..., goto=...)`. C needs `Command` because crossing from a subgraph to a parent-graph node requires `graph=Command.PARENT` (with a state update), which only `Command` provides. A is the most tempting distractor but is wrong — routing on state *without* changing it is precisely what `add_conditional_edges` is for, so a `Command` is unnecessary. D is a plain `add_edge`. E is a static fan-out via two `add_edge(START, ...)` calls, no `Command` needed.

   </details>

4. Two nodes, X and Y, both fed from `START`, plus a node Z fed by edges from both X and Y. In terms of super-steps, when do X, Y, and Z run, and why does that ordering make Z's view of state safe?

   <details><summary>Answer</summary>

   X and Y both receive input from `START`, so they run **in parallel in the same super-step** (super-step 1). Z has incoming edges from both X and Y, so it runs in the **next super-step (2)**, only after *both* X and Y have completed and their updates have been applied through the reducers at the super-step boundary. That fan-in ordering is what makes Z's read safe: because updates are committed at the boundary before Z activates, Z sees the merged result of both branches rather than a half-updated state. The tempting wrong answer is "Z might run as soon as X finishes" — it won't; a node activates only when all its incoming branches for that step have delivered, so Z waits for the whole super-step to settle.

   </details>

5. A teammate ships a supervisor graph with the default `recursion_limit` (1000) and reports occasional runs that burn hundreds of model calls before failing. Traces show the supervisor ping-ponging between two worker agents that never route to `END`. Evaluate the fix and the trade-off.

   <details><summary>Answer</summary>

   The root cause is a **routing bug** (the supervisor never converges to `END`) combined with an effectively-unbounded backstop: at the 1000-step default, the cycle runs up to 1000 super-steps of model calls before `GraphRecursionError`. The immediate mitigation is to set `recursion_limit` to the smallest value covering the worst *legitimate* number of hops (say 15–25) and pass it as a **top-level** `config` key (`config={"recursion_limit": 20}`), not nested under `configurable` where it's ignored — so a mis-routing loop fails fast and cheap. The deeper fix is correcting the router (or using `RemainingSteps` to divert to a graceful fallback node before the limit). The trade-off: too low a limit will falsely abort legitimately long runs; too high re-opens the runaway-cost window — so size it to the real worst-case path, not a round number. The tempting wrong answer is "swap to a bigger model" — that changes nothing, because the loop is a control-flow/termination problem, not a reasoning-quality one.

   </details>

---

## Further Reading

- [LangGraph Graph API overview (StateGraph, State, reducers, nodes, edges, Command, START/END, super-steps, recursion_limit)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — The authoritative reference for every primitive in this chapter, including the reducer merge semantics and the Pregel super-step execution model.
- [LangGraph Persistence overview (checkpointers vs stores)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29* — How `compile(checkpointer=...)` adds durable, thread-scoped state — the bridge to the HITL/durable-execution chapter.
- [LangChain Multi-agent handoffs (Command, Command.PARENT, subgraph transfers)](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs) — *verified 2026-07-29* — Shows how supervisor/handoff topologies are built directly on the `Command` primitive introduced here.
- [Pregel: A System for Large-Scale Graph Processing (Malewicz et al., 2010)](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) — *verified 2026-07-29* — The super-step / message-passing model LangGraph's execution engine is inspired by.
