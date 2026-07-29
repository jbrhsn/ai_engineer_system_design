# Supervisor and Hierarchical Multi-Agent Topologies

**Section:** 03 Agentic & Multi-Agent Systems → Multi-Agent Orchestration with LangGraph | **Est. time:** 3.5 hrs | **Interview relevance:** High — "when do you split into multiple agents, and how do you wire them together?" is the pivotal design question in any agentic-systems interview; you must be able to name the topologies, defend the split, and quantify the token-cost multiplier you're signing up for.

---

## TL;DR

A multi-agent system is several LLM-driven agents (each a model-in-a-loop from chapter 01) coordinating on one task. The dominant coordination shape is the **supervisor** (a.k.a. orchestrator-worker) pattern: one central agent routes work to specialized workers and aggregates their results — either by calling workers **as tools** (subagents) or by handing off control to workers **as graph nodes** via `Command(goto=..., update=...)`. The alternatives are the **network/swarm** (agents hand off peer-to-peer with no central coordinator) and **hierarchical teams** (a supervisor-of-supervisors). The catch nobody mentions up front: every agent is more model calls and more tokens — Anthropic measured multi-agent systems burning ~15× the tokens of a plain chat — so you split into multiple agents only when context isolation, parallelism, or distinct roles genuinely pay for that multiplier, and you reach for a single agent otherwise. **The one thing to remember: a supervisor coordinates specialists (as tools or as nodes) and the win comes from context isolation and parallelism — but token cost scales with agent count, so "more agents" is a cost you justify, never a quality default.**

---

## ELI5 — Explain It Like I'm 5

Imagine a big kitchen for a banquet. One way is a single chef who does everything — chops, sautés, plates, and washes up — walking back and forth; for a simple omelette that's actually the fastest and cheapest. The other way is a head chef (the *supervisor*) who doesn't cook at all but reads the order, hands "make the sauce" to the sauce cook, "grill the fish" to the grill cook, and — crucially — gives each one *only the part of the order they need*, not the whole banquet menu, so nobody gets confused or overwhelmed. Because the cooks work at their own stations at the same time, a huge multi-course order finishes far faster than one chef could manage alone. But here's the misconception to correct: adding cooks is not free and does not automatically make the food better — every extra cook needs their own copy of instructions and their own ingredients (that's the extra "tokens"), so you only hire the team when the meal is genuinely big and parallel; for one omelette, the lone chef wins every time.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Decide when to split one task across multiple agents versus keeping a single tool-calling agent, and justify the decision against the token-cost multiplier.
- [ ] Compare the three core topologies — supervisor (orchestrator-worker), network/swarm, and hierarchical teams — and select one for a given requirement.
- [ ] Distinguish agents-as-tools (subagents) from agents-as-nodes (handoffs) and explain the control-flow and context difference between them.
- [ ] Implement a handoff with `Command(goto=..., update=..., graph=Command.PARENT)` and a supervisor by wrapping subagents as tools, passing correctly scoped context.
- [ ] Diagnose the two dominant multi-agent failure modes: runaway token cost from full-history broadcast and over-decomposition of a task a single agent handles.

---

## Visual Overview

### Supervisor (Orchestrator-Worker) Topology

```
                    ┌───────────────────┐
        user ─────► │    SUPERVISOR     │ ◄─── aggregates results,
                    │  (routes + plans) │      decides next step
                    └───────────────────┘
                       │      │      │
             delegate  │      │      │  delegate
                       ▼      ▼      ▼
                 ┌────────┐┌────────┐┌────────┐
                 │worker A ││worker B││worker C│   (specialized:
                 │(search) ││(math)  ││(SQL)   │    own prompt + tools)
                 └────────┘└────────┘└────────┘
                       │      │      │
                       └──────┴──────┘  results flow back UP to supervisor
                              ▼
                        final answer ──► user
```

### Network / Swarm vs Hierarchical Teams

```
NETWORK / SWARM (peer-to-peer)          HIERARCHICAL (supervisor-of-supervisors)
──────────────────────────────         ─────────────────────────────────────────
   ┌─────────┐                                     ┌──────────────┐
   │ agent A │◄──────┐                              │ TOP SUPERVISOR│
   └─────────┘       │                              └──────────────┘
      │  ▲           │                                 │          │
 handoff │ handoff   │                          delegate│          │delegate
      ▼  │           │                                 ▼          ▼
   ┌─────────┐   ┌─────────┐                   ┌────────────┐ ┌────────────┐
   │ agent B │◄─►│ agent C │                   │ research    │ │ writing    │
   └─────────┘   └─────────┘                   │ SUPERVISOR  │ │ SUPERVISOR │
   any agent can hand off to any other         └────────────┘ └────────────┘
   (no central coordinator)                       │      │       │      │
                                                   ▼      ▼       ▼      ▼
                                                 [search][math] [writer][editor]
```

### Agents-as-Tools vs Agents-as-Nodes

```
AGENTS-AS-TOOLS (subagents)             AGENTS-AS-NODES (handoffs)
─────────────────────────────          ─────────────────────────────
supervisor calls worker like a tool,    control TRANSFERS to the next agent;
worker returns a value, control         it does NOT return to the caller
RETURNS to the supervisor               unless it hands back explicitly

  supervisor                              agent_A ──Command(goto="agent_B")──►
    └─ call_worker("do X") ──►               agent_B  (now in control,
         worker runs, returns "Y"            talks to user directly)
    └─ supervisor continues with "Y"      graph=Command.PARENT routes in
                                          the parent graph
  → centralized control, easy aggregate  → decentralized, direct user turns
```

### Decision Tree: Single Agent vs Supervisor vs Network

```
Does the task genuinely need >1 role / big isolated contexts / parallel work?
├── No  ──► SINGLE tool-calling agent  (cheapest, easiest to debug — chapter 01)
└── Yes ──► Do sub-tasks run independently & in parallel, aggregated centrally?
            ├── Yes ──► SUPERVISOR (orchestrator-worker); subagents-as-tools
            │           └── too many workers / distinct teams? ──► HIERARCHICAL
            └── No  ──► Do agents need to converse with the USER across handoffs
                        with no central coordinator?
                        ├── Yes ──► NETWORK / SWARM (peer handoffs)
                        └── No  ──► SUPERVISOR with sequential (multi-hop) handoffs
```

---

## Key Concepts

### Single-Agent vs Multi-Agent (When to Split)

**What it is.** A choice between one model-in-a-loop with all tools bound (chapter 01) and several coordinating agents. LangChain's own guidance is explicit: "not every complex task requires this approach — a single agent with the right (sometimes dynamic) tools and prompt can often achieve similar results."

**How it works mechanistically.** Developers reach for multi-agent to get one of three things: **context management** (a worker sees only its slice, so the main context doesn't bloat), **distributed development** (different teams own different agents behind clean boundaries), and **parallelization** (spawn workers that run concurrently). The trigger conditions are: a single agent has *too many tools* and mis-selects; tasks need *specialized knowledge with long, domain-specific context*; or you must *enforce sequential constraints* that unlock capabilities only after a precondition is met. Absent one of those, the extra agents add only coordination latency and token cost.

**Where it appears in real systems.** The split shows up as a topology choice in a LangGraph `StateGraph`: one `create_agent(...)` node versus several agent nodes wired with conditional edges/`Command`. Anthropic's Research feature is the canonical multi-agent case — open-ended research where "you can't hardcode a fixed path," so a lead agent spawns parallel subagents; their eval showed the multi-agent system beat single-agent Claude Opus by 90.2% on breadth-first research, precisely the workload that justifies the split.

### The Supervisor (Orchestrator-Worker) Pattern

**What it is.** A central supervisor agent coordinates specialized worker/sub-agents: it decides which worker to invoke, what input to give it, and how to combine the results — while the workers do the domain work.

**How it works mechanistically.** The supervisor is a full agent (it maintains conversation context and decides across turns), unlike a router which is a single classification step. On each turn the supervisor's LLM either answers or emits a call to a worker; the worker runs in its own (often clean) context, returns a result, and control comes back to the supervisor, which aggregates and decides the next move. This is Anthropic's **orchestrator-worker** architecture: "a lead agent coordinates the process while delegating to specialized subagents that operate in parallel." Anthropic dispatches subagents synchronously by default — the lead waits for each batch — which simplifies coordination but bottlenecks information flow.

**Where it appears in real systems.** In LangChain the recommended implementation is **subagents-as-tools**: build each worker with `create_agent(...)`, wrap it in an `@tool` whose function calls `subagent.invoke({"messages": [...]})` and returns `result["messages"][-1].content`, then give those tools to a supervisor `create_agent`. Because multiple tool calls in one supervisor turn are executed in parallel by the runtime, this also gives you fan-out. (The older `langgraph-supervisor` package's `create_supervisor([...], model=...)` is now in maintenance mode; LangChain recommends the tool-calling supervisor pattern directly.)

### Handoffs via `Command(goto=..., update=...)` (Agents-as-Nodes)

**What it is.** A handoff transfers *control* from one agent to another, rather than calling a worker and getting a value back. A tool returns a `Command` that both updates shared state and routes to the next agent node.

**How it works mechanistically.** `Command` is a LangGraph primitive with four parameters; for handoffs you use `goto` (which node runs next), `update` (state to apply, including messages), and `graph=Command.PARENT` (to route in the parent graph when each agent is a subgraph). Critically, an LLM that calls a tool expects a response, so a handoff tool must append a `ToolMessage` with the matching `tool_call_id` — otherwise the conversation history is malformed and the receiving agent errors. You also decide *what messages travel*: the docs recommend passing only the triggering `AIMessage` + the acknowledging `ToolMessage`, not the whole worker history, to keep the parent context focused.

**Where it appears in real systems.** A handoff tool looks like:
`return Command(goto="sales_agent", update={"active_agent": "sales_agent", "messages": [last_ai_message, transfer_message]}, graph=Command.PARENT)`. This is the mechanism behind both the network/swarm pattern (any agent hands off to any other) and multi-agent handoffs where a worker must talk to the user directly — something a subagent-as-tool cannot do, since subagents return to the supervisor, not the user.

### Network / Swarm and Hierarchical Topologies

**What it is.** Two topologies beyond the flat supervisor. In a **network/swarm**, agents hand off to each other peer-to-peer with no central coordinator. In a **hierarchical** system, a top-level supervisor manages *other supervisors*, each running its own team of workers.

**How it works mechanistically.** The swarm is built from handoff tools: each agent has tools like `transfer_to_X` that `Command(goto="X", ...)` to a peer, and an `active_agent` state field tracks who is in control across turns; routing is fully decentralized. The hierarchy is recursion of the supervisor pattern: a "research team" supervisor and a "writing team" supervisor are each themselves compiled agents, and a top-level supervisor treats each *team* as a worker — Anthropic's system nests a LeadResearcher over parallel subagents plus a separate CitationAgent. Hierarchy is how you scale past the point where one supervisor has too many workers to route among reliably.

**Where it appears in real systems.** In LangChain, hierarchy composes by wrapping a whole compiled supervisor as a tool for a higher supervisor (subagents can invoke subagents). The swarm shows up wherever agents need direct user interaction and there's no natural single coordinator; the trade-off is that decentralized routing is harder to reason about and debug than a single supervisor's decisions.

### Message / State Scoping (Full History vs Scoped Context)

**What it is.** The decision of how much conversation/state each worker receives — the entire message history, or a narrow, task-scoped slice.

**How it works mechanistically.** LangChain calls this "context engineering," and it's the center of multi-agent design: "the quality of your system depends on ensuring each agent has access to the right data for its task." With subagents-as-tools, the default is a *clean* context — the supervisor passes just a query string, giving strong context isolation (each worker's tokens don't accumulate in the main thread). With handoffs you must *explicitly* choose; passing the full subagent history bloats tokens and can confuse the receiver with irrelevant internal reasoning. Anthropic's fix at scale: subagents write large outputs to a filesystem and pass only lightweight references back, avoiding the "game of telephone" of copying everything through the coordinator.

**Where it appears in real systems.** Scoping is a concrete API choice. Full history: `langgraph-supervisor`'s `output_mode="full_history"` vs `"last_message"`. Scoped input to a subagent: inside the tool, transform `runtime.state["messages"]` into a minimal `subagent_input` before `subagent.invoke(...)`. Scoped handoff: `update={"messages": [last_ai_message, transfer_message]}` passes only the pair, not the transcript.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Number of workers / sub-agents | How many specialists the supervisor routes among | Match to *distinct roles*, not task size; keep a single supervisor's fan-out small (Anthropic embeds scaling rules: 1 agent for simple fact-finding, 2–4 for comparisons, 10+ only for genuinely complex research). Beyond ~10 workers, go **hierarchical** rather than widen one supervisor. |
| Context passed to workers (full vs scoped) | Whether a worker sees full history or a task-scoped slice | Default to **scoped** (query-only / clean context) for isolation and token savings; pass full/extra context only when the worker must resolve references like "same time as before" that require prior turns. |
| Parallel fan-out width | How many workers run concurrently in one super-step | Fan out only *independent* sub-tasks; Anthropic runs 3–5 subagents in parallel and cut research time up to 90%. Serialize when a worker's output feeds the next (dependency). |
| Coordination style (tools vs handoffs) | Whether control returns to a supervisor or transfers to a peer | Use **subagents-as-tools** when you want centralized aggregation and workers needn't talk to the user; use **handoffs** when an agent must converse with the user directly or you want decentralized (swarm) routing. |
| `recursion_limit` (per graph) | Hard cap on super-steps before `GraphRecursionError` | Set higher than a single agent (multi-agent has more super-steps) but still bounded; multi-agent multiplies runaway-cost risk, so pair it with per-worker iteration caps. |
| Handoff message payload | Which messages the `Command.update` carries across a handoff | Pass only the triggering `AIMessage` + acknowledging `ToolMessage` by default; summarize rather than forward raw worker history to control cost and avoid confusing the receiver. |

### Worked Example: Requirement → Decision

**Given:** You're building a research assistant for an investment team. A typical request is *"Identify every board member of the top-10 IT companies in the S&P 500 and flag any who also sit on a competitor's board."* Answers require pursuing many independent leads (one company at a time), each lead can blow past a single context window, latency matters (analysts won't wait 20 minutes), and the task value is high (this informs real decisions). There's no fixed number of lookups and no fixed order.

- **Step 1 — Identify the goal:** Answer open-ended, breadth-first research questions that decompose into many *independent* sub-investigations, then synthesize one report.
- **Step 2 — Define inputs:** The user query; a set of search/browse tools; a lead model (strong) and worker models (cheaper/faster); scoped task descriptions per worker.
- **Step 3 — Define outputs:** A synthesized answer aggregated from all workers, with each worker returning a concise findings summary (not its raw transcript) to the coordinator.
- **Step 4 — Apply constraints:** High parallelism available (companies are independent); per-company context can exceed one window → need context isolation; latency budget favors concurrent workers; the task's value justifies a high token spend, but you must still bound it.
- **Step 5 — Select the approach:** Use a **supervisor / orchestrator-worker** topology with **subagents-as-tools**, fanning out one worker per company in parallel and passing each a *scoped* task (just that company), each returning a summary the lead aggregates. *Rationale vs alternatives:* a **single agent** would search sequentially and drown in context (Anthropic's single-agent baseline failed this exact task); a **network/swarm** adds no value because there's no need for peer handoffs or direct user turns and decentralized routing would be harder to control; **hierarchy** is unnecessary at ~10 workers under one supervisor — you'd only add a layer if you also had, say, a separate writing team. You explicitly accept the ~15× token multiplier because the task is high-value and heavily parallel — the exact profile Anthropic says multi-agent is *for*.

---

## Implementation

```python
# Scenario: A personal-assistant supervisor must handle requests spanning two
# distinct domains (calendar and email) that each have their own tools and prompt.
# We build the RECOMMENDED supervisor pattern: each worker is its own agent,
# wrapped as a tool, so the supervisor routes at the domain level and the
# runtime runs independent tool calls in parallel. Context stays SCOPED — each
# worker gets only the request string, not the whole conversation.
# API verified against docs.langchain.com multi-agent/subagents + supervisor tutorial.
from langchain.agents import create_agent
from langchain.tools import tool

# 1. Specialized workers — each a full agent with its own narrow tool set + prompt.
calendar_agent = create_agent(model, tools=[create_calendar_event, get_available_time_slots],
                              system_prompt="You are a calendar scheduling assistant. ...")
email_agent    = create_agent(model, tools=[send_email],
                              system_prompt="You are an email assistant. ...")

# 2. Wrap each worker AS A TOOL. It runs in a clean context and returns only its
#    final message — the supervisor never sees the worker's intermediate reasoning.
@tool
def schedule_event(request: str) -> str:
    """Schedule calendar events from natural language (create/modify/check appointments)."""
    result = calendar_agent.invoke({"messages": [{"role": "user", "content": request}]})
    return result["messages"][-1].content          # scoped: only the summary flows back

@tool
def manage_email(request: str) -> str:
    """Send emails from natural language (reminders, notifications)."""
    result = email_agent.invoke({"messages": [{"role": "user", "content": request}]})
    return result["messages"][-1].content

# 3. Supervisor sees only high-level tools and aggregates. Independent calls in one
#    turn (schedule_event + manage_email) are executed in parallel by the runtime.
supervisor = create_agent(
    model,
    tools=[schedule_event, manage_email],
    system_prompt=("You are a personal assistant. Break requests into the right tool "
                   "calls and coordinate results. Use multiple tools in parallel when "
                   "the sub-tasks are independent."),
)
result = supervisor.invoke({"messages": [{"role": "user",
    "content": "Book the design review next Tuesday 2pm AND email the team a reminder."}]})
```

```python
# Scenario: Two agents (sales, support) must converse with the USER directly and
# transfer the live conversation between themselves — a subagent-as-tool can't do
# this (it returns to the supervisor, not the user), so we use HANDOFFS: each agent
# is a graph NODE, and a handoff tool transfers control with Command(goto=...).
# API verified against docs.langchain.com multi-agent/handoffs.
from langchain.messages import AIMessage, ToolMessage
from langchain.tools import tool, ToolRuntime
from langgraph.types import Command

@tool
def transfer_to_sales(runtime: ToolRuntime) -> Command:
    """Transfer the conversation to the sales agent."""
    # The AIMessage that triggered this handoff must be paired with a ToolMessage,
    # or the receiving agent sees a malformed (unanswered tool call) history.
    last_ai = next(m for m in reversed(runtime.state["messages"]) if isinstance(m, AIMessage))
    ack = ToolMessage(content="Transferred to sales agent", tool_call_id=runtime.tool_call_id)
    return Command(
        goto="sales_agent",                         # control TRANSFERS (no return)
        update={"active_agent": "sales_agent",
                "messages": [last_ai, ack]},        # SCOPED: pass only the pair, not history
        graph=Command.PARENT,                       # route in the parent graph
    )
```

```python
# Anti-pattern: using a multi-agent supervisor for a simple LINEAR task a single
# agent handles at a fraction of the cost — AND broadcasting full history to every
# worker, blowing the context/token budget. Both mistakes together.
supervisor = create_agent(model, tools=[greet_agent_tool, format_agent_tool])  # WRONG
@tool
def greet_agent_tool(request: str, runtime: ToolRuntime) -> str:
    # WRONG: dumps the ENTIRE conversation into the worker every call.
    full_history = runtime.state["messages"]                     # unbounded context
    result = greet_agent.invoke({"messages": full_history})      # 15× tokens for a greeting
    return result["messages"][-1].content
# What breaks: (1) A "greet then format a name" task has one role and 2 trivial steps —
# a single agent (or no agent) does it in ~1 call; the supervisor adds an extra
# routing call PER worker (subagents cost +1 call each) plus coordination latency
# for zero benefit. (2) Passing full_history to every worker means each worker
# reprocesses the whole transcript — Anthropic measured multi-agent at ~15× the
# tokens of chat, and full-history broadcast is how you hit the worst end of that.

# Correct approach: single agent for the simple task; if you DO need workers, pass
# SCOPED context (just the sub-request), which is the default subagent behavior.
agent = create_agent(model, tools=[greet, format_name],
                     system_prompt="Greet the user and format their name.")   # one role, one loop
result = agent.invoke({"messages": [{"role": "user", "content": "Hi, I'm ada lovelace"}]})
# If genuinely multi-domain, keep the tool-wrapped worker scoped:
@tool
def greet_agent_tool_fixed(request: str) -> str:
    """Greet the user."""
    result = greet_agent.invoke({"messages": [{"role": "user", "content": request}]})  # scoped
    return result["messages"][-1].content
```

---

## Common Pitfalls & Misconceptions

- **"More agents = better."** — The word "agentic" and demos of swarms make teams equate agent count with capability. Agent count is a *cost axis*, not a quality axis: Anthropic found multi-agent burns ~15× the tokens of chat and works "mainly because they help spend enough tokens"; you split only when context isolation, parallelism, or distinct roles justify that spend, and a single agent is the default.
- **Confusing agents-as-tools with agents-as-nodes.** — Both "call another agent," so beginners treat them as interchangeable. They differ in control flow: a subagent-as-tool *returns a value* to the supervisor (centralized, can't talk to the user); a handoff *transfers control* via `Command(goto=...)` (the receiver runs next and can converse with the user). Pick tools for centralized aggregation, handoffs for direct user turns / peer swarms.
- **Broadcasting full conversation history to every worker.** — It feels "safer" to give each worker everything so it has all the context. In reality each worker then reprocesses the whole transcript (multiplying tokens) and can be *confused* by another agent's irrelevant internal reasoning; the correct model is scoped context — pass the task-relevant slice (default subagent behavior), or summarize rather than forward raw history across a handoff.
- **Forgetting the `ToolMessage` on a handoff.** — People `Command(goto=...)` and move the `AIMessage` but omit the acknowledging `ToolMessage`. An LLM that emitted a tool call expects a response; without the matching `tool_call_id` the receiving agent sees a malformed history and errors. Always pass the `AIMessage` + `ToolMessage` pair.
- **Widening one supervisor instead of going hierarchical.** — When workers multiply, the instinct is to add more tools to the single supervisor. Past ~10 workers the supervisor mis-routes (too many similar choices, à la the single-agent tool-overload problem); the correct model is to group workers into *teams* under sub-supervisors and let a top supervisor route among teams.

---

## Key Definitions

| Term | Definition |
|---|---|
| Multi-agent system | Several LLM-driven agents (each a model-in-a-loop) coordinating on one task for context isolation, distributed development, or parallelism. |
| Supervisor / orchestrator-worker | A topology where one central agent routes work to specialized workers and aggregates their results across turns. |
| Router (vs supervisor) | A single classification step that dispatches to an agent once, without maintaining ongoing conversation state — narrower than a supervisor. |
| Subagent (agents-as-tools) | A worker agent wrapped as a tool; the supervisor calls it, it runs in a clean context, and returns a value (control returns to the supervisor). |
| Handoff (agents-as-nodes) | A control transfer between agents via a tool returning `Command(goto=..., update=...)`; the receiver runs next rather than returning to the caller. |
| Network / swarm | A decentralized topology where agents hand off peer-to-peer with no central coordinator, tracked by an `active_agent` state field. |
| Hierarchical teams | A supervisor-of-supervisors: a top agent routes among team supervisors, each managing its own workers. |
| `Command(goto, update, graph)` | LangGraph primitive combining routing (`goto`), state update (`update`), and parent-graph targeting (`graph=Command.PARENT`). |
| Context scoping | Deciding how much history/state each worker receives — full history vs a task-scoped slice; the core lever for multi-agent cost and quality. |
| Fan-out (parallelism) | The supervisor spawning multiple independent workers to run concurrently in one super-step. |
| Token-cost multiplier | The empirical fact that multi-agent systems consume far more tokens than a single agent (~15× vs chat in Anthropic's data). |

---

## Summary / Quick Recall

- Default to a **single agent**; split into multiple only for context isolation, distributed development, or parallelism — a single agent with the right tools "can often achieve similar results."
- The **supervisor / orchestrator-worker** pattern is the workhorse: a central agent routes to specialized workers and aggregates; it's a full agent, not a one-shot router.
- **Agents-as-tools** (subagents) *return a value* to the supervisor (centralized, clean context); **agents-as-nodes** (handoffs) *transfer control* via `Command(goto=..., update=...)` (decentralized, can talk to the user).
- **Network/swarm** = peer handoffs, no coordinator; **hierarchical** = supervisor-of-supervisors — go hierarchical instead of widening one supervisor past ~10 workers.
- **Scope the context**: default to query-only/clean worker contexts; broadcasting full history multiplies tokens and confuses workers. Always pair the `AIMessage` + `ToolMessage` on a handoff.
- **Fan out** independent sub-tasks in parallel (Anthropic: 3–5 subagents, ~90% latency cut); serialize when outputs are dependent.
- Multi-agent costs **~15× the tokens of chat** — reserve it for high-value, parallel, context-heavy tasks; it's a cost you justify, never a default.

---

## Self-Check Questions

1. What distinguishes the supervisor pattern from a router, and what are the two ways a supervisor can invoke its workers?

   <details><summary>Answer</summary>

   A **supervisor** is a full agent that maintains conversation context and decides *across multiple turns* which workers to call and how to combine results; a **router** is a single classification step that dispatches once without maintaining ongoing state. A supervisor invokes workers either as **tools** (subagents-as-tools — the worker returns a value and control returns to the supervisor) or as **nodes via handoffs** (`Command(goto=...)` — control transfers to the worker). The tempting wrong answer is "a supervisor and a router are the same thing" — they aren't: the router doesn't carry conversation state or orchestrate multi-step aggregation, which is exactly what makes the supervisor a coordinator rather than a dispatcher.

   </details>

2. You're wrapping a worker agent as a tool for a supervisor. The worker keeps returning "I completed the analysis" without the actual findings, so the supervisor can't aggregate. What's the fix, and what context should the tool pass into the worker?

   <details><summary>Answer</summary>

   Fix the **subagent output**: prompt the worker to include all results in its *final message*, because the supervisor only sees `result["messages"][-1].content` — a common failure mode is a worker doing the tool calls but not putting the results in its final response. For **input**, pass **scoped context** (typically just the task/request string), which is the default subagent behavior and gives context isolation. The tempting wrong answer is "pass the full conversation history so the worker has everything" — that multiplies tokens and can confuse the worker with irrelevant reasoning; the output problem is about the worker's *final message*, not about giving it more input.

   </details>

3. **Which TWO** of the following are true about handoffs implemented with `Command(goto=..., update=...)`?
   - A. A handoff transfers control to the target agent, which runs next instead of returning a value to the caller.
   - B. The handoff tool should include a `ToolMessage` with the matching `tool_call_id` in its `update`.
   - C. Passing the full worker message history in every handoff is the recommended default for correctness.
   - D. `graph=Command.PARENT` is only used for cosmetic graph rendering and has no effect on routing.
   - E. Handoffs are the only way to make a supervisor aggregate results from parallel subagents.

   <details><summary>Answer</summary>

   **A and B.** A is correct — a handoff *transfers* control (the receiver runs next), unlike a subagent-as-tool which returns a value to the supervisor. B is correct — the LLM that emitted the tool call expects a response, so the `update` must carry a `ToolMessage` with the matching `tool_call_id` or the receiving agent sees a malformed history. C is the tempting distractor and is wrong — the docs recommend passing only the triggering `AIMessage` + `ToolMessage` pair (or a summary), because full-history broadcast bloats tokens and confuses the receiver. D is false: `graph=Command.PARENT` actually routes to a node in the parent graph. E is false: parallel aggregation is the *subagents-as-tools* pattern (multiple tool calls run in parallel and results return to the supervisor), not handoffs.

   </details>

4. A stakeholder proposes a 6-agent swarm for a task: "answer support tickets that each need one or two lookups, one ticket at a time, decided per ticket." Evaluate the proposal and recommend a topology with justification.

   <details><summary>Answer</summary>

   Reject the swarm; recommend a **single tool-calling agent** (or at most a simple supervisor if there are truly distinct domains). The task has one coherent role, low branching (1–2 lookups), and no parallelism or peer-to-peer user handoffs — none of the three triggers for splitting (context isolation, distributed development, parallelism) is present. A network/swarm adds decentralized routing that's hard to debug and multiplies token cost (~15× chat) for zero benefit. The tempting wrong answer is "a swarm scales better / is more robust" — but agent count is a cost axis, not a quality axis; swarms are for peer handoffs and direct user turns across agents, which this workload doesn't need.

   </details>

5. You have a valid multi-agent research system (breadth-first, parallelizable, high-value) whose cost is 15× your single-agent baseline and whose latency is dominated by one slow worker per request. Analyze the two levers you'd pull and their trade-offs.

   <details><summary>Answer</summary>

   **Lever 1 — context scoping:** ensure each worker gets only its task-scoped slice (not full history) and returns a *summary* (or writes large outputs to a filesystem and passes a reference back, per Anthropic's "game of telephone" fix). This attacks the token multiplier directly — full-history broadcast is what pushes cost toward the worst end of the ~15× range — with the trade-off that over-aggressive scoping can starve a worker of context it needs. **Lever 2 — parallel fan-out / model tiering:** run independent workers concurrently (Anthropic cut research time up to 90% with 3–5 parallel subagents) and use a cheaper/faster model for workers while reserving the strong model for the lead; the trade-off is that only *independent* sub-tasks can be parallelized (dependent ones must serialize) and cheaper worker models may reduce quality. The tempting wrong answer is "switch back to a single agent to save cost" — but the task is exactly the breadth-first, high-value profile where single-agent *fails* (Anthropic's single-agent baseline couldn't complete it), so the fix is to optimize the multi-agent system, not abandon it.

   </details>

---

## Further Reading

- [Multi-agent (LangChain — patterns, choosing, performance comparison)](https://docs.langchain.com/oss/python/langchain/multi-agent) — *verified 2026-07-29* — Overview of the subagents/handoffs/router/skills patterns, when to use each, and per-pattern model-call and token comparisons.
- [Subagents (LangChain — supervisor via tools, context engineering)](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents) — *verified 2026-07-29* — The recommended supervisor pattern (workers wrapped as tools), sync vs async, subagent inputs/outputs, and scoped-context design decisions.
- [Handoffs (LangChain — Command, Command.PARENT, context passing)](https://docs.langchain.com/oss/python/langchain/multi-agent/handoffs) — *verified 2026-07-29* — Agents-as-nodes handoffs, the `AIMessage`+`ToolMessage` pairing rule, and why to pass only the handoff pair rather than full history.
- [Build a personal assistant with subagents (LangChain supervisor tutorial)](https://docs.langchain.com/oss/python/langchain/supervisor) — *verified 2026-07-29* — End-to-end supervisor build wrapping calendar/email agents as tools, parallel dispatch, and controlling information flow to/from workers.
- [Graph API — Command, Send, recursion limit (LangGraph)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — Reference for `Command(update/goto/graph/resume)`, the `Send` map-reduce/fan-out API, and `recursion_limit`/`RemainingSteps` bounding.
- [How we built our multi-agent research system (Anthropic)](https://www.anthropic.com/engineering/multi-agent-research-system) — *verified 2026-07-29* — Orchestrator-worker lessons, the ~15× token multiplier, effort-scaling rules, parallel subagents, and when multi-agent is (and isn't) worth it.
- [langgraph-supervisor (prebuilt create_supervisor, output_mode, hierarchies)](https://github.com/langchain-ai/langgraph-supervisor-py) — *verified 2026-07-29* — The prebuilt supervisor library (now maintenance-mode) showing `create_supervisor`, `full_history` vs `last_message`, and multi-level hierarchies.
