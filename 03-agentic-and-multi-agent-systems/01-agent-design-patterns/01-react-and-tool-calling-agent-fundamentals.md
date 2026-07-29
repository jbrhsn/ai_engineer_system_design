# ReAct and Tool-Calling Agent Fundamentals

**Section:** 03 Agentic & Multi-Agent Systems → Agent Design Patterns | **Est. time:** 3 hrs | **Interview relevance:** High — this is the primitive every agentic-systems question builds on; if you can't crisply describe the model→tool→observation loop and how you bound it, you can't defend a multi-agent design.

---

## TL;DR

An LLM by itself only emits text; it becomes an *agent* when you put it in a loop where it can call tools, read the results, and decide what to do next. The **ReAct** pattern (Reason + Act) is the original recipe for this: interleave a **Thought** (reasoning), an **Action** (a tool call), and an **Observation** (the tool's result), repeating until the model produces a final answer. Modern models replace the fragile *text-parsed* ReAct format with **native tool calling**, where the model emits structured `tool_calls` (a tool name + JSON arguments validated against a schema) instead of prose you have to parse. The whole agent is just this loop plus a harness — a prompt, a set of tool definitions, and, critically, a hard cap on iterations so a stuck model can't spin forever. **The one thing to remember: an agent is a model calling tools in a loop until it decides to stop — your job is to define the tools well and to *bound the loop* so termination and cost are guaranteed.**

---

## ELI5 — Explain It Like I'm 5

Imagine you hire a smart assistant to answer a hard question, but you lock them in a room with only a telephone. They can *think out loud*, and they can *make a phone call* to look something up — call the weather line, call the library, call the bank — but they can't do the lookup themselves; they have to ask you to dial and read back what the other end says. So it goes: they think ("I need today's weather"), they tell you which number to dial and what to ask, you dial it and read the answer back, and then they think again with that new fact — over and over until they say "okay, I have the final answer." The common misconception is that the assistant is "doing" the searches; they are not — they only ever produce *requests*, and your code (you, dialing the phone) is what actually runs anything. That separation is the whole trick, and it's also why you must set a rule like "you get at most five phone calls," or a confused assistant will keep dialing forever.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the ReAct Thought/Action/Observation loop and why interleaving reasoning with acting reduces hallucination versus reasoning alone.
- [ ] Contrast native (structured) tool calling with legacy text-parsed ReAct and justify why native calling is the production default.
- [ ] Trace the agent loop (model → tool call → observation → model → final answer) and identify exactly where your code runs versus where the model runs.
- [ ] Write a tool definition/schema that a model can reliably call, and diagnose failures caused by weak names/descriptions.
- [ ] Bound an agent loop for guaranteed termination and handle tool failures without crashing the run, and decide when a *single* tool-calling agent is the right pattern.

---

## Visual Overview

### ReAct Thought / Action / Observation Loop

```
        ┌──────────────────────────────────────────────┐
        ▼                                                │
   ┌─────────┐   ┌──────────┐   ┌──────────────┐         │
   │ THOUGHT │──►│  ACTION  │──►│ OBSERVATION  │─────────┘
   │ (reason)│   │(tool call)│  │ (tool result)│   loop until
   └─────────┘   └──────────┘   └──────────────┘   model emits
        │                                            "Final Answer"
        └──────────────► FINAL ANSWER ──────────────────►
```

### The Agent Loop (native tool calling)

```
user message
     │
     ▼
┌──────────────────────────────┐
│ MODEL (tools bound)          │◄────────────────┐
│  emits AIMessage             │                 │
└──────────────────────────────┘                 │
     │                                            │
 has tool_calls?                                  │
     ├── NO ──► return content as FINAL ANSWER    │
     │                                            │
     └── YES ─► run tool(s) in YOUR code          │
                append ToolMessage(observation) ──┘
                      (bounded by max iterations)
```

### Text-Parsed ReAct vs Native Tool Calling

```
LEGACY (text ReAct)                    NATIVE (structured tool calling)
─────────────────────                  ────────────────────────────────
model prints prose:                    model emits typed object:
  "Thought: I should search              tool_calls=[{
   Action: search[weather SF]              name: "search",
   ..."                                     args: {"query": "weather SF"}
        │                                 }]
        ▼                                        │
  YOU regex-parse it  ← brittle                  ▼
  (breaks on format drift)               validated vs JSON schema ← robust
```

### When Is a Single Tool-Calling Agent the Right Pattern?

```
Does the task need external actions/data at all?
├── No  ──► just call the model (no agent, no loop)
└── Yes ──► Are the steps a fixed, known sequence?
            ├── Yes ──► hard-coded chain/workflow (cheaper, deterministic)
            └── No  ──► Does one coherent role + tool set cover it?
                        ├── Yes ──► SINGLE tool-calling agent  ◄── this chapter
                        └── No  ──► multi-agent (later chapters)
```

---

## Key Concepts

### The ReAct Pattern (Reason + Act)

**What it is.** ReAct is a prompting/agent pattern in which the model alternates between generating a reasoning trace (a **Thought**) and a task action (an **Action**), then consumes the result of that action (an **Observation**) before reasoning again — introduced by Yao et al. (2022) to combine chain-of-thought reasoning with the ability to interact with external sources.

**How it works mechanistically.** Reasoning-only prompting (chain-of-thought) lets a model plan but keeps it trapped with its own (possibly wrong) internal knowledge; acting-only lets it fetch facts but with no plan. ReAct interleaves them: the Thought decomposes the problem and decides *what to look up next*, the Action performs a lookup against an external source (e.g. a Wikipedia API), and the Observation feeds the real result back into the context so the next Thought is grounded in fresh evidence rather than a guess. The paper shows this cuts the hallucination-and-error-propagation failure mode of pure chain-of-thought on tasks like HotpotQA and Fever, because the model can correct course when an observation contradicts its plan.

**Where it appears in real systems.** In the original formulation ReAct is a *text format* — the model literally prints `Thought:`, `Action:`, `Observation:` lines and the harness parses them. Every modern agent framework descends from this idea; LangChain's `create_agent` describes the agent as exactly "a model calling tools in a loop until a given task is complete," which is the ReAct loop with the text format replaced by native tool calls.

### Native (Structured) Tool Calling vs Text-Parsed ReAct

**What it is.** Native tool calling (a.k.a. function calling) is a model capability where, given a list of tool definitions, the model returns a *structured* tool-call object — a tool `name` plus JSON `arguments` — instead of free-form text you must scrape.

**How it works mechanistically.** You pass a `tools` array (each tool is a JSON-schema-typed function) in the API request. On each turn the model decides to either answer in text or emit one or more `function_call` items, each with a `call_id`, a `name`, and JSON-encoded `arguments` that conform to the declared schema. Your code executes the function, then sends the result back as a `function_call_output` referencing that `call_id`; the model reads it and continues. The win over legacy text ReAct is reliability: the arguments are schema-validated (OpenAI's `strict: true` mode guarantees the call adheres to the schema), so you eliminate the brittle regex parsing that broke whenever the model drifted from the `Action:` format.

**Where it appears in real systems.** OpenAI's Responses API exposes this via the `tools` parameter and `function_call`/`function_call_output` items; LangChain wraps it as `model.bind_tools([...])`, which attaches tool schemas to the model so the returned `AIMessage` carries a `.tool_calls` attribute. In LangGraph you inspect `last_message.tool_calls` to route the loop. Text-parsed ReAct still matters for models *without* native tool-calling support, but for any current frontier model, native calling is the default.

### The Agent Loop (model → tool → observation → model)

**What it is.** The agent loop is the control flow that turns a stateless model call into an agent: call the model; if it requested a tool, run the tool and feed the result back; repeat until the model returns a plain answer.

**How it works mechanistically.** State is a growing list of messages. Step 1: the model (with tools bound) is invoked on the message list and returns an `AIMessage`. Step 2: your harness checks whether that message contains `tool_calls`. If **not**, the loop ends and the message content is the final answer. If it **does**, your code executes each requested tool, wraps each result in a `ToolMessage` (the Observation), appends it to the state, and loops back to step 1 — now the model sees the observation and can call another tool or answer. Crucially the model never executes anything; it only *emits requests*, and the harness is what runs code, which is why tool side-effects (writes, payments) live entirely in your control.

**Where it appears in real systems.** In LangGraph you build this explicitly with `StateGraph(MessagesState)`, a model node, a `ToolNode` that executes the tools, and `add_conditional_edges` that routes on whether the last message has `tool_calls` (looping back to the model after the tool, or to `END`). The prebuilt `create_agent(model=..., tools=...)` assembles this same loop for you. LangGraph enforces a `recursion_limit` (default 1000 super-steps) that raises `GraphRecursionError` as a backstop against infinite loops.

### Tool Definitions and Schemas

**What it is.** A tool definition is the contract the model reads to decide *whether*, *when*, and *how* to call a function: a name, a natural-language description, and a typed parameter schema.

**How it works mechanistically.** The model only "sees" the definition, not your implementation — so the definition *is* the interface. The description tells the model the tool's purpose and when to use it; the parameter schema (JSON Schema, often generated from Python type hints or a Pydantic model) constrains what arguments the model may produce. These definitions are injected into the model's context and count as input tokens, which is why you keep the set small and the descriptions tight. Weak names/descriptions are the dominant cause of an agent calling the wrong tool or filling arguments incorrectly — the fix is almost always clearer wording, not a bigger model.

**Where it appears in real systems.** In LangChain the `@tool` decorator turns a typed Python function into a tool; the docstring becomes the description and the type hints define the input schema (type hints are *required*). You can override the name (`@tool("web_search")`) or supply an explicit `args_schema` (a Pydantic model). At the API level this is the `tools=[{"type":"function","name":...,"parameters":{...JSON schema...}}]` array. Best practice (OpenAI): keep fewer than ~20 tools available per turn for accuracy, and make invalid states unrepresentable via enums.

### Termination, Loop-Bounding, and Tool Error Handling

**What it is.** The safety machinery of an agent: a guaranteed stopping condition (so the loop can't run forever) and a strategy for when a tool call throws instead of returning a clean result.

**How it works mechanistically.** Termination has two layers. The *natural* stop is the model returning a message with no `tool_calls`. But a confused model can loop indefinitely (re-calling a tool that never satisfies it), so you add a *hard* bound: a max-iterations / recursion limit that forces the run to end and return a best-effort or fallback answer. For tool failures, the robust pattern is to *catch the exception and convert it into an observation the model can read* (e.g. a `ToolMessage` containing "Tool error: bad input, try again") rather than letting it crash the process — this lets the model self-correct (fix its arguments, try a different tool) on the next turn.

**Where it appears in real systems.** LangGraph's `recursion_limit` config key is the backstop, and its `RemainingSteps` managed value lets you *proactively* route to a fallback node before the limit is hit. For tool errors, LangChain exposes the `wrap_tool_call` middleware (or `ToolRetryMiddleware(max_retries=...)`) to turn exceptions into `ToolMessage`s or retry transient failures. `@tool(return_direct=True)` short-circuits the loop to return a tool's output as the final answer without another model call.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `recursion_limit` (LangGraph) / max iterations | Hard cap on loop super-steps before `GraphRecursionError` | Set to the smallest value that covers your worst *legitimate* multi-tool path (often 6–15 for a single agent), not the 1000 default — an effectively-unbounded loop is the #1 runaway-cost bug. |
| Number of tools bound | How many tools the model chooses among per turn | Keep under ~20; every tool definition is input tokens and more choices raise mis-selection. Split into multiple agents or defer rarely-used tools if you exceed this. |
| `tool_choice` | Whether/which tools the model may call (`auto` / `required` / forced / `none`) | Use `auto` normally; force a specific tool only when the task *must* start with a known call; use `none`/no tools when you want a plain answer. |
| `parallel_tool_calls` | Whether the model may emit multiple tool calls in one turn | Leave on for independent lookups (latency win); turn off when tools have ordering dependencies or shared side-effects. |
| `strict` (function schema) | Enforce that arguments exactly match the JSON schema | Enable for any tool with side-effects or strict argument types; requires `additionalProperties:false` and all fields `required`. |
| Tool error strategy (retry vs. observe) | What happens when a tool raises | Retry (bounded) for *transient* failures (timeouts, rate limits); convert-to-`ToolMessage` for *input* errors so the model can self-correct; never let the exception kill the run. |

### Worked Example: Requirement → Decision

**Given:** You are building an internal assistant for a SaaS support team. It must answer questions like *"What plan is customer #4491 on, and is their latest invoice paid?"* by looking up a customer record and an invoice, then summarizing. Some questions are trivial ("hi", "what timezone are you in?") and need no lookups. The sub-answers sometimes depend on each other (you need the customer's account ID before you can fetch invoices), latency budget is ~4 s, and any write action is out of scope (read-only).

- **Step 1 — Identify the goal:** Answer support questions that require 0, 1, or a few *read* lookups, deciding at runtime whether and what to look up.
- **Step 2 — Define inputs:** The user message; two read tools — `get_customer(customer_id)` and `get_invoices(account_id)` — each a typed function; a chat model with tool calling.
- **Step 3 — Define outputs:** A grounded natural-language answer, or a direct answer with no tool calls for trivial turns; never a fabricated invoice status.
- **Step 4 — Apply constraints:** Read-only (no write tools bound → no dangerous side-effects); latency ≤ 4 s allows a couple of sequential tool round-trips; termination must be guaranteed; the steps are *not* a fixed sequence (some turns need zero lookups, some need two dependent ones).
- **Step 5 — Select the approach:** Use a **single tool-calling agent** (the LangGraph model→ToolNode→conditional-edge loop, or `create_agent`) with the two read tools bound and a `recursion_limit` sized to ~8. *Rationale vs alternatives:* a fixed chain is wrong because the number/order of lookups varies per question; a multi-agent system is overkill because one role and one small tool set cover everything and would only add coordination latency; the single agent's loop naturally handles the "get customer, then get their invoices" dependency because the model reads the first `ToolMessage` before composing the second call.

---

## Implementation

```python
# Scenario: A support assistant must decide at runtime whether to look anything up.
# Trivial turns ("hi") should skip tools; factual turns should call read-only tools,
# and the whole thing must be guaranteed to terminate. We build the explicit agent
# loop so the termination bound is visible and tunable.
# API verified against LangGraph Graph API + create_agent docs (docs.langchain.com).
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode
from langchain.chat_models import init_chat_model
from langchain.tools import tool

model = init_chat_model("openai:gpt-4o-mini", temperature=0)

@tool
def get_customer(customer_id: str) -> str:
    """Look up a customer's plan and account_id by their customer_id."""
    return _crm.fetch_customer(customer_id)          # your read-only CRM call

@tool
def get_invoices(account_id: str) -> str:
    """List invoices and their paid/unpaid status for an account_id."""
    return _billing.fetch_invoices(account_id)       # your read-only billing call

tools = [get_customer, get_invoices]

def call_model(state: MessagesState):
    # Model decides: answer directly, OR emit tool_calls. It never runs tools itself.
    response = model.bind_tools(tools).invoke(state["messages"])
    return {"messages": [response]}

def route(state: MessagesState):
    last = state["messages"][-1]
    return "tools" if getattr(last, "tool_calls", None) else END

builder = StateGraph(MessagesState)
builder.add_node("call_model", call_model)
builder.add_node("tools", ToolNode(tools))           # executes the requested tools
builder.add_edge(START, "call_model")
builder.add_conditional_edges("call_model", route, {"tools": "tools", END: END})
builder.add_edge("tools", "call_model")              # observation flows back to model
graph = builder.compile()

# Hard termination backstop: cap the loop instead of the 1000-step default.
result = graph.invoke(
    {"messages": [{"role": "user", "content": "Is customer 4491's latest invoice paid?"}]},
    config={"recursion_limit": 8},
)
```

```python
# Anti-pattern: an agent loop with NO iteration cap and NO tool-error handling.
# If a tool keeps raising (bad ID) or the model keeps re-calling a tool that never
# satisfies it, this spins forever and the first tool exception crashes the whole run.
def run_agent(messages):                              # BROKEN: unbounded + unguarded
    while True:                                       # no max iterations
        ai = model.bind_tools(tools).invoke(messages)
        messages.append(ai)
        if not ai.tool_calls:
            return ai.content
        for call in ai.tool_calls:
            result = TOOLS[call["name"]].invoke(call["args"])   # raises -> uncaught crash
            messages.append(_to_tool_message(result, call["id"]))

# Correct approach: bound the loop AND turn tool failures into observations the
# model can read and recover from (fix its args / try another tool) next turn.
from langchain.messages import ToolMessage

MAX_ITERS = 8

def run_agent(messages):
    for _ in range(MAX_ITERS):                        # guaranteed termination
        ai = model.bind_tools(tools).invoke(messages)
        messages.append(ai)
        if not ai.tool_calls:
            return ai.content                         # natural stop: no tool calls
        for call in ai.tool_calls:
            try:
                result = TOOLS[call["name"]].invoke(call["args"])
            except Exception as e:                    # don't crash the run
                result = f"Tool error: {e}. Check the arguments and try again."
            messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
    return "Stopped: reached the maximum number of tool-calling steps."  # fallback
# What breaks without this: cost/latency are unbounded and one malformed tool call
# kills the process. The fix makes worst-case cost = MAX_ITERS model calls and lets
# the model self-correct from a failed call, which is what production agents rely on.
```

---

## Common Pitfalls & Misconceptions

- **Thinking the model "runs" the tools** — beginners picture the LLM reaching out and executing the search or the DB query, because the transcript reads that way. The model only ever *emits a request* (`tool_calls`); your harness executes the code and feeds the result back — which is precisely why all side-effects, auth, and safety checks live in your code, not the model.
- **Leaving the loop unbounded** — the happy path always terminates in testing, so a cap feels unnecessary. In production a model that keeps re-calling an unsatisfying tool (or a tool that never returns what it wants) loops until it exhausts your budget; always set a small `recursion_limit`/max-iterations and route to a fallback when hit.
- **Blaming the model for wrong tool calls** — when an agent picks the wrong tool or garbles arguments, people reach for a bigger model. The model reasons over your *tool name, description, and schema*; a vague description ("gets data") is the usual culprit, and a precise description plus enums fixes far more mis-calls than a model swap.
- **Reaching for multi-agent too early** — "agentic" sounds like it should mean many agents, so teams build a swarm for a task one agent handles. A single tool-calling agent with a coherent tool set is simpler, cheaper, and easier to debug; escalate to multi-agent only when distinct roles or context isolation genuinely require it (later chapters).
- **Crashing on the first tool exception** — new implementations let a tool's exception propagate and kill the run, treating tools as reliable. Tools fail (bad input, timeout, rate limit); the robust pattern converts the failure into a `ToolMessage` observation (or a bounded retry) so the agent can recover on the next turn instead of dying.

---

## Key Definitions

| Term | Definition |
|---|---|
| ReAct | An agent pattern that interleaves reasoning traces (Thought) with task actions (Action) and their results (Observation), looping until a final answer. |
| Thought / Action / Observation | The three repeating elements of a ReAct step: reason about what to do, take an action (tool call), read the result. |
| Tool / function calling | A model capability to emit a structured request (tool name + JSON arguments) for an external function instead of plain text. |
| `tool_calls` | The field on a model's response listing the tools it wants invoked, each with a name and arguments; absence of it signals the final answer. |
| Tool definition / schema | The name, description, and JSON-Schema-typed parameters that tell the model whether/when/how to call a tool — the model's only view of the tool. |
| Agent loop | The control flow: call model → if `tool_calls`, run tools and append observations → loop → stop when no tool calls or a bound is hit. |
| `ToolNode` | LangGraph component that executes the tools requested in the last message and appends the results as `ToolMessage`s. |
| `bind_tools` | LangChain method that attaches tool schemas to a model so it can emit `tool_calls`. |
| `recursion_limit` | LangGraph config capping the number of super-steps per run; the hard backstop against infinite agent loops. |
| Single tool-calling agent | One model + one tool set in a loop — the right pattern when a task needs adaptive (non-fixed) external actions under one coherent role. |

---

## Summary / Quick Recall

- An agent = a model calling tools **in a loop** until it decides to stop; the model emits requests, your code runs them.
- ReAct = interleave **Thought → Action → Observation**; grounding each reasoning step in a real observation is what curbs hallucination versus reasoning alone.
- **Native tool calling** (structured, schema-validated `tool_calls`) replaces fragile text-parsed ReAct as the production default; text ReAct survives only for models lacking native support.
- The loop's natural stop is a model message with **no `tool_calls`**; always add a **hard cap** (`recursion_limit`/max iterations) so a stuck model can't run forever.
- A tool's **definition is its interface** — the model sees only the name/description/schema, so weak descriptions (not weak models) cause most mis-calls; keep the tool set small.
- Handle tool failures by turning them into **observations** (or bounded retries) so the agent self-corrects instead of crashing.
- Choose a **single** tool-calling agent when steps are adaptive but one role/tool set covers them; use a fixed chain when steps are known, and multi-agent only when roles truly diverge.

---

## Self-Check Questions

1. In the agent loop, what condition signals that the loop should stop and return a final answer, and who actually executes a tool when the model requests one?

   <details><summary>Answer</summary>

   The loop stops when the model returns a message with **no `tool_calls`** (a plain content answer); that message content is the final answer. When the model *does* request a tool, **your harness/code executes it** (e.g. LangGraph's `ToolNode`), not the model — the model only emits a structured request. The tempting wrong answer is "the model stops when it finishes running the tools," which misattributes execution to the model; the model never runs anything, it only produces `tool_calls`, and the presence/absence of that field is the loop's routing signal.

   </details>

2. You have a model that supports native function calling. Your legacy agent uses a text-ReAct prompt and regex-parses `Action:` lines, and it intermittently fails when the model's formatting drifts. What change do you make and why?

   <details><summary>Answer</summary>

   Switch to **native tool calling**: define the tools as JSON-schema functions (e.g. via `bind_tools`/the `tools` API param) and read the structured `tool_calls` off the response instead of regex-parsing prose. The arguments are then schema-validated (with OpenAI `strict: true` guaranteeing schema adherence), which eliminates the format-drift failures. The tempting wrong answer is "tune the prompt / add more few-shot examples of the `Action:` format" — that only reduces, never removes, parse breakage, whereas structured calling removes the text-parsing surface entirely.

   </details>

3. **Which TWO** of the following are correct about tool definitions and loop safety in a single tool-calling agent?
   - A. A tool's description and parameter schema are what the model reasons over to decide whether and how to call it.
   - B. Setting a `recursion_limit` / max-iterations is only needed for multi-agent systems, not single agents.
   - C. Converting a tool exception into a `ToolMessage` observation lets the model attempt to self-correct on the next turn.
   - D. Binding more tools always improves accuracy because the model has more options.
   - E. The model executes the tool function directly once it decides to call it.

   <details><summary>Answer</summary>

   **A and C.** A is correct because the model only ever sees the definition (name/description/schema), so that text *is* the interface it reasons over. C is correct because turning a raised exception into an observation the model can read enables recovery (fix arguments, try another tool) rather than crashing the run. B is the most tempting distractor and is wrong — a *single* agent can loop forever just as easily (re-calling an unsatisfying tool), so bounding is essential regardless of agent count. D is false: more tools mean more input tokens and more mis-selection; keep the set small (OpenAI suggests <~20). E is false: the model emits a request; your harness executes.

   </details>

4. Your single-agent support bot's cost per conversation is wildly variable and occasionally 30× the median. Traces show some turns issue a long chain of the *same* tool call that never satisfies the model. What is the most likely root cause and the fix?

   <details><summary>Answer</summary>

   The agent loop is effectively **unbounded** (or bounded only by the 1000-step default): for inputs the tool can't satisfy, the model keeps re-calling it, reading an unhelpful observation, and calling again, accumulating model + tool calls on a single turn. The fix is to set a small `recursion_limit`/max-iterations sized to your worst legitimate path and route to a fallback answer when it's hit (LangGraph's `RemainingSteps` lets you do this proactively before `GraphRecursionError`). Swapping models or adding tools would not stop the runaway loop; only bounding it makes worst-case cost deterministic.

   </details>

5. A stakeholder wants to build a five-agent system for a task that is: "answer customer questions that may need one or two read-only lookups, decided per question." Evaluate whether a single tool-calling agent suffices and justify the trade-off.

   <details><summary>Answer</summary>

   A **single tool-calling agent suffices** and is the better choice. The task has one coherent role (answer support questions) and a small read-only tool set; the only variability is *whether/how many* lookups per turn, which the single-agent loop handles natively (it reads each observation before deciding the next call, so dependent lookups work). Multi-agent adds coordination latency, more failure surface, and harder debugging with no benefit here. The tempting wrong answer is "five agents scale better / are more 'agentic'" — but agent count isn't a quality axis; you escalate to multi-agent only when distinct roles or context isolation genuinely require it, not for a task one bounded agent covers.

   </details>

---

## Further Reading

- [ReAct: Synergizing Reasoning and Acting in Language Models (Yao et al., 2022)](https://arxiv.org/abs/2210.03629) — *verified 2026-07-29* — The primary source for the Thought/Action/Observation pattern and the reasoning-plus-acting synergy.
- [LangChain Agents (`create_agent`, the model-in-a-loop harness)](https://docs.langchain.com/oss/python/langchain/agents) — *verified 2026-07-29* — Official definition of an agent as "a model calling tools in a loop," plus `tools=`, `system_prompt=`, streaming, and fault-tolerance middleware.
- [LangChain Tools (`@tool`, schemas, error handling, `return_direct`)](https://docs.langchain.com/oss/python/langchain/tools) — *verified 2026-07-29* — How to define tool schemas from type hints/Pydantic and handle tool errors with `wrap_tool_call`.
- [LangGraph Graph API (StateGraph, ToolNode, conditional edges, recursion limit)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — Reference for building the explicit agent loop and bounding it with `recursion_limit`/`RemainingSteps`.
- [Build a custom RAG agent with LangGraph](https://docs.langchain.com/oss/python/langgraph/agentic-rag) — *verified 2026-07-29* — Worked example of the model→tool→observation loop with `bind_tools` and conditional-edge routing.
- [OpenAI Function Calling guide](https://platform.openai.com/docs/guides/function-calling) — *verified 2026-07-29* — Canonical description of the structured tool-calling flow, JSON-schema function definitions, `tool_choice`, `parallel_tool_calls`, and `strict` mode.
