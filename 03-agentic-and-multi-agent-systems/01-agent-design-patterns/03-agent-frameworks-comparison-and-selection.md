# Agent Frameworks: Comparison and Selection

**Section:** 03 Agentic & Multi-Agent Systems → Agent Design Patterns | **Est. time:** 2.5 hrs | **Interview relevance:** High — "which framework would you use and why" is a near-guaranteed follow-up, and the strong answer is a criteria-based selection, not brand loyalty.

---

## TL;DR

Every agent is the same core loop — a model calls tools in a loop until the task is done — so the real question is *how much of that loop you build yourself vs. hand to a framework*. Three tiers matter: a **hand-written tool-calling loop** on the provider's native API (maximum control, zero lock-in, you own retries/state), **LangChain's `create_agent`** prebuilt harness (the loop plus middleware for memory, retries, HITL, structured output), and **LangGraph** (a low-level orchestration runtime for durable, stateful, human-in-the-loop, multi-agent graphs). Other options (OpenAI Agents SDK, CrewAI, AutoGen, LlamaIndex agents) occupy points on the same control-vs-abstraction spectrum. The selection is driven by concrete needs — do you need persistence across failures? human approval gates? multiple coordinating agents? — not by which framework is newest. **The one thing to remember: pick the least machinery that meets your durability, HITL, and multi-agent requirements — a raw loop is the right answer more often than interview candidates think, and you only climb the abstraction ladder when a specific requirement forces you to.**

---

## ELI5 — Explain It Like I'm 5

Imagine you need something to keep your food cold. You have three choices: buy a finished refrigerator off the shop floor and just plug it in, buy a build-it-yourself kit where the parts are pre-cut and you snap them together, or buy raw metal and wiring and build a cooler from scratch. The finished appliance is fastest to start but you can only change what its dials allow; the kit lets you swap some parts; the raw build gives you total freedom but you have to solder every wire and make sure it doesn't leak. The mistake people make is grabbing the newest, fanciest, most-talked-about appliance because it looks impressive — when their actual need is "keep one lunch cold for an afternoon," and a cheap cooler (the raw build, but tiny) would do the job with nothing to break. The right choice depends entirely on *what you need to keep cold and for how long*, not on which box has the shiniest sticker.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the common "model calls tools in a loop" core that every agent framework wraps, and place the major options on a control-vs-abstraction spectrum.
- [ ] Describe LangGraph, LangChain's `create_agent`, and the no-framework native tool-calling loop in terms of what each buys and costs.
- [ ] Apply a needs-based decision procedure (persistence? HITL? multi-agent? one tool loop?) to select a framework for a given requirement.
- [ ] Diagnose and correct framework-selection anti-patterns such as reaching for a heavy multi-agent framework for a single-tool task or hand-rolling durable state a checkpointer already provides.
- [ ] Compare frameworks on lock-in, observability, and team-familiarity trade-offs and defend a choice under stated constraints.

---

## Visual Overview

### Control vs. Abstraction Spectrum

```
 MORE CONTROL / LESS ABSTRACTION                        LESS CONTROL / MORE ABSTRACTION
 you own the loop, state, retries                       framework owns the loop & scaffolding
 ◄─────────────────────────────────────────────────────────────────────────────────────►

 native tool-calling      LangGraph            LangChain          opinionated multi-agent
 loop (no framework)      (StateGraph:         create_agent       frameworks (CrewAI /
   provider API only      you wire nodes/      (prebuilt loop     AutoGen / OpenAI Agents
   + your own while-loop   edges/persistence)  + middleware)      SDK / LlamaIndex agents)

 LangGraph is "low-level orchestration": more structure than a raw loop, but you still
 wire the graph — it sits left of the prebuilt harnesses, not right of them.
```

### The Common Core Every Framework Wraps

```
                 ┌───────────────────────────────────────┐
   user input ──►│  model.generate( messages + tools )    │◄────────┐
                 └───────────────────────────────────────┘         │
                                │                                    │
                    tool_calls in response?                          │
                     ├── NO ──► return final answer                  │
                     └── YES ─► run tool(s) ──► append tool result ──┘
                                                   (loop until no tool_calls,
                                                    or a step/iteration cap)
   Every framework — and a hand-written loop — implements THIS. The differences are
   what they add around it: state persistence, HITL pauses, retries, multi-agent routing.
```

### Framework Selection Decision Tree

```
Do you need to persist/resume state across process restarts or long-running runs,
OR pause for human approval mid-run (HITL), OR coordinate multiple agents?
├── NO  ──► Is it one/a-few tools, short-lived, single provider?
│           ├── YES ──► Hand-written native tool-calling loop (least machinery, no lock-in)
│           └── "want batteries: memory, retries, structured output, but still simple"
│                      ──► LangChain create_agent (prebuilt harness)
└── YES ──► Do you need FINE-GRAINED control over the graph
            (deterministic + agentic steps mixed, custom branches, durable checkpoints)?
            ├── YES ──► LangGraph (StateGraph + checkpointer + interrupts)  ◄─ repo default
            └── "I want an opinionated, higher-level multi-agent abstraction"
                       ──► OpenAI Agents SDK / CrewAI / AutoGen / LlamaIndex agents
                          (accept their model & lock-in in exchange for less wiring)
```

---

## Key Concepts

### The Common Agent Core (why "which framework" is a wrapping question)

**What it is.** An agent is a model calling tools in a loop until a task is complete; every framework is scaffolding *around* that loop, not a different mechanism.

**How it works mechanistically.** On each turn the model receives the message history plus tool schemas and returns either a final answer or one-or-more structured `tool_calls`. A driver inspects the response: if there are tool calls, it executes them, appends the results as tool messages, and re-invokes the model; if not, it returns. This loop is identical whether you hand-write it or a framework runs it — LangChain's own docs define an agent as exactly "a model calling tools in a loop until a given task is complete," and note the *harness* (prompt, tools, middleware) is everything around that loop.

**Where it appears in real systems.** In a raw loop it's a `while` with `client.responses.create(..., tools=...)` and a check on `response.output` for tool calls; in LangChain it's `create_agent(model, tools)`; in LangGraph it's a `StateGraph` whose model node and `ToolNode` are connected by a conditional edge that routes on whether the last message has `tool_calls`. Recognizing this shared core is what lets you argue framework choice on *scaffolding needs* rather than hype.

### No Framework — Native Provider Tool-Calling Loop

**What it is.** Using the model provider's own tool-calling API (e.g. OpenAI's Responses/Chat Completions, Anthropic's Messages) inside a `while` loop you write yourself, with no agent library at all.

**How it works mechanistically.** You pass tool JSON schemas on each request; the model returns tool-call objects; your code dispatches to the matching Python function, appends the result to the message list, and calls again until the model answers with no tool call. You own everything: the iteration cap, error handling, retries, and any state you want to keep between turns. OpenAI's docs explicitly frame the Responses API as the "own the loop, tool dispatch, and state handling yourself" option, versus the Agents SDK when you want a runtime to manage turns.

**Where it appears in real systems.** A single-tool support endpoint, a scripted data-enrichment job, or any place where a dependency-light, fully-auditable, portable loop matters more than convenience. The cost is that features frameworks give free (durable state, HITL pauses, streaming plumbing, retry middleware) become your code to write and maintain — worth it only while those needs are minimal.

### LangChain Agents — `create_agent` (the prebuilt harness)

**What it is.** LangChain's high-level, prebuilt agent constructor: `create_agent(model, tools, system_prompt=...)` gives you the tool-calling loop plus a configurable "harness" of middleware.

**How it works mechanistically.** `create_agent` assembles the standard loop and lets you extend it via composable **middleware** — each piece hooks the loop at the right moment: summarization for context management, `ModelRetryMiddleware`/`ToolRetryMiddleware` for fault tolerance, human-in-the-loop approval steps, PII guardrails, and structured output via `response_format=`. Under the hood it is built on LangGraph, so passing a `checkpointer` (e.g. `InMemorySaver`) plus a `thread_id` gives conversation persistence without you wiring a graph. (Note: the older prebuilt was named `create_react_agent`; current LangChain docs standardize on `create_agent` — verify the exact import against the version you install.)

**Where it appears in real systems.** A chatbot or task agent that needs "batteries included" — memory, retries, structured responses — but not a custom control graph. It's the recommended starting point in LangChain's own docs for "common LLM and tool-calling loops," with LangGraph reserved for when you need lower-level control.

### LangGraph — Graph Orchestration, Durability, HITL (repo default)

**What it is.** A low-level orchestration framework and runtime for building long-running, **stateful** agents as an explicit graph of nodes and edges — this repo's primary framework.

**How it works mechanistically.** You define a `StateGraph` over a typed state schema, add nodes (functions/LLM calls/`ToolNode`), and connect them with normal and *conditional* edges to mix deterministic, hand-coded steps with LLM-driven steps in one graph. Compiling with a **checkpointer** persists a snapshot of state after each step, keyed by `thread_id`, which is what enables **fault-tolerant resume**, **time-travel**, and **human-in-the-loop** pauses (via `interrupt`s that suspend the run for inspection/edits). A separate **store** holds long-term, cross-thread memory. LangGraph does *not* abstract your prompts or architecture — it deliberately stays low-level and can be used without LangChain.

**Where it appears in real systems.** Durable multi-step workflows (a run that survives a crash and resumes), approval-gated actions (pause before a high-impact tool), and multi-agent systems where agents are subgraphs — the exact territory of the next chapter (section 03 chapter 02, multi-agent orchestration with LangGraph). Primitives: `StateGraph`, `add_conditional_edges`, `ToolNode`, `compile(checkpointer=..., store=...)`, `interrupt`.

### Other Notable Frameworks (described conservatively)

**What they are.** Higher-level or more opinionated agent frameworks that trade wiring for a prescribed model of how agents work.

**How they work mechanistically (per official docs).**
- **OpenAI Agents SDK** — a lightweight, few-primitive SDK: **Agents** (LLM + instructions + tools), **handoffs / agents-as-tools** for delegation, **guardrails** for input/output validation, and **sessions** for memory, with a built-in agent loop and tracing. It uses OpenAI's Responses API by default but supports other providers.
- **CrewAI** — organizes work as **Crews** (teams of role-playing agents that collaborate on a task) orchestrated by **Flows** (event-driven, stateful workflows with control flow and state management).
- **AutoGen (Microsoft)** — **AgentChat** for conversational single/multi-agent apps built on an event-driven **Core** runtime for scalable multi-agent systems; ships extensions (e.g. MCP, code executors).
- **LlamaIndex agents** — `FunctionAgent` (a function/tool-calling agent), `ReActAgent`/`CodeActAgent` (different prompting strategies), and `AgentWorkflow` for managing multiple agents; tightly integrated with LlamaIndex retrieval.

**Where they appear in real systems.** Choose these when their built-in model of orchestration matches your problem (e.g. OpenAI-native stack → Agents SDK; role-based collaborative teams → CrewAI; research-style conversational multi-agent → AutoGen; retrieval-centric agents → LlamaIndex). The trade-off is accepting each framework's abstractions and portability cost.

### Selection Criteria / Configuration Knobs

| Criterion | What it drives | Decision rule |
|---|---|---|
| Control granularity | How much of the loop/graph you can shape (custom branches, mixed deterministic + agentic steps) | Need explicit per-step control or to interleave hand-coded logic → LangGraph or a raw loop; happy with a standard loop → `create_agent` or an opinionated SDK. |
| Durability / persistence | Whether state survives crashes/restarts and long runs can resume | Runs are long-lived or must resume after failure → use a persistent checkpointer (LangGraph, or `create_agent` with a `PostgresSaver`); short-lived request/response → a raw loop is fine. |
| Human-in-the-loop (HITL) | Ability to pause mid-run for human approval/edit before continuing | Any high-impact action needs sign-off → LangGraph `interrupt`s (or `create_agent` HITL middleware); no approval gates → don't pay for HITL machinery. |
| Multi-agent support | Coordinating several specialized agents (routing, delegation, handoffs) | Multiple coordinating agents → LangGraph subgraphs or an opinionated multi-agent framework; one agent, few tools → a raw loop or `create_agent`. |
| Observability | Tracing steps, tool calls, and state transitions for debugging/eval | Production debugging/eval is required → prefer frameworks with first-class tracing (LangGraph+LangSmith, Agents SDK tracing); a raw loop means you instrument it yourself. |
| Lock-in / portability | How hard it is to swap models/providers or leave the framework | Portability or multi-provider is a hard requirement → raw loop or provider-agnostic layers (LangGraph is usable without LangChain); provider-tied SDKs maximize convenience but bind you. |
| Team familiarity | Ramp-up cost and maintainability for your team | Team already fluent in one stack and it meets the needs → use it; don't introduce a new framework whose only advantage is novelty. |

### Worked Example: Requirement → Decision

**Given:** You are building an internal "operations copilot" that executes multi-step tasks (query systems, draft a change, then **apply an infrastructure change**). Requirements: the destructive apply step must **pause for human approval**; a run may take minutes and must **resume cleanly if the worker restarts**; you will later add a second, specialized "reviewer" agent. Latency is not tight; the team already uses Python and Postgres.

- **Step 1 — Identify the goal:** Reliable, auditable multi-step execution with a mandatory human gate before the irreversible action, surviving restarts, extensible to multiple agents.
- **Step 2 — Define inputs:** User task, tool set (read APIs + a privileged "apply" tool), a persistence backend (Postgres available), a state schema tracking progress and approval status.
- **Step 3 — Define outputs:** A completed task with an audit trail, or a run **paused at the approval interrupt** awaiting a human decision, resumable by `thread_id`.
- **Step 4 — Apply constraints:** HITL is mandatory (not optional); durable resume across restarts is required; a second agent is on the near-term roadmap; latency is generous; portability is not a stated hard requirement.
- **Step 5 — Select the approach:** Use **LangGraph** — a `StateGraph` with a `PostgresSaver` checkpointer gives durable resume by `thread_id`, an `interrupt` before the apply node gives the human gate for free, and agents-as-subgraphs cleanly extends to the reviewer later. *Rationale vs alternatives:* a raw loop would force you to hand-build durable checkpointing and pause/resume (exactly the machinery that breaks silently); `create_agent` could do HITL + persistence for a single agent but you'd fight it once you need custom branch control and multi-agent routing; an opinionated framework like CrewAI/Agents SDK adds lock-in without giving finer graph control than LangGraph, which is why the repo standardizes on LangGraph for this class of problem.

---

## Implementation

```python
# Scenario: A price-lookup helper needs exactly ONE tool and lives behind a stateless
# HTTP endpoint (no persistence, no HITL, no second agent, one provider). The right
# choice is the LEAST machinery: a hand-written native tool-calling loop, no agent
# library — zero extra dependencies, fully portable, trivially auditable.
# Pattern verified against OpenAI Agents SDK docs ("own the loop" = Responses API directly).
import json
from openai import OpenAI

client = OpenAI()
TOOLS = [{
    "type": "function",
    "name": "get_price",
    "description": "Return the current price for a product SKU.",
    "parameters": {"type": "object", "properties": {"sku": {"type": "string"}},
                   "required": ["sku"]},
}]

def get_price(sku: str) -> str:
    return json.dumps({"sku": sku, "price": 42.00})

def run(user_msg: str, max_steps: int = 3) -> str:
    messages = [{"role": "user", "content": user_msg}]
    for _ in range(max_steps):                      # bound the loop — never trust it to self-halt
        resp = client.responses.create(model="gpt-4o-mini", tools=TOOLS, input=messages)
        calls = [o for o in resp.output if getattr(o, "type", None) == "function_call"]
        if not calls:                               # no tool call -> model is done
            return resp.output_text
        for c in calls:                             # dispatch, append result, loop
            result = get_price(**json.loads(c.arguments))
            messages += [c, {"type": "function_call_output",
                             "call_id": c.call_id, "output": result}]
    return "Stopped: step budget exhausted."        # deterministic worst case
```

```python
# Anti-pattern: reaching for a heavy multi-agent framework (or hand-rolling durable
# state + retries) for a task that needs neither. Here someone hand-builds crash-safe
# checkpointing and manual retry bookkeeping for a long-running agent — re-implementing,
# buggily, exactly what a LangGraph checkpointer already provides.
# WRONG: DIY persistence smeared through the loop
state = load_state_from_disk() or {"messages": [], "step": 0}   # bespoke, untested format
while not done:
    try:
        step(state)
    except Exception:
        state["retries"] = state.get("retries", 0) + 1          # ad-hoc retry counter
    save_state_to_disk(state)                                    # no atomicity, races on restart
# What breaks: partial writes corrupt state on crash, there's no time-travel/inspection,
# resume semantics are undefined, and every new requirement (HITL, a 2nd agent) means
# more bespoke plumbing. This is the "build a fridge from raw metal for one lunch" mistake
# — but inverted: too MUCH DIY where a framework primitive is the right amount.

# Correct approach: use LangGraph's checkpointer — durable, resumable, inspectable state
# keyed by thread_id, so resume-after-crash and HITL come from primitives, not hand code.
# API verified against docs.langchain.com LangGraph persistence + overview.
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.postgres import PostgresSaver
from langchain.chat_models import init_chat_model
from langchain.tools import tool

@tool
def apply_change(plan: str) -> str:
    """Apply an infrastructure change described by `plan`."""
    return "applied"

model = init_chat_model("openai:gpt-4o-mini", temperature=0)

def call_model(state: MessagesState):
    return {"messages": [model.bind_tools([apply_change]).invoke(state["messages"])]}

def route(state: MessagesState):
    return "tools" if getattr(state["messages"][-1], "tool_calls", None) else END

builder = StateGraph(MessagesState)
builder.add_node(call_model)
builder.add_node("tools", ToolNode([apply_change]))
builder.add_edge(START, "call_model")
builder.add_conditional_edges("call_model", route, {"tools": "tools", END: END})
builder.add_edge("tools", "call_model")

checkpointer = PostgresSaver.from_conn_string("postgresql://...")   # durable, atomic
graph = builder.compile(checkpointer=checkpointer)                  # resume by thread_id
# Run/resume the SAME conversation across restarts by reusing the thread_id:
graph.invoke({"messages": [{"role": "user", "content": "apply the staging patch"}]},
             {"configurable": {"thread_id": "run-123"}})
# What this fixes: persistence, atomic checkpoints, resume, and (via interrupts) HITL are
# framework primitives — no bespoke state file, no ad-hoc retry counter, no corruption on crash.
```

---

## Common Pitfalls & Misconceptions

- **Framework cargo-culting ("use the newest/most popular one")** — beginners equate the trendiest framework with the best engineering choice because that's what conference talks and threads amplify. The correct mental model is that all frameworks wrap the same tool-calling loop, so the right one is whichever adds *exactly* the scaffolding your requirements demand (persistence, HITL, multi-agent) and no more.
- **Confusing LangChain with LangGraph** — people treat them as competitors or interchangeable because they share a vendor and often appear together. LangChain is the agent framework (components + the `create_agent` prebuilt loop); LangGraph is the lower-level orchestration runtime (durable, stateful graphs) that `create_agent` is actually built on — you drop to LangGraph when you need control the harness doesn't expose.
- **Ignoring lock-in until it hurts** — teams adopt a provider-tied or highly-opinionated framework for a quick demo, assuming they can swap later, then find their orchestration model and prompts are entangled with that framework. Weigh portability up front: a raw loop or a provider-agnostic runtime (LangGraph is usable without LangChain) keeps switching cost low when multi-provider or migration is plausible.
- **Reaching for multi-agent frameworks for single-agent work** — the "multi-agent" label sounds more capable, so beginners adopt CrewAI/AutoGen for one agent with one tool. A single agent with a bounded tool loop is simpler, cheaper, and has less failure surface; only introduce multi-agent orchestration when you genuinely have multiple specialized agents that must coordinate.
- **Hand-rolling what a primitive already solves** — engineers who love control re-implement durable state, retries, and pause/resume by hand, assuming it's "just a loop." Those are precisely the primitives (checkpointers, retry middleware, interrupts) that frameworks harden; DIY versions tend to corrupt state on crash and lack inspection/time-travel — use the primitive once the requirement appears.

---

## Key Definitions

| Term | Definition |
|---|---|
| Agent (core loop) | A model that calls tools in a loop, appending results and re-invoking, until it returns a final answer or hits a step cap. |
| Harness | Everything around the core loop — prompt, tools, and middleware — that shapes the model's behavior (LangChain's term for `create_agent`'s scaffolding). |
| Native tool-calling loop | A hand-written `while` loop over a provider's tool-calling API, with no agent library; you own iteration cap, state, and retries. |
| `create_agent` | LangChain's prebuilt agent constructor: the standard tool-calling loop plus composable middleware (memory, retries, HITL, structured output); built on LangGraph. (Older name: `create_react_agent`.) |
| LangGraph | A low-level orchestration framework/runtime for stateful agents defined as a `StateGraph` of nodes/edges, with checkpointers, interrupts (HITL), and stores; usable without LangChain. |
| Checkpointer | A LangGraph persistence layer that snapshots graph state after each step by `thread_id`, enabling resume-after-failure, time-travel, and HITL. |
| Interrupt (HITL) | A LangGraph mechanism that pauses a run so a human can inspect/modify state before it continues. |
| Handoff | Delegating a task from one agent to another (a first-class primitive in the OpenAI Agents SDK and similar frameworks). |
| Lock-in | The switching cost incurred when orchestration logic/prompts are entangled with a specific framework or provider. |

---

## Summary / Quick Recall

- Every agent is the same loop (model → tool_calls → run tools → repeat); frameworks differ only in the scaffolding they add around it.
- The spectrum runs: native loop (max control, no lock-in) → LangGraph (low-level orchestration, durable/stateful) → `create_agent` (prebuilt harness) → opinionated multi-agent SDKs.
- LangChain ≠ LangGraph: LangChain's `create_agent` is the higher-level loop; LangGraph is the lower-level runtime it's built on — drop down for control, durability, HITL, multi-agent.
- Select on needs: persistence/resume, HITL approval, multi-agent coordination → LangGraph; simple "batteries-included" single agent → `create_agent`; one tool, short-lived, portable → raw loop.
- Other frameworks (OpenAI Agents SDK, CrewAI, AutoGen, LlamaIndex agents) trade wiring for opinionated models and lock-in — pick when their model fits your problem.
- The strong interview answer is criteria-based (control, durability, HITL, multi-agent, observability, lock-in, team familiarity), never "the newest one."

---

## Self-Check Questions

1. What is the "common core" that every agent framework — and a hand-written loop — implements, and why does recognizing it change how you answer "which framework"?

   <details><summary>Answer</summary>

   The common core is: the model receives messages plus tool schemas and returns either a final answer or structured `tool_calls`; a driver runs any tool calls, appends the results, and re-invokes the model until there are no tool calls (or a step cap is hit). Recognizing it reframes framework choice as "how much scaffolding around this loop do I need?" rather than "which product is best" — so you select on requirements (persistence, HITL, multi-agent) instead of popularity. The tempting wrong framing is that each framework does something fundamentally different; they don't — they wrap the same loop with different amounts of durability, HITL, and orchestration machinery.

   </details>

2. You must ship a stateless HTTP endpoint that answers product questions using exactly one lookup tool, single provider, no approvals, no memory across requests. Which option fits and why?

   <details><summary>Answer</summary>

   A **hand-written native tool-calling loop** on the provider's API — it's the least machinery: no extra dependencies, fully portable, trivially auditable, and there are no persistence/HITL/multi-agent needs that would justify a framework. Reaching for LangGraph here is over-engineering (you'd add checkpointers and graph wiring you never use); reaching for a multi-agent framework is worse (an entire coordination model for one tool). The one non-negotiable even in the raw loop is a bounded iteration cap so a misbehaving model can't loop forever.

   </details>

3. **Which TWO** statements about LangChain vs. LangGraph are correct?
   - A. LangGraph is a lower-level orchestration runtime that LangChain's `create_agent` is built on top of.
   - B. LangChain and LangGraph are competing products that cannot be used together.
   - C. You drop from `create_agent` down to LangGraph when you need control the prebuilt harness doesn't expose (custom branches, durable checkpoints, HITL, multi-agent subgraphs).
   - D. LangGraph abstracts your prompts and architecture so you write less than with `create_agent`.
   - E. `create_agent` cannot persist conversation state under any circumstances.

   <details><summary>Answer</summary>

   **A and C.** A is correct: LangChain docs describe `create_agent` as a prebuilt harness built on LangGraph, and LangGraph as the low-level orchestration runtime. C is correct: LangGraph is the escape hatch for fine-grained control (mixing deterministic + agentic steps, checkpointers, interrupts, multi-agent subgraphs). B is the most tempting distractor and is wrong — they're layers of the same stack designed to be used together. D is backwards: LangGraph *deliberately does not* abstract prompts/architecture and is more verbose, not less. E is false: `create_agent` persists state when given a checkpointer and a `thread_id`.

   </details>

4. A team proposes adopting CrewAI for a system that currently has one agent with two tools, arguing it's "more advanced and multi-agent-ready." Evaluate the trade-off and recommend.

   <details><summary>Answer</summary>

   Recommend **against** adopting it now. CrewAI's value is orchestrating *teams* of role-playing agents via Flows; for a single agent with two tools that machinery is unused weight, adds an opinionated model and lock-in, and increases failure surface with no capability gain. The right move is the least machinery that meets today's need — a raw loop or `create_agent` — and to revisit an orchestration framework only when a *concrete* multi-agent requirement (multiple specialized, coordinating agents) actually materializes. "Multi-agent-ready" is not a present requirement; you can migrate when the need is real, and keeping switching cost low is itself an argument against premature adoption.

   </details>

5. Your agent runs for several minutes, must resume cleanly after a worker crash, and must pause for human approval before a destructive action. Compare a hand-written loop against LangGraph for this and justify the choice.

   <details><summary>Answer</summary>

   Choose **LangGraph**. The requirements name exactly the primitives it provides: a **checkpointer** snapshots state by `thread_id` for atomic, resumable durability after a crash, and an **interrupt** pauses the run for human approval before the destructive step. A hand-written loop *can* technically do this, but you'd re-implement durable, atomic checkpointing and pause/resume by hand — the classic anti-pattern that corrupts state on partial writes and lacks inspection/time-travel. The raw loop wins on portability and zero dependencies, but here the durability + HITL requirements dominate, and paying for framework primitives that are already hardened is the correct trade-off; a raw loop would be right only if those requirements were absent.

   </details>

---

## Further Reading

- [LangGraph overview (low-level orchestration: durable execution, HITL, persistence)](https://docs.langchain.com/oss/python/langgraph/overview) — *verified 2026-07-29* — Official positioning of LangGraph vs. LangChain agents and its core benefits (mix deterministic + agentic steps, persistence, HITL).
- [LangChain Agents (`create_agent`, harness, middleware)](https://docs.langchain.com/oss/python/langchain/agents) — *verified 2026-07-29* — Official reference defining the agent core loop and the prebuilt `create_agent` harness with retry/HITL/structured-output middleware.
- [LangGraph Persistence (checkpointers vs. stores)](https://docs.langchain.com/oss/python/langgraph/persistence) — *verified 2026-07-29* — Official guide to checkpointers (short-term, thread-scoped: HITL, time-travel, fault tolerance) and stores (long-term memory).
- [OpenAI Agents SDK (intro: primitives, agent loop, Agents SDK vs. Responses API)](https://openai.github.io/openai-agents-python/) — *verified 2026-07-29* — Official docs describing Agents/handoffs/guardrails/sessions and when to own the loop with the Responses API instead.
- [CrewAI Introduction (Crews and Flows)](https://docs.crewai.com/en/introduction) — *verified 2026-07-29* — Official overview of CrewAI's Crews (role-playing agent teams) and Flows (event-driven stateful workflows).
- [AutoGen (Microsoft) — AgentChat and Core](https://microsoft.github.io/autogen/stable/) — *verified 2026-07-29* — Official landing page for AutoGen's AgentChat (conversational multi-agent) and event-driven Core runtime.
- [LlamaIndex — Building an agent (`FunctionAgent`, `AgentWorkflow`)](https://developers.llamaindex.ai/python/framework/understanding/agent/) — *verified 2026-07-29* — Official tutorial for LlamaIndex's function-calling agent and pointers to its multi-agent workflows.
