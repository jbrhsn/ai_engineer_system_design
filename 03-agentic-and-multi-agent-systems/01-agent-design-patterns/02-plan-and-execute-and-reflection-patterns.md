# Plan-and-Execute and Reflection Patterns

**Section:** 03 Agentic & Multi-Agent Systems → Agent Design Patterns | **Est. time:** 3 hrs | **Interview relevance:** High — "your ReAct agent is slow, expensive, and drifts on 12-step tasks — what do you change?" is a classic senior follow-up, and separating *planning* from *self-correction* is exactly what interviewers probe.

---

## TL;DR

A pure ReAct loop (chapter 01) interleaves think→act→observe one step at a time, re-invoking a big reasoning model on every turn — great for short, unpredictable tasks, but on long-horizon work it is slow, expensive, and prone to losing the thread. **Plan-and-Execute** fixes this by having a planner write the whole multi-step plan up front, an executor (often a cheaper model or plain code) run each step, and an optional replanner revise the plan only when reality diverges — fewer expensive reasoning calls, an inspectable plan, and lower latency. **Reflection / self-critique** patterns (generate → critique → revise, including Reflexion-style verbal feedback and the generator/critic "evaluator–optimizer" topology) trade *more* calls for *higher quality* by letting the model grade and improve its own output. Tree-of-Thoughts generalises this to searching multiple reasoning branches. Every one of these is extra LLM calls you are paying for, and every loop can run forever. **The one thing to remember: Plan-and-Execute buys efficiency and inspectability on long tasks; Reflection buys accuracy at the cost of more calls — and any replan or reflection loop MUST be bounded or it becomes a runaway-cost, non-terminating bug.**

---

## ELI5 — Explain It Like I'm 5

Imagine you are cooking a big holiday dinner. One way is to stand at the stove and, after every single action, stop and think hard from scratch about what to do next — "okay, I chopped the onion, now what?" — that is a ReAct agent, and for a huge meal it is exhausting and you lose track. A smarter cook writes the whole recipe plan on a whiteboard first ("1. prep veg, 2. make sauce, 3. roast, 4. plate"), then just works down the list, only rewriting the board if something goes wrong like the oven breaking — that is Plan-and-Execute. Reflection is when, before serving, the cook tastes the sauce, decides "too salty," and fixes it — generate, then critique, then revise. The common mistake is thinking the fancy plan-then-taste-and-fix cook is always better: for making a single piece of toast, writing a whiteboard plan and tasting it three times just wastes time and gas for the same toast.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Explain the Plan-and-Execute topology (planner → executor → optional replanner) and articulate *why* it reduces reasoning-model calls, latency, and drift versus a pure ReAct loop on long-horizon tasks.
- [ ] Design a Reflection loop (generate → critique → revise) and distinguish the generator/critic (evaluator–optimizer) topology from Reflexion-style verbal-feedback memory.
- [ ] Contrast Tree-of-Thoughts against linear reflection and say when its branching search is worth the cost.
- [ ] Compose these patterns with the ReAct/tool-calling loop from chapter 01 and place them correctly in a LangGraph `StateGraph`.
- [ ] Bound replan and reflection loops with explicit iteration caps and justify the accuracy-vs-latency-vs-cost trade-off for a given task.

---

## Visual Overview

### Plan → Execute → Replan Loop

```
                          ┌──────────────────────────────────────────┐
                          ▼                                            │ (replan, capped)
user goal ──► PLANNER ──► [ step 1, step 2, step 3, ... ]              │
             (big model)         │                                     │
                                 ▼                                     │
                            EXECUTOR runs step i ──► observation ──► REPLANNER
                           (cheap model / code)                        │
                                                                       ▼
                                                    plan empty? ──► YES ──► final answer
                                                                       │
                                                                       └► NO ──► loop with revised plan
```

### Reflection Loop (Generate → Critique → Revise)

```
task ──► GENERATOR ──► draft answer ──┐
              ▲                        ▼
              │                    CRITIC / EVALUATOR grades draft
              │                        │
              │            ┌───────────┴────────────┐
        revised draft   "good enough"          "needs work" + written feedback
        (+ feedback)         │                        │
              └──────────────┘◄───────────────────────┘  (loop, bounded by max_reflections)
                             ▼
                        accept ──► return
```

### ReAct vs Plan-and-Execute vs Reflection (decision tree)

```
Is the task short / unpredictable (few steps, path not knowable up front)?
├── Yes ──► ReAct / tool-calling loop (chapter 01) — flexible, low overhead
└── No  ──► Is it long-horizon with mostly-knowable sub-steps?
            ├── Yes ──► Plan-and-Execute — plan once, execute cheaply, replan only on divergence
            └── Is the FAILURE mode "answer is low quality / wrong, not incomplete"?
                        ├── Yes ──► add a Reflection (generate→critique→revise) loop, bounded
                        └── Need to explore several distinct solution paths & compare?
                                    └──► Tree-of-Thoughts (branch + self-evaluate + prune) — highest cost
```

---

## Key Concepts

### Plan-and-Execute

**What it is.** An agent topology that separates *planning* from *doing*: a planner LLM produces an explicit, ordered multi-step plan for the whole task before any step runs, an executor carries out each step, and an optional replanner revises the remaining plan after observing results.

**How it works mechanistically.** The planner is called *once* on the full goal and emits a structured list of steps (e.g. a Pydantic `Plan(steps: list[str])`). The executor then processes steps one at a time — each step can itself be a small ReAct agent or a deterministic tool call — appending each result to state. After a step (or the whole plan), a replanner node inspects progress and either (a) returns a revised list of remaining steps or (b) signals completion with a final answer. The efficiency win is that the *expensive* reasoning happens in the planner/replanner, while execution can use a cheaper model or plain code; you also get an artifact — the plan — that you can log, inspect, and interrupt. This contrasts with ReAct, where the big model re-reasons the entire trajectory on every single turn.

**Where it appears in real systems.** In LangGraph this is a `StateGraph` whose state carries `input`, `plan: list[str]`, and `past_steps`, with nodes `planner`, `agent` (executor), and `replan`; the replan node returns a `Command`/conditional edge that routes back to the executor or to `END`. The `create_agent` harness (`langchain.agents`) ships a `TodoListMiddleware` that gives an agent an explicit write-a-todo-list-then-work-it capability — the productised form of plan-and-execute — and `SubAgentMiddleware` for delegating plan steps to isolated sub-agents.

### Replanning

**What it is.** The step that closes the Plan-and-Execute loop: after executing part of the plan, an LLM decides whether the original plan still holds, needs revision, or the task is done.

**How it works mechanistically.** The replanner is prompted with the original goal, the current plan, and the observations from executed steps, and returns a discriminated union — typically either a `Response` (final answer, terminate) or a `Plan` (new list of remaining steps, continue). This is what makes the pattern adaptive rather than a rigid one-shot script: if step 2 revealed the API is deprecated, the replanner can rewrite steps 3–5. The danger is that a replanner that never emits `Response` — because the goal is genuinely unachievable — will loop forever, so replanning must be capped.

**Where it appears in real systems.** A LangGraph `replan` node returning `Command(goto=..., update={"plan": ...})`, or a conditional edge whose routing function returns `END` versus the executor node. LangGraph's `recursion_limit` (default 1000 super-steps) is the backstop, but you should enforce a *task-level* cap far below that using a counter in state or the `RemainingSteps` managed value.

### Reflection / Self-Critique (generate → critique → revise)

**What it is.** A pattern where the model's first output is treated as a draft: a critique step evaluates it (against the task, rubric, or tool feedback) and the model regenerates an improved version, looping until "good enough" or a cap.

**How it works mechanistically.** The minimal loop has two roles that can be the same model with different prompts: a **generator** produces a draft, and a **reflector/critic** produces written feedback ("the intro is off-topic; cite a source for claim X"). That feedback is appended to the conversation and the generator runs again with it in context. Because the feedback is *natural-language* and stays in the message history, later generations condition on all prior critiques. This directly targets *quality* failures (weak reasoning, unsupported claims, style) that a single forward pass misses — at the cost of at least 2× the calls (generate + critique) per round.

**Where it appears in real systems.** In LangGraph, a two-node `StateGraph` — a `generate` node and a `reflect` node — with a conditional edge that ends when `len(messages) > threshold` or a critique-passes flag is set; the reflect node is a chat model given a "you are a teacher grading this draft" system prompt whose `AIMessage` is fed back in as a `HumanMessage` to the generator. The evaluator–optimizer workflow in the LangGraph docs is the same idea with an explicit accept/reject grade.

### Generator/Critic (Evaluator–Optimizer) Topology

**What it is.** A named variant of reflection where one LLM call (the optimizer/generator) proposes a solution and a *separate* LLM call (the evaluator/critic) scores it against explicit criteria and returns a pass/fail plus feedback, looping until pass.

**How it works mechanistically.** The evaluator uses structured output (e.g. a Pydantic schema `Grade(score: Literal["pass","fail"], feedback: str)`) so its verdict is machine-routable by a conditional edge. On "fail," the feedback is threaded back to the generator; on "pass," the graph terminates. Separating the two roles (often two prompts, sometimes two different models — a strong critic, a cheaper generator) reduces the "model rubber-stamps its own work" failure that happens when one prompt does both. This is the topology behind self-correcting code generation ("write code → run tests → if failing, feed errors back → rewrite").

**Where it appears in real systems.** LangGraph's *Evaluator-optimizer* workflow: two nodes (`generate`, `evaluate`) and a routing function that returns `"Accepted" -> END` or `"Rejected + Feedback" -> generate`. The evaluator commonly grades against real signals (unit tests, a retrieval-groundedness check) rather than pure LLM judgment, which makes the loop trustworthy.

### Reflexion (verbal reinforcement with episodic memory)

**What it is.** A specific reflection framework (Shinn et al., 2023) where an agent that fails a task writes a *verbal* self-reflection about *why* it failed, stores that reflection in an episodic memory buffer, and uses it to do better on the next attempt — reinforcement learning via language instead of weight updates.

**How it works mechanistically.** After a trial, an evaluator produces a feedback signal (a test result, a score, or environment reward); a self-reflection LLM converts that signal into a short natural-language lesson ("I assumed the file existed; next time check first"); this lesson is persisted across trials and prepended to the agent's context on the next attempt. The distinction from plain reflection is the *cross-trial episodic memory* — Reflexion accumulates lessons over multiple full attempts, not just within one generate→critique loop. The paper reports 91% pass@1 on HumanEval versus 80% for the GPT-4 baseline.

**Where it appears in real systems.** Implemented in LangGraph as a graph that persists reflection strings via a checkpointer / `store` and re-injects them; conceptually it is the "long-term memory of past mistakes" layer that sits above a single-turn reflection loop.

### Tree-of-Thoughts (brief contrast)

**What it is.** A reasoning framework (Yao et al., 2023) that generalises linear chain-of-thought into a *tree*: the model generates multiple candidate next "thoughts" at each step, self-evaluates them, and searches (BFS/DFS) over the tree, backtracking from dead ends.

**How it works mechanistically.** At each node the model proposes N candidate partial solutions, a value/evaluation prompt scores each ("sure / maybe / impossible"), and a search algorithm expands promising branches and prunes bad ones — enabling lookahead and backtracking that linear reflection cannot do. It shines on problems with a large search space and verifiable intermediate states (Game of 24: 74% vs 4% for CoT). The cost is steep: the branching factor multiplies LLM calls, so ToT is reserved for hard search/planning problems, not everyday agent tasks.

**Where it appears in real systems.** Rarely productised as-is due to cost; the practical takeaway is the *idea* — generate several candidates, evaluate, prune — which shows up as "sample N plans, score, pick best" in planning nodes and as best-of-N generation with an evaluator.

### Composition with the ReAct / tool-calling loop (chapter 01)

**What it is.** These patterns are not replacements for ReAct — they *wrap* or *contain* it. A plan step is often executed *by* a ReAct agent; a reflection loop can critique the output *of* a ReAct agent.

**How it works mechanistically.** In Plan-and-Execute, the executor node is frequently a small `create_agent` ReAct agent that handles one plan step with its own tool-calling loop, so you get local flexibility (ReAct) under global structure (the plan). In an evaluator–optimizer, the generator can be a full tool-using agent whose transcript the critic grades. LangGraph makes this composition literal: a compiled agent is a subgraph you drop in as a node inside a larger `StateGraph`.

**Where it appears in real systems.** A LangGraph `StateGraph` where the `agent` node is itself `create_agent(...)` (a subgraph), invoked once per plan step; the planner/replanner and critic are separate nodes in the outer graph.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| `max_replan_iterations` | Hard cap on planner→execute→replan cycles before forced termination | Set to 3–5 for long tasks; below 3 you lose adaptivity, uncapped is the #1 runaway-cost bug. Force a "best-effort / gave up" answer at the cap. |
| `max_reflections` (reflection rounds) | How many generate→critique→revise cycles run | Start at 1–2; diminishing returns and cost climb fast after 2. Cap at 3 unless an offline eval shows later rounds still lift quality. |
| Executor model tier | Which model runs plan steps | Use a cheaper/faster model for execution and reserve the expensive model for planning/replanning — that split is the main cost win of the pattern. |
| Critic separation (shared vs distinct model/prompt) | Whether generator and critic are the same call | Use a distinct prompt (ideally distinct/stronger model) for the critic; a single prompt that self-grades tends to rubber-stamp its own output. |
| Evaluation signal (LLM-judge vs verifiable) | What the critic grades against | Prefer a *verifiable* signal (unit tests, groundedness check, schema validation) over pure LLM judgment whenever one exists — it makes the loop trustworthy and cheaper to trust. |
| `recursion_limit` (LangGraph) | Max super-steps before `GraphRecursionError` | Leave the default (1000) as a safety net but enforce a much smaller *task-level* cap in state; do not rely on `recursion_limit` as your primary bound. |
| ToT branching factor / beam width | Candidates generated & kept per step | Keep tiny (2–3) — cost is multiplicative per depth; only widen for hard search problems with cheap, verifiable evaluation. |

### Worked Example: Requirement → Decision

**Given:** You are building an agent that generates a small Python data-transformation script from a natural-language spec, for an internal data team. The current single-shot ReAct agent produces code that runs but frequently has subtle logic bugs (~30% of outputs fail the team's hidden unit tests). Correctness matters more than latency (users wait happily up to ~30 s), cost is moderate, and you already have a sandbox that can execute candidate code against tests.

- **Step 1 — Identify the goal:** Cut the rate of logically-wrong-but-runnable scripts by letting the agent detect and fix its own failures before returning.
- **Step 2 — Define inputs:** The NL spec, a code-generation model, a sandbox that runs the generated code against the team's tests and returns pass/fail + error output.
- **Step 3 — Define outputs:** Either code that passes the tests, or an explicit "couldn't satisfy the spec after N tries" message with the last attempt and errors — never a silently-wrong script.
- **Step 4 — Apply constraints:** Correctness ≫ latency (≤ ~30 s allows several rounds); a *verifiable* signal exists (the test suite); loop must terminate deterministically; cost moderate.
- **Step 5 — Select the approach:** Use an **evaluator–optimizer (generator/critic) reflection loop** where the "critic" is the *actual test run* (not an LLM judge): generate → run tests → on failure, feed the errors back and regenerate, capped at `max_reflections = 3`, then return best-effort. *Rationale vs alternatives:* Plan-and-Execute is overkill — this is one short task, not a long-horizon multi-step plan; Tree-of-Thoughts' branching is unjustified cost when a linear fix-the-errors loop plus a real test oracle already targets the exact failure; a pure LLM-judge critic is weaker and more expensive to trust than the free, verifiable test signal you already have.

---

## Implementation

```python
# Scenario: A long-horizon research task ("compile a competitor pricing brief") is slow and
# drifts on a pure ReAct agent because the big model re-reasons the whole trajectory every turn.
# Plan-and-Execute plans ONCE with the strong model, executes each step with a cheaper ReAct
# agent, and replans only after each step — fewer expensive calls + an inspectable plan.
# API verified against LangGraph Graph API (StateGraph / Command) at docs.langchain.com.
from typing import Annotated, Union, Literal
from typing_extensions import TypedDict
from operator import add
from pydantic import BaseModel
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent

class Plan(BaseModel):
    steps: list[str]                       # ordered remaining steps

class Response(BaseModel):
    answer: str                            # signals completion

class PlanState(TypedDict):
    input: str
    plan: list[str]
    past_steps: Annotated[list[tuple], add]  # (step, result) pairs accumulate
    replans: int                             # <-- bound the loop

planner_model = init_chat_model("openai:gpt-5.5", temperature=0)   # expensive: plan/replan
executor = create_agent("openai:gpt-5-mini", tools=[...])          # cheap: run one step

def plan_step(state: PlanState):
    plan = planner_model.with_structured_output(Plan).invoke(
        [{"role": "user", "content": f"Make a step-by-step plan for: {state['input']}"}])
    return {"plan": plan.steps, "replans": 0}

def execute_step(state: PlanState):
    step = state["plan"][0]
    result = executor.invoke({"messages": [{"role": "user", "content": step}]})
    return {"past_steps": [(step, result["messages"][-1].content)]}

MAX_REPLANS = 5

def replan_step(state: PlanState) -> Command[Literal["execute", "__end__"]]:
    decision = planner_model.with_structured_output(Union[Plan, Response]).invoke(
        [{"role": "user", "content":
          f"Goal: {state['input']}\nDone so far: {state['past_steps']}\n"
          "Return a Response if finished, else the remaining Plan."}])
    if isinstance(decision, Response) or state["replans"] >= MAX_REPLANS:
        return Command(goto=END, update={"plan": []})
    return Command(goto="execute",
                   update={"plan": decision.steps, "replans": state["replans"] + 1})

g = StateGraph(PlanState)
g.add_node("plan", plan_step)
g.add_node("execute", execute_step)
g.add_node("replan", replan_step)
g.add_edge(START, "plan")
g.add_edge("plan", "execute")
g.add_edge("execute", "replan")     # replan_step's Command routes to "execute" or END
graph = g.compile()
```

```python
# Anti-pattern: an UNBOUNDED reflection loop that critiques forever.
# The critic almost never says "perfect," so generate->critique->generate cycles indefinitely,
# burning 2 LLM calls per round; a single request can rack up dozens of calls and blow any
# latency/cost budget. The happy path terminates in testing, so the bug ships to prod.
def should_continue(state):                      # BROKEN: no cap
    if _critique_says_perfect(state):
        return "end"
    return "generate"                            # -> critique -> generate -> ... forever

# Correct approach: cap the rounds in state AND prefer a verifiable signal over LLM self-judging.
class ReflectState(TypedDict):
    messages: Annotated[list, add]
    reflections: int

MAX_REFLECTIONS = 2

def reflect(state: ReflectState):
    # Verifiable critic: run tests, not an LLM opinion, when a signal exists.
    passed, feedback = _run_tests(state["messages"][-1].content)
    return {"messages": [{"role": "user", "content": feedback}],
            "reflections": state["reflections"] + 1, "_passed": passed}

def should_continue(state: ReflectState) -> Literal["generate", "__end__"]:
    if state.get("_passed") or state["reflections"] >= MAX_REFLECTIONS:
        return END                               # accept, or give up at the cap
    return "generate"
# What breaks without the cap: cost and latency are unbounded and non-deterministic; the fix
# makes worst-case cost = MAX_REFLECTIONS + 1 generations and guarantees termination, while the
# real test signal stops the critic from rubber-stamping (or endlessly nitpicking) its own draft.
```

---

## Common Pitfalls & Misconceptions

- **Treating Plan-and-Execute as a rigid one-shot script** — beginners assume the plan, once written, must be followed to the letter, so they skip the replanner. Reality diverges (a tool errors, a step reveals new facts); the correct mental model is *plan-then-adapt* — the replanner exists precisely to revise remaining steps when observations contradict the plan.
- **Unbounded reflection / replan loops** — the happy path always terminates in local testing, so caps feel unnecessary. In production a genuinely unsatisfiable task makes the critic reject forever or the replanner never finish; always thread an iteration counter through state and force a best-effort/give-up branch at the cap (`recursion_limit` is a crash backstop, not a design bound).
- **Letting one prompt both generate and grade itself** — it seems efficient to have the model critique its own output in the same call, but a self-grading generator tends to rubber-stamp its work. Use a separate critic prompt (ideally a distinct or stronger model), and grade against a *verifiable* signal (tests, groundedness) whenever one exists.
- **Reaching for reflection to fix an incompleteness problem (or planning to fix a quality problem)** — people apply whichever pattern they just learned. Match the pattern to the failure: reflection fixes *low-quality/wrong* outputs, Plan-and-Execute fixes *drift/inefficiency on long multi-step* tasks; using reflection on a task that simply needs more steps just re-polishes an incomplete answer.
- **Believing Plan-and-Execute is always faster/cheaper than ReAct** — it *can* be (fewer big-model calls, cheap executor), but on short unpredictable tasks the upfront planning overhead and rigid structure make it slower and clunkier than a lightweight ReAct loop; the win is specifically on *long-horizon* work where re-reasoning every turn is the bottleneck.

---

## Key Definitions

| Term | Definition |
|---|---|
| Plan-and-Execute | Agent topology that plans all steps up front (planner), runs them (executor), and revises the plan on divergence (replanner), separating expensive reasoning from cheap execution. |
| Planner | The LLM call that turns a goal into an explicit ordered list of steps before execution begins. |
| Executor | The component that carries out an individual plan step — often a cheaper model, plain code, or a small ReAct sub-agent. |
| Replanner | The step that, after execution, either revises the remaining plan or emits a final answer to terminate the loop. |
| Reflection / self-critique | Pattern where a draft output is critiqued and regenerated in a generate→critique→revise loop until acceptable or capped. |
| Evaluator–optimizer (generator/critic) | Reflection variant where a separate evaluator LLM grades the generator's output (often via structured pass/fail + feedback) and routes accept vs regenerate. |
| Reflexion | A framework (Shinn et al., 2023) that stores verbal self-reflections about past failures in episodic memory to improve across trials — "verbal reinforcement learning." |
| Tree-of-Thoughts (ToT) | A reasoning framework (Yao et al., 2023) that generates multiple candidate thoughts per step, self-evaluates, and searches/backtracks over a tree of reasoning paths. |
| `recursion_limit` | LangGraph's max super-steps per run (default 1000) before `GraphRecursionError`; a crash backstop, not a substitute for a task-level loop cap. |

---

## Summary / Quick Recall

- ReAct re-reasons every turn — flexible but slow/expensive/drift-prone on long tasks; Plan-and-Execute plans once, executes cheaply, and replans only on divergence.
- Plan-and-Execute wins buy: fewer expensive reasoning calls, an inspectable/interruptible plan, and lower latency on long-horizon work — not on short unpredictable tasks.
- Reflection (generate→critique→revise) trades *more* calls for *higher quality*; the evaluator–optimizer topology separates a critic from the generator, ideally grading a verifiable signal (tests, groundedness).
- Reflexion adds cross-trial episodic memory of past mistakes; Tree-of-Thoughts branches and searches multiple reasoning paths — powerful but the branching factor makes it expensive.
- These compose *with* ReAct, not against it: a plan step is often executed by a ReAct sub-agent; a reflection loop critiques an agent's output. In LangGraph a compiled agent is a subgraph node.
- **Always bound replan and reflection loops** with an explicit iteration cap and a forced fallback; `recursion_limit` is only a crash backstop.

---

## Self-Check Questions

1. In the Plan-and-Execute pattern, what is the specific job of the *replanner*, and how does it differ from the planner?

   <details><summary>Answer</summary>

   The **planner** runs once at the start and turns the goal into an initial ordered list of steps. The **replanner** runs *after* one or more steps have executed: it inspects the goal, the current plan, and the observations, and returns either a revised list of remaining steps (continue) or a final response (terminate). The tempting wrong answer is "they're the same LLM so they do the same thing" — the distinction is *when* they run and *what they consume*: the planner sees only the goal, the replanner sees goal + progress and is what makes the pattern adaptive rather than a rigid one-shot script.

   </details>

2. You have a pure ReAct agent doing a 15-step research-and-report task. It is slow, costs a lot, and sometimes loses track of the original goal halfway through. Which pattern change most directly addresses all three symptoms, and why?

   <details><summary>Answer</summary>

   **Plan-and-Execute.** Writing the full plan up front with the strong model and then executing steps with a cheaper executor cuts the number of expensive reasoning calls (cost + latency), and the explicit persisted plan keeps the agent anchored to the original goal instead of re-deriving intent every turn (drift). Adding a *reflection* loop instead would be the tempting wrong choice — it improves output *quality* but adds even more calls and does nothing about the long-horizon drift/efficiency problem, which is what all three symptoms point to.

   </details>

3. **Which TWO** of the following correctly describe the evaluator–optimizer (generator/critic) reflection topology?
   - A. Using a separate critic prompt/model reduces the tendency of a model to rubber-stamp its own output.
   - B. It eliminates the need to bound the number of reflection rounds.
   - C. Grading against a verifiable signal (e.g. unit tests) is more trustworthy than pure LLM self-judgment.
   - D. It reduces total LLM calls compared to a single forward pass.
   - E. It is only applicable to code-generation tasks.

   <details><summary>Answer</summary>

   **A and C.** A is correct because separating the critic (distinct prompt, often a stronger model) counters the self-grading rubber-stamp failure. C is correct because a verifiable oracle (tests, groundedness check) is a stronger, cheaper-to-trust signal than an LLM opinion. B is the most tempting distractor and is wrong — the critic rarely says "perfect," so the loop still *must* be capped or it runs forever. D is false: reflection *adds* calls (generate + critique per round), it never reduces them below a single pass. E is false: the topology applies to any task with gradeable output (writing, analysis, retrieval answers), not just code.

   </details>

4. A teammate implements a reflection loop for a summarization agent with no iteration cap, reasoning "the critic will naturally stop once the summary is good." In production, cost per request occasionally spikes 20×. Diagnose the root cause and give the fix.

   <details><summary>Answer</summary>

   The loop is **unbounded**: for inputs where the critic never judges the summary "good enough" (subjective quality has no hard oracle), generate→critique→generate cycles indefinitely, each round costing ~2 LLM calls, so a single request accumulates many calls and spikes cost. The fix is to thread a `reflections` counter through state and force acceptance/best-effort at `max_reflections` (typically 2), and — since summary quality has no verifiable signal — accept that the cap, not the critic, guarantees termination. Swapping to a bigger critic model would not fix it; it would make each of the unbounded rounds *more* expensive.

   </details>

5. For each scenario pick ReAct, Plan-and-Execute, reflection, or Tree-of-Thoughts, and justify: (a) a 2–3 step "look up a fact and answer" bot; (b) an agent solving Game-of-24-style puzzles needing lookahead over many candidate moves; (c) a long multi-stage migration script generator that keeps producing subtly wrong code but you have a test suite.

   <details><summary>Answer</summary>

   (a) **ReAct** — few, unpredictable steps; plan/reflection overhead isn't justified for a short lookup. (b) **Tree-of-Thoughts** — the task needs exploring and comparing many candidate reasoning paths with backtracking, exactly what ToT's branch-evaluate-prune search provides; linear reflection can't do lookahead. (c) **Reflection (evaluator–optimizer with the test suite as critic)** — the failure is *quality/correctness* with a verifiable oracle available, so generate→run tests→fix, bounded. The tempting error is choosing Plan-and-Execute for (c) because it says "long multi-stage"; but the observed failure is wrong code, not step-drift, and the test suite makes a bounded reflection loop the direct fix — Plan-and-Execute wouldn't correct logic bugs a step produces.

   </details>

---

## Further Reading

- [LangGraph Graph API (StateGraph, nodes, conditional edges, Command, recursion_limit, RemainingSteps)](https://docs.langchain.com/oss/python/langgraph/graph-api) — *verified 2026-07-29* — Reference for the `StateGraph`/`Command`/conditional-edge primitives used to build bounded plan-and-execute and reflection loops.
- [LangChain Agents (`create_agent`, TodoListMiddleware, SubAgentMiddleware)](https://docs.langchain.com/oss/python/langchain/agents) — *verified 2026-07-29* — Official agent harness; the productised planning/delegation middleware and the ReAct executor you drop into a plan step.
- [LangChain Middleware overview (call limits, summarization, HITL)](https://docs.langchain.com/oss/python/langchain/middleware) — *verified 2026-07-29* — Hooks for bounding and steering the agent loop, including call limits that cap runaway reflection/replan cycles.
- [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al., 2023)](https://arxiv.org/abs/2303.11366) — *verified 2026-07-29* — Primary source for verbal self-reflection with episodic memory across trials (91% pass@1 on HumanEval).
- [Tree of Thoughts: Deliberate Problem Solving with Large Language Models (Yao et al., 2023)](https://arxiv.org/abs/2305.10601) — *verified 2026-07-29* — Primary source for branching thought generation, self-evaluation, and search/backtracking over reasoning paths.
