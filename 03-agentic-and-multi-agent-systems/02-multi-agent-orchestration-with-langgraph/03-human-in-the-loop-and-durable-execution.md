# Human-in-the-Loop and Durable Execution in LangGraph

**Section:** 03 Agentic & Multi-Agent Systems → Multi-Agent Orchestration with LangGraph | **Est. time:** 3 hrs | **Interview relevance:** High — production agentic systems are judged on whether they can pause for a human approval before an irreversible action and survive a crash mid-run; if you can't explain checkpointing + `interrupt()`, you can't defend a real agentic architecture.

---

## TL;DR

By default a LangGraph run is ephemeral: state lives in memory and vanishes when the run ends or the process dies. Attaching a **checkpointer** changes that — the graph saves a snapshot of its state at every super-step, keyed by a `thread_id`, so a run becomes **durable** (survives crashes) and **resumable** (can continue later). On top of that persistence layer, the `interrupt()` function lets a node *pause* the graph to ask a human to approve, edit, or supply input; you resume by re-invoking the graph with `Command(resume=value)`, and that value becomes the return of the `interrupt()` call. The same checkpoints also enable **time travel** — replaying or forking from any past checkpoint via `get_state_history()`. **The one thing to remember: a checkpointer turns your agent from a one-shot function into a durable, pausable process — persistence is the foundation, and human-in-the-loop, crash recovery, and time travel are all features built on top of it.**

---

## ELI5 — Explain It Like I'm 5

Think of cooking a complicated meal from a recipe where, at one step, the rule says "before you add the expensive saffron, get the head chef to sign off." A checkpointer is like writing down *exactly* where you are on a sticky note after finishing each step — pot on, onions in, sauce reduced — so if the kitchen loses power, you don't start over from washing your hands; you read the last sticky note and carry on. When you reach the saffron step you literally stop, put the recipe down with your sticky-note bookmark, and go find the head chef; you might wait five minutes or you might wait until tomorrow, and that's fine because the bookmark holds your place. When the chef says "yes, go ahead" (or "no, skip it"), you pick the recipe back up at the bookmark and continue with their answer in hand. The common misconception is that the cook has to do the whole meal start-to-finish in one unbroken stretch — they don't; the bookmark is what lets a long recipe survive interruptions and wait on someone else without losing any progress.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain how a checkpointer persists graph state per super-step and why that makes execution durable and resumable.
- [ ] Choose the correct checkpointer backend (`InMemorySaver` vs `SqliteSaver` vs `PostgresSaver`) for dev versus production and justify the choice.
- [ ] Implement a human approval gate with `interrupt()` and resume it with `Command(resume=...)`, and explain why the interrupting node re-runs from its start on resume.
- [ ] Distinguish dynamic `interrupt()` (for HITL) from static `interrupt_before`/`interrupt_after` breakpoints (for debugging).
- [ ] Use `get_state`/`get_state_history` to inspect, replay, and fork from a past checkpoint (time travel), and diagnose why an in-memory checkpointer loses state on restart.

---

## Visual Overview

### Checkpoint per Super-Step (durability timeline)

```
run START ──► super-step 0 ──► super-step 1 ──► super-step 2 ──► END
                  │                │                │
                  ▼                ▼                ▼
              ┌────────┐       ┌────────┐       ┌────────┐
              │ ckpt A │       │ ckpt B │       │ ckpt C │   thread_id = "t1"
              └────────┘       └────────┘       └────────┘
   each StateSnapshot stores: values + next nodes + metadata(step)
   crash after step 1?  ──► restart re-loads ckpt B, resumes at step 2
                            (successful node writes are NOT re-run)
```

### interrupt() → human → Command(resume=...) flow

```
graph.invoke / stream_events
        │
        ▼
   node calls interrupt({...})
        │                                  state SAVED to checkpointer
        ├───────────────────────────────►  (thread_id holds the place)
        │                                  run pauses INDEFINITELY
        ▼
   payload surfaces to caller
   (__interrupt__ / stream.interrupts)
        │
        ▼
   HUMAN reviews  ──► decision (approve / edit / input)
        │
        ▼
   graph.invoke(Command(resume=decision), same thread_id)
        │
        ▼
   node RE-RUNS from its start; interrupt() now RETURNS decision
        │
        ▼
   graph continues ──► END
```

### Approve / Reject a Tool Call (HITL gate)

```
        model proposes irreversible tool call (e.g. transfer $500)
                        │
                        ▼
               approval_node: interrupt({question, details})
                        │
             ┌──────────┴───────────┐
    resume=True                  resume=False
             │                       │
             ▼                       ▼
       execute tool             skip / cancel
      (side effect fires)     (no side effect)
             │                       │
             └───────────┬───────────┘
                         ▼
                        END
```

### Time Travel: Replay vs Fork from a Checkpoint

```
history = get_state_history(config)   # newest-first list of StateSnapshots

   ckpt0 ──► ckpt1 ──► ckpt2 (final)
              │
     REPLAY   └─ invoke(None, ckpt1.config)
              │     └─ nodes AFTER ckpt1 re-execute; earlier ones skipped
              │
     FORK     └─ update_state(ckpt1.config, {new values}) ──► new branch
                    └─ invoke(None, fork_config)  explores alternate path
                       (original history stays intact)
```

---

## Key Concepts

### Persistence & Checkpointing

**What it is.** A checkpointer is a component you attach at compile time (`builder.compile(checkpointer=...)`) that saves a snapshot of the graph's state — a `StateSnapshot` — at each super-step boundary. This is LangGraph's short-term, thread-scoped memory.

**How it works mechanistically.** LangGraph executes in **super-steps**: one "tick" in which all nodes scheduled for that step run (possibly in parallel). After each super-step completes, the checkpointer writes a checkpoint containing the state channel `values`, the `next` nodes to execute, and `metadata` (including the `step` counter). Within a super-step, each node's output is also written at the task level to a writes table — these **pending writes** mean that if one node in a super-step fails, the successful nodes' outputs are already durable and are *not* re-run when you resume. This is what makes execution durable: a process crash after step 1 restarts by re-loading the last checkpoint and continuing at step 2 rather than from scratch.

**Where it appears in real systems.** In code it is `from langgraph.checkpoint.memory import InMemorySaver; graph = builder.compile(checkpointer=InMemorySaver())`. The checkpointer implements the `BaseCheckpointSaver` interface (`.put`, `.put_writes`, `.get_tuple`, `.list`). You inspect what it saved with `graph.get_state(config)` (latest snapshot) and `graph.get_state_history(config)` (all snapshots newest-first). This layer builds directly on the `StateGraph` primitives from chapter 01 — the same state channels, now snapshotted.

### Checkpointer Backends (InMemory / Sqlite / Postgres)

**What it is.** The concrete storage engine behind the checkpointer. LangGraph ships three: `InMemorySaver` (RAM), `SqliteSaver` (local file), and `PostgresSaver` (a real database), each a separate installable package.

**How it works mechanistically.** All three conform to the same `BaseCheckpointSaver` contract, so the graph code is identical regardless of backend — you only swap the object you pass to `compile()`. `InMemorySaver` keeps checkpoints in a Python dict in process memory: fast, zero setup, but **everything is lost when the process restarts**. `SqliteSaver` writes to a local `.db` file (durable across restarts on one machine, single-writer). `PostgresSaver` writes to Postgres with async support and is what LangSmith's managed platform uses; `checkpointer.setup()` creates the tables and indexes. Production checkpointers can also encrypt state at rest via `EncryptedSerializer`.

**Where it appears in real systems.** `InMemorySaver()` in tests and notebooks; `SqliteSaver(sqlite3.connect("checkpoints.db", check_same_thread=False))` for local single-user apps; `PostgresSaver.from_conn_string("postgresql://...")` followed by `.setup()` for a multi-replica production service. The documented failure — "`MemorySaver` does not persist between restarts" — is exactly why using `InMemorySaver` in production is the classic HITL/durability bug.

### Threads and `thread_id`

**What it is.** A thread is the unit of persistent state — a unique ID under which a sequence of runs accumulates its checkpoints. It represents one conversation, session, or workflow instance.

**How it works mechanistically.** You pass `config={"configurable": {"thread_id": "..."}}` on every invocation. The checkpointer uses `thread_id` as the primary key: reusing the same `thread_id` loads and continues that thread's saved state (this is how conversational memory and resume-after-interrupt work); using a new `thread_id` starts a fresh thread with empty state. The docs describe `thread_id` as "your persistent cursor" — it is literally the pointer that tells the checkpointer *which* saved state to load. Without a `thread_id`, a checkpointed graph cannot save or resume.

**Where it appears in real systems.** The `configurable.thread_id` config key. In a web backend you typically map it to a session/conversation ID (a UUID). Note the `PostgresSaver` constraint: `thread_id` is stored in a length-limited column, so keep it under 255 characters (use a UUID or hash).

### `interrupt()` and `Command(resume=...)`

**What it is.** `interrupt()` (from `langgraph.types`) is the function you call *inside a node* to pause the graph and surface a JSON-serializable payload to the caller; `Command(resume=value)` is what you pass back to the graph to continue, and `value` becomes the return of that `interrupt()` call.

**How it works mechanistically.** Calling `interrupt(payload)` raises a special exception that the runtime catches; it saves the current state via the checkpointer and the run pauses **indefinitely** (it can wait seconds or days). The payload appears to the caller under `result["__interrupt__"]` (with `invoke`) or on `stream.interrupts` (with `stream_events`). To continue, you re-invoke with the **same `thread_id`** passing `Command(resume=value)`. Critically, on resume LangGraph **re-runs the entire node from its beginning** — it does not resume from the exact line — so any code before the `interrupt()` executes again. This has a hard rule: side effects before an `interrupt()` must be idempotent (use upserts, not blind inserts), or place them after the interrupt.

**Where it appears in real systems.** An `approval_node` that calls `is_approved = interrupt({"question": "...", "details": ...})` then routes on the result (e.g. `Command(goto="proceed" if is_approved else "cancel")`). You can also place `interrupt()` *inside a tool* so the tool itself pauses for approval before executing — useful for a reusable "send email"/"transfer money" tool that must always be human-gated. A checkpointer is mandatory: `interrupt()` without one has nowhere to save the paused state.

### Static Breakpoints: `interrupt_before` / `interrupt_after`

**What it is.** Compile-time (or run-time) flags that pause the graph *before* or *after* named nodes, without any `interrupt()` call in your code.

**How it works mechanistically.** You pass `interrupt_before=["node_a"]` / `interrupt_after=["node_b"]` to `compile()` or to the invocation. The run stops at that node boundary and saves state; you resume by invoking with `None` as the input (not `Command`), which continues to the next breakpoint or the end. Because they pause at fixed node boundaries rather than at an arbitrary point in your logic, they are **static** — the docs explicitly recommend them for *debugging / stepping through a graph*, not for production HITL, for which the dynamic `interrupt()` function is preferred (it can be conditional and can carry a payload + resume value).

**Where it appears in real systems.** `builder.compile(interrupt_before=["tools"], checkpointer=...)` to inspect state before every tool call while debugging, or setting breakpoints in LangSmith Studio's UI to step through a run node by node.

### Time Travel: `get_state_history`, Replay, and Fork

**What it is.** The ability to list every past checkpoint of a thread and resume from any one of them — either **replaying** (re-run from that point unchanged) or **forking** (branch with modified state to explore an alternative).

**How it works mechanistically.** `graph.get_state_history(config)` returns the thread's `StateSnapshot`s newest-first, each carrying a `checkpoint_id` in its config and a `next` tuple showing what would run next. **Replay:** call `graph.invoke(None, some_checkpoint.config)` — nodes *before* that checkpoint are skipped (their results are saved) and nodes *after* re-execute (LLM calls, API calls, and interrupts all fire again and may differ). **Fork:** call `graph.update_state(checkpoint.config, values={...})` to write a *new* checkpoint that branches from that point (the original history is untouched), then `invoke(None, fork_config)` to run the alternate path. `update_state` respects reducers and accepts an `as_node` argument to control which node the update is attributed to (and thus what runs next).

**Where it appears in real systems.** Debugging a bad agent decision by replaying from the checkpoint just before it; A/B-testing a different prompt or tool result by forking; letting a human rewind a multi-step form to change an earlier answer. You find the target with a filter like `next(s for s in history if s.next == ("write_joke",))`.

### Durability Modes

**What it is.** A per-invocation setting (`durability="exit" | "async" | "sync"`) that trades checkpoint-write frequency against performance.

**How it works mechanistically.** `"exit"` persists only when the run exits (success, error, or interrupt) — fastest, but a mid-run crash loses intermediate state and cannot be recovered. `"async"` writes checkpoints asynchronously while the next step runs — good balance, tiny risk of a lost write on crash. `"sync"` writes each checkpoint synchronously *before* the next step starts — highest durability, some latency cost. You set it like `graph.stream({...}, durability="sync")`.

**Where it appears in real systems.** Use `"sync"` for workflows where a mid-run crash must be fully recoverable and steps have irreversible side effects; use `"async"` (or the default) for typical latency-sensitive chat; `"exit"` only for short runs where re-running from scratch is acceptable.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Checkpointer backend | Where graph state is persisted | `InMemorySaver` only in tests/notebooks; `SqliteSaver` for local single-machine dev; `PostgresSaver` (or a managed platform) for any production/multi-replica deployment — never ship `InMemorySaver` if you need HITL or crash recovery. |
| `thread_id` (`configurable`) | Which persisted state to load/resume | Scope one `thread_id` per conversation/session/workflow instance; reuse it to resume or continue memory, generate a new one to start fresh; keep under 255 chars for `PostgresSaver`. |
| `interrupt()` placement | Where in a node the graph pauses for a human | Place it *before* any irreversible side effect (send money, delete data, external write); keep pre-`interrupt()` code idempotent because the node re-runs on resume. |
| Static `interrupt_before` / `interrupt_after` | Fixed node-boundary breakpoints | Use for debugging/stepping only; for production human approval use dynamic `interrupt()` instead (conditional + carries payload). |
| `durability` (`exit`/`async`/`sync`) | Checkpoint-write frequency vs latency | `sync` when a mid-run crash must be recoverable and side effects are irreversible; `async`/default for latency-sensitive chat; `exit` only for short, safely-restartable runs. |
| Checkpoint retention (pruning) | Storage growth of long threads | Add a retention job (e.g. delete checkpoints older than N days) for long-lived threads; unbounded accumulation raises latency and storage cost. |

### Worked Example: Requirement → Decision

**Given:** You are building a customer-support agent that can issue refunds. When it decides to refund a customer, it calls an irreversible `issue_refund(account_id, amount)` tool that moves real money. Policy requires: any refund over $100 must be approved by a human agent, who may also *edit the amount* down before it executes; approvals can take minutes to hours (the human is off doing other work); and the service runs on multiple replicas behind a load balancer and gets redeployed frequently, so a run may outlive the process it started on.

- **Step 1 — Identify the goal:** Pause the graph before executing a high-value, irreversible refund, let a human approve/edit/reject, then resume with their decision — and guarantee the paused run survives restarts and redeploys.
- **Step 2 — Define inputs:** The proposed `issue_refund` call (account_id, amount); a durable checkpointer; a stable `thread_id` (the support-case ID); the human's decision passed back as `Command(resume=...)`.
- **Step 3 — Define outputs:** Either the refund executed at the approved amount, or a cancelled/edited outcome — never a refund that fired without approval, and never a lost paused run.
- **Step 4 — Apply constraints:** Approval latency is unbounded (minutes→hours) so state must persist indefinitely; the tool is irreversible so the gate must sit *before* the side effect; multi-replica + redeploys mean in-memory state is unacceptable; the resume must land on the exact paused case (correct `thread_id`).
- **Step 5 — Select the approach:** Compile with `PostgresSaver` (durable, shared across replicas), gate the refund in an `approval_node` that calls `interrupt({...})` *before* the tool executes, and resume with `Command(resume={"approved": True, "amount": edited})`. *Rationale vs alternatives:* `InMemorySaver` is wrong because a redeploy during the hours-long wait would silently drop the paused case; a static `interrupt_before` is weaker because it can't carry the refund details in a payload or accept an edited amount on resume; running the tool first and "undoing" it on rejection is impossible for an irreversible transfer — the interrupt must precede the side effect.

---

## Implementation

```python
# Scenario: A support agent can issue irreversible refunds. Any refund must be
# human-approved (and optionally edited) BEFORE the money moves, the paused run
# must survive process restarts (hours-long waits + redeploys), and resuming must
# land on the exact case. We gate with interrupt() before the side effect and
# persist with a durable checkpointer.
# API verified against LangGraph interrupts + checkpointers docs (docs.langchain.com).
import sqlite3
from typing import TypedDict, Literal, Optional
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.sqlite import SqliteSaver   # SqliteSaver for dev; PostgresSaver in prod
from langgraph.types import Command, interrupt

class RefundState(TypedDict):
    account_id: str
    amount: float
    status: Optional[Literal["pending", "refunded", "cancelled"]]

def approval_node(state: RefundState) -> Command[Literal["execute", "cancel"]]:
    # PAUSE before the irreversible action. Payload surfaces to the caller so a
    # human UI can render the details and approve / edit / reject.
    decision = interrupt({
        "question": "Approve this refund?",
        "account_id": state["account_id"],
        "amount": state["amount"],
    })
    if not decision.get("approved"):
        return Command(goto="cancel")
    # Human may edit the amount down; resume value overrides state.
    return Command(goto="execute", update={"amount": decision.get("amount", state["amount"])})

def execute_node(state: RefundState):
    issue_refund(state["account_id"], state["amount"])   # irreversible side effect, runs ONCE post-approval
    return {"status": "refunded"}

def cancel_node(state: RefundState):
    return {"status": "cancelled"}

builder = StateGraph(RefundState)
builder.add_node("approval", approval_node)
builder.add_node("execute", execute_node)
builder.add_node("cancel", cancel_node)
builder.add_edge(START, "approval")
builder.add_edge("execute", END)
builder.add_edge("cancel", END)

checkpointer = SqliteSaver(sqlite3.connect("refunds.db", check_same_thread=False))
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "case-4491"}}     # thread_id = the support case
result = graph.invoke({"account_id": "acct-1", "amount": 250.0, "status": "pending"}, config)
print(result["__interrupt__"])            # -> paused, awaiting human approval

# ...minutes or hours later, on a possibly-different replica, SAME thread_id...
final = graph.invoke(
    Command(resume={"approved": True, "amount": 100.0}),  # human edited 250 -> 100
    config,
)
print(final["status"])                    # -> "refunded" (at the approved $100)
```

```python
# Anti-pattern: auto-execute an irreversible tool with NO approval gate, and
# (compounding it) use InMemorySaver so a paused/interrupted run is lost on restart.
# There is no human checkpoint before money moves, and even if you added interrupt(),
# InMemorySaver would drop the paused state on the next deploy.
from langgraph.checkpoint.memory import InMemorySaver

def refund_node(state):                                   # BROKEN: no interrupt, fires immediately
    issue_refund(state["account_id"], state["amount"])    # irreversible, no human sign-off
    return {"status": "refunded"}

builder.add_node("refund", refund_node)
graph = builder.compile(checkpointer=InMemorySaver())     # BROKEN: state lost on restart

# Correct approach: interrupt() BEFORE the side effect (human gate) AND a durable
# checkpointer so the pause survives restarts/redeploys.
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt

def refund_node(state):
    approved = interrupt({"question": "Approve refund?", "amount": state["amount"]})  # PAUSE first
    if not approved:
        return {"status": "cancelled"}
    issue_refund(state["account_id"], state["amount"])    # runs only after approval, exactly once
    return {"status": "refunded"}

checkpointer = PostgresSaver.from_conn_string("postgresql://...")
checkpointer.setup()                                      # create tables/indexes once
graph = builder.compile(checkpointer=checkpointer)
# What breaks without this: (1) with no interrupt, every refund fires unreviewed and
# cannot be undone; (2) with InMemorySaver, any run paused for human input is silently
# lost when the process restarts — the customer's case vanishes mid-approval. The fix
# makes the money-move human-gated AND the paused case durable across deploys.
# NOTE: because the node re-runs from its start on resume, keep pre-interrupt() code
# idempotent — here issue_refund() sits AFTER interrupt(), so it fires exactly once.
```

---

## Common Pitfalls & Misconceptions

- **Using `InMemorySaver` in production** — it's the default in every tutorial and "just works" in a notebook, so it feels production-ready. It stores checkpoints in RAM and loses everything on restart; any run paused for hours-long human approval is silently dropped on the next deploy. Use `SqliteSaver` for local dev and `PostgresSaver` (or a managed platform) for anything real.
- **Assuming the node resumes from the exact `interrupt()` line** — resume "continuing where it left off" sounds like it picks up on the next line. LangGraph re-runs the **entire node from its beginning** on resume, so any code before the `interrupt()` executes again; place irreversible side effects *after* the interrupt (or make pre-interrupt code idempotent with upserts).
- **Wrapping `interrupt()` in a bare `try/except`** — defensive habit says wrap risky calls in try/except. `interrupt()` pauses by *raising* a special exception; a bare `except Exception` swallows it and the graph never pauses. Keep `interrupt()` out of broad try/except blocks (catch only specific exception types for surrounding code).
- **Forgetting the `thread_id` (or reusing the wrong one)** — with in-memory testing it seems optional. A checkpointed graph *requires* `thread_id` to know which state to save/load; omitting it breaks resume, and reusing one case's `thread_id` for another mixes their state. Scope one `thread_id` per conversation/workflow instance.
- **Using static `interrupt_before` for production approvals** — it looks like the built-in "pause" feature. Static breakpoints can't be conditional and can't carry a review payload or accept a typed resume value; they're for debugging/stepping. Use the dynamic `interrupt()` function for real human-in-the-loop approve/edit/input flows.
- **Expecting replay to be a cached no-op** — "replay from a checkpoint" sounds like it just re-reads saved outputs. Replay re-*executes* every node after the checkpoint, so LLM calls, API requests, and interrupts all fire again and may return different results; only nodes *before* the checkpoint are skipped.

---

## Key Definitions

| Term | Definition |
|---|---|
| Checkpointer | A `BaseCheckpointSaver` attached at `compile()` that saves a state snapshot per super-step, enabling durability, HITL, memory, and time travel. |
| Checkpoint / `StateSnapshot` | A saved snapshot of graph state at one super-step: channel `values`, `next` nodes, and `metadata` (including `step`). |
| Super-step | One "tick" of the graph where all scheduled nodes run; a checkpoint is written at each super-step boundary. |
| Thread / `thread_id` | The persistent-state unit; `thread_id` (in `configurable`) is the primary key the checkpointer uses to save/load a run's state. |
| `InMemorySaver` / `SqliteSaver` / `PostgresSaver` | Checkpointer backends: RAM (dev/tests), local file (local dev), and Postgres (production). |
| `interrupt()` | A function called inside a node that pauses the graph, surfaces a JSON payload to the caller, and waits indefinitely for resume. |
| `Command(resume=value)` | The input you pass to re-invoke a paused graph; `value` becomes the return of the `interrupt()` call inside the node. |
| `interrupt_before` / `interrupt_after` | Static breakpoints that pause before/after named nodes; resumed by invoking with `None` — for debugging, not production HITL. |
| Pending writes | Node outputs persisted within a super-step so a partial-failure resume doesn't re-run already-successful nodes. |
| `get_state` / `get_state_history` | Methods returning the latest / all `StateSnapshot`s for a thread — the basis for inspection and time travel. |
| Replay / Fork | Time-travel operations: re-run from a past checkpoint unchanged (replay) or branch with modified state via `update_state` (fork). |
| Durability mode | Per-run `exit`/`async`/`sync` setting trading checkpoint-write frequency against latency. |

---

## Summary / Quick Recall

- A **checkpointer** saves state **per super-step** keyed by `thread_id`, turning a one-shot run into a **durable, resumable** process (crash recovery + resume-hours-later).
- Backend choice: `InMemorySaver` (tests only — lost on restart), `SqliteSaver` (local dev), `PostgresSaver` (production/multi-replica).
- `interrupt()` pauses a node and surfaces a payload; resume with `Command(resume=value)` on the **same `thread_id`**, and that value becomes the `interrupt()` return.
- On resume the **whole node re-runs from its start** — put irreversible side effects *after* the `interrupt()`; keep pre-interrupt code idempotent; never wrap `interrupt()` in a bare `try/except`.
- Use dynamic `interrupt()` for production HITL (approve / edit / input); use static `interrupt_before`/`interrupt_after` only for debugging.
- **Time travel** via `get_state_history` → **replay** (re-run after a checkpoint) or **fork** (`update_state` to branch); replay re-executes nodes, it is not a cached read.
- The HITL gate must sit **before** any irreversible action; a durable checkpointer is mandatory for both approval waits and crash recovery.

---

## Self-Check Questions

1. What does a checkpointer save, at what granularity, and what does the `thread_id` do?

   <details><summary>Answer</summary>

   A checkpointer saves a **`StateSnapshot` of the graph's state at each super-step boundary** (channel `values`, `next` nodes, and `metadata` including the step counter). The **`thread_id`** (in `config["configurable"]`) is the primary key the checkpointer uses to store and retrieve those snapshots — reusing it resumes/continues that thread's state, a new one starts fresh. The tempting wrong answer is "it saves state after every line of code / every node call individually" — checkpoints are written at super-step boundaries (a tick where all scheduled nodes run), not per line; within a step only task-level *pending writes* are stored for partial-failure recovery.

   </details>

2. You need an agent to pause before executing an irreversible `delete_records` tool, wait for a human (possibly hours), then continue with their yes/no. Which LangGraph mechanism do you use, and what must you pass to resume?

   <details><summary>Answer</summary>

   Call **`interrupt(payload)` inside the node before the tool executes**, compiled with a **durable checkpointer** and a stable `thread_id`; the run pauses and saves state. To resume, re-invoke the graph with **`Command(resume=decision)`** on the **same `thread_id`** — that value becomes the return of the `interrupt()` call. The tempting wrong answer is "use `interrupt_before=['delete']`" — static breakpoints are for debugging, can't carry the review payload, and are resumed with `None` rather than a decision value; they don't fit a real approve/reject HITL flow.

   </details>

3. **Which TWO** of the following are true about `interrupt()` and resuming a graph?
   - A. On resume, the node containing `interrupt()` re-runs from its beginning, so code before the interrupt executes again.
   - B. `interrupt()` works even without a checkpointer, since it holds state in the function's local variables.
   - C. The value passed to `Command(resume=...)` becomes the return value of the `interrupt()` call.
   - D. Wrapping `interrupt()` in a bare `try/except Exception` is the recommended way to handle resume errors.
   - E. A paused run must be resumed with a *different* `thread_id` than it started with.

   <details><summary>Answer</summary>

   **A and C.** A is correct because LangGraph pauses by raising a special exception and, on resume, restarts the whole node — so pre-`interrupt()` code runs again (hence the idempotency rule). C is correct by definition: the resume value is injected as the return of `interrupt()`. B is false — `interrupt()` requires a checkpointer to persist the paused state (there is nowhere else to save it). D is the most tempting distractor and is wrong: a bare `try/except Exception` *swallows* the interrupt's special exception so the graph never pauses; catch only specific exception types around other code. E is false — you must resume with the **same** `thread_id` so the checkpointer loads the right paused state.

   </details>

4. Your HITL refund agent runs fine in staging with `InMemorySaver`, but in production, refunds occasionally "disappear mid-approval": a human approves later and nothing happens. What is the most likely root cause and the fix?

   <details><summary>Answer</summary>

   The paused runs are being lost because **`InMemorySaver` stores checkpoints in RAM**, and production redeploys/restarts (or the request landing on a different replica) wipe that memory — so when the human approves hours later, there is no saved state to resume. The fix is a **durable, shared checkpointer**: `PostgresSaver` (call `.setup()` once) so the paused case persists across restarts and is visible to all replicas. Swapping the model or adding retries wouldn't help — the state is simply gone; only durable persistence keeps a long-pending approval alive.

   </details>

5. A teammate proposes putting the actual `charge_card()` side effect *before* the `interrupt()` call in the same node "so the charge is ready the moment the human approves." Evaluate this against placing it *after* the interrupt.

   <details><summary>Answer</summary>

   Putting `charge_card()` **before** the `interrupt()` is a bug: on resume LangGraph **re-runs the node from its beginning**, so any code before the `interrupt()` executes again — the card would be charged on the *initial* run (before any approval) and potentially **charged again on resume**, producing an unauthorized and/or duplicate charge. The correct placement is **after** the `interrupt()`, where the side effect runs exactly once, only following approval. (If a side effect genuinely must precede the interrupt, it must be idempotent — e.g. an upsert — but a card charge is not.) The tempting wrong intuition is that resume "continues where it left off" and won't repeat earlier lines; LangGraph's node-level replay semantics make that assumption false.

   </details>

---

## Further Reading

- [LangGraph Persistence (checkpointers vs stores, quickstart, durability modes)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29* — Overview of the persistence layer, `InMemorySaver`/`SqliteSaver`/`PostgresSaver` selection, `durability` modes, and the "MemorySaver doesn't persist between restarts" pitfall.
- [LangGraph Checkpointers (super-steps, threads, get_state/get_state_history, pending writes)](https://docs.langchain.com/oss/python/langgraph/checkpointers) — *verified 2026-07-29* — Reference for `StateSnapshot` fields, super-step checkpointing, thread semantics, backend libraries, and the `BaseCheckpointSaver` interface.
- [LangGraph Interrupts (`interrupt()`, `Command(resume=...)`, HITL patterns, rules)](https://docs.langchain.com/oss/python/langgraph/interrupts) — *verified 2026-07-29* — Canonical guide to pausing/resuming, approve-reject/review-edit/validate patterns, interrupts-in-tools, static breakpoints, and the node-re-run + idempotency rules.
- [LangGraph Use Time Travel (replay and fork from a checkpoint)](https://docs.langchain.com/oss/python/langgraph/use-time-travel) — *verified 2026-07-29* — How to `get_state_history`, replay from a prior checkpoint, and fork with `update_state`/`as_node`, including behaviour with interrupts and subgraphs.
