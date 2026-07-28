# Observability & Monitoring Patterns for AI System Design

**Section:** 05 Interview Practice → Cross-Cutting Concerns & Trade-off Patterns | **Interview relevance:** High — once you have designed the happy path, the interviewer asks "it's live and giving wrong answers — how do you find out *which step* broke?"; answering forces you to reason about distributed tracing across agent/RAG steps, LLM-specific telemetry (tokens, cost, cache hits), online quality/drift monitoring, alerting, and feedback loops back into evaluation.

---

## TL;DR

An LLM or agent request is a distributed, multi-step operation (retrieve → tool call → LLM call → …), so you observe it the way you observe any distributed system — one **trace** per request, one **span** per step — plus LLM-specific signals the classic three telemetry pillars (traces, metrics, logs) don't cover: token counts, cost, cache-hit rate, tool-call success, and guardrail/hallucination flags. You alert on p95 latency, error rate, and cost spikes; you monitor output quality and input drift *online*; and you capture user feedback (thumbs up/down) to feed offline evaluation. Everything hangs off a propagated **trace ID** so a bad answer in production can be replayed step-by-step. **The one thing to remember: if you only log the final answer, you cannot debug an agent — you need one span per step under a shared trace ID, so instrument the pipeline, not just the endpoint.**

---

## ELI5 — Explain It Like I'm 5

Imagine a package delivery that passes through five sorting hubs before it reaches you, and it arrives damaged. If the only record is "package arrived damaged," you have no idea which hub dropped it. But if every hub stamps the package with the same tracking number and logs "received OK / handed off OK," you can pull up the whole journey and see exactly where the dent appeared. A trace is that tracking number, and each span is one hub's stamp. The beginner mistake is checking only the final delivery photo — you must be able to open the timeline and watch every hop, because the failure is almost never in the last step you can see.

---

## Learning Objectives

By the end of this note you will be able to:
- [ ] Decompose a production AI request into a trace of nested spans (retrieval, tool call, LLM call) each carrying token/cost/latency attributes.
- [ ] Name the three classic telemetry types (traces, metrics, logs) and the LLM-specific signals layered on top (tokens, cost, cache-hit, tool-call success, guardrail flags).
- [ ] Select the right observability pattern — tracing, structured logging, online quality monitoring, drift detection, alerting, feedback capture — for a stated production symptom.
- [ ] Justify sampling, retention, and alert-threshold choices against their trade-offs (storage cost, PII exposure, alert fatigue) in an interview.

---

## Visual Overview

### Trace tree of one agent run (nested spans under a shared trace ID)

```
trace_id=abc123  ── "support-agent request"  (root span, 4.2 s)
   │
   ├─► span: retrieve            (0.3 s)  k=6, hit=true,  input_tokens=—
   │
   ├─► span: agent-turn-1        (1.1 s)
   │      └─► span: llm.call     (0.9 s)  in=1450 out=80  cost=$0.004  cache_read=1200
   │
   ├─► span: tool.call get_order (0.4 s)  status=OK
   │
   └─► span: agent-turn-2        (2.4 s)
          └─► span: llm.call     (2.2 s)  in=1600 out=420 cost=$0.011 guardrail=flag
                                          ▲ wrong answer surfaced HERE
```

### Metrics → alerting → feedback loop

```
per-span attributes ──► metrics aggregator ──► dashboards (p50/p95/p99, cost/day, cache-hit%)
        │                                            │
        │                                            └──► alert rules ──► page on p95>SLO, err%>2%, cost spike
        ▼
   structured logs (prompt, response, tokens)
        │
 user thumbs up/down ──► feedback store ──► offline eval set ──► regression gate (03-evaluation-*)
```

---

## The Core Problem

A single production AI request is not one call — it is a *pipeline* of dependent steps (embed → retrieve → rerank → agent turn → tool call → LLM call → agent turn …), often spread across services. When the final answer is wrong, slow, or expensive, the interviewer's question is "which step caused it?" You cannot answer that from an application log that records only the endpoint's final output. You need the *shape* of the request: what each step received, what it returned, how long it took, how many tokens and dollars it burned, and whether a guardrail fired.

This is classic distributed-systems observability with an LLM twist. The three telemetry pillars apply directly:

- **Traces** — the causal timeline of one request as a tree of spans (one span per step), tied together by a propagated trace ID.
- **Metrics** — numeric time series aggregated across requests (p95 latency, error rate, requests/sec, cost/day).
- **Logs** — discrete structured records of events (the prompt sent, the response received, token usage).

But LLM systems add signals the three pillars don't natively describe: **token counts** (input/output/cached), **cost**, **cache-hit rate**, **tool-call success/failure**, and **quality signals** (guardrail flags, hallucination detectors, user feedback). These attach *to spans and metrics* — e.g. the OpenTelemetry GenAI conventions define span attributes like `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and `gen_ai.usage.cache_read.input_tokens` precisely so a token/cost view falls out of the same trace you use to debug latency. The design task is deciding *what* to record, *how much* (sampling), *how long* (retention), and *what to alert on* — under real constraints of storage cost and PII exposure.

---

## Resolution Options

| Option | How it works | Buys you | Costs you | Reach for it when… |
|---|---|---|---|---|
| **Distributed tracing (spans per step)** | One trace ID per request; wrap each step (retrieval, tool, LLM) in a span with attributes | Step-level root-cause; replay; latency attribution | Instrumentation effort; span storage; PII in captured prompts | Any multi-step agent/RAG system — the default |
| **Structured logging** | Emit JSON logs of prompt/response/tokens/latency keyed by trace_id | Queryable audit trail; grep-free analysis | Log volume/cost; PII handling; noise if unstructured | You need to search/aggregate over request content |
| **Metrics + dashboards** | Aggregate span attributes into time series (p95, err%, cost/day, cache-hit%) | System-wide health at a glance; SLO tracking | Cardinality blow-up if over-labelled; no per-request detail | Ongoing production health monitoring |
| **Alerting** | Threshold/anomaly rules over metrics → page/notify | Fast detection of regressions before users complain | Alert fatigue if thresholds wrong; missed signals if too loose | You have an SLO to defend (latency, error, cost) |
| **Online quality monitoring & drift detection** | Score live outputs (LLM-judge/heuristics) and compare input/output distributions vs baseline | Catch silent quality decay & input drift without labels | Judge cost; false alarms; defining "drift" is non-trivial | Quality can degrade silently (model/data change) |
| **Feedback capture** | Collect thumbs up/down + edits, bind to run/trace | Real labels for eval sets; prioritizes what to fix | Sparse/biased signal; UI work; storage | You want production signal to drive offline eval |

**Distributed tracing** — the backbone. A trace ID is generated at the request boundary and propagated through every downstream step; each step opens a span that records its start/end, status, and typed attributes. For LLM calls the OpenTelemetry GenAI conventions standardize a span named `{operation} {model}` (e.g. `chat gpt-4`) carrying `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`, and (opt-in) the input/output messages. In practice this appears as a LangSmith **run** (LangSmith explicitly models a trace as a collection of runs/spans, grouped into a **project**, and multi-turn conversations as a **thread**) or an OTel span exporter feeding your backend. This is what lets you replay a bad answer step-by-step. See `01-latency-and-response-time-patterns.md` — the same spans give you TTFT and per-hop latency.

**Structured logging** — emit each notable event as a JSON object (`{trace_id, step, prompt_hash, input_tokens, output_tokens, latency_ms, cache_hit}`) rather than free-text. Structured means machine-queryable: you can aggregate token usage per user or filter to all runs where a guardrail fired. The key discipline is keying every log line by `trace_id` so logs, traces, and metrics correlate. Sensitive fields (raw prompts/responses) are the PII risk — the GenAI conventions make full message capture **opt-in** for exactly this reason.

**Metrics + dashboards** — roll span attributes up into time series: latency histograms (p50/p95/p99), request and error rates, cost per day, tokens per request, cache-hit rate, tool-call success rate. These are the numbers you put on a dashboard and attach SLOs to. The GenAI metrics conventions define instruments such as token-usage and operation-duration histograms so these views are consistent across providers. The trap is **cardinality**: labelling metrics with high-uniqueness fields (user ID, full prompt) explodes the time-series count and cost.

**Alerting** — rules over metrics that fire when a threshold or anomaly is crossed: p95 latency > SLO, error rate > 2%, daily cost > budget, cache-hit rate collapsing (a caching regression), tool-call failure spiking. Providers also give account-level guards — OpenAI's dashboard supports spend alerts and hard spend limits — which complement in-app alerts. The trade-off is fatigue: thresholds too tight page constantly and get muted; too loose and you learn about incidents from users.

**Online quality monitoring & drift detection** — quality can decay with no error and no latency change (a model version shift, a data-source change, a prompt edit). You monitor it online by scoring a sample of live outputs (heuristics, guardrail checks, or an LLM-as-judge) and by watching *distributions*: input drift (query embeddings/topics shifting away from your eval baseline) and output drift (answer length, refusal rate, sentiment). LangSmith supports **online evaluations** via automation rules for this. A spike in the guardrail-flag rate or a drop in judge score is your early-warning signal. Deep-dives live in `03-evaluation-and-quality-assurance-patterns.md`.

**Feedback capture** — collect explicit signals (thumbs up/down, star ratings, user edits) and bind each to its run/trace. LangSmith models this as **feedback** — a tag + score attached to a run by run ID. This turns production into a labelling pipeline: negatively-rated traces become candidate regression cases for your offline eval set, closing the loop from monitoring back into evaluation.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Trace sampling rate | Fraction of requests fully traced | Trace 100% while debugging or at low volume; head/tail-sample (e.g. keep all errors + N% of successes) once volume makes full capture costly |
| Message/content capture | Whether full prompts/responses are stored on spans | Off (or hashed/redacted) by default for PII; enable full capture only in envs where prompt data is permitted — GenAI conventions make it opt-in |
| Log/trace retention | How long telemetry is kept | Match the longest window you actually debug/analyze over; keep costly full-content traces short, cheap aggregate metrics long (LangSmith SaaS default is 400 days; datasets persist indefinitely) |
| Alert threshold (p95, err%, cost) | When a rule pages | Set from the SLO plus a margin above normal variance; tune until true-positive rate is high enough that on-call trusts the page |
| Online-eval sample rate | Fraction of live outputs scored by judge/heuristics | Scale down as judge cost rises; sample enough to detect a meaningful quality drop within your detection-time budget |
| Metric label cardinality | Which dimensions tag a metric | Label only low-cardinality fields (model, route, status); never label with user ID or raw prompt |

### Worked Example: Requirement → Decision

**Given:** A production support agent (RAG + tools, built on LangGraph) starts returning confidently wrong order statuses. Users complain; there are no errors and latency looks normal. You must find which step failed and stop the bleeding.

- **Step 1 — Identify the goal:** Root-cause *which step* produces the wrong answer, then decide the mitigation — this is a diagnosis, not a latency tune.
- **Step 2 — Define inputs:** The traces for a set of thumbs-down runs, each with spans for retrieval, tool calls, and LLM calls, plus their attributes (retrieved doc IDs, tool status, token counts, guardrail flags).
- **Step 3 — Define outputs:** The offending span and a hypothesis (bad retrieval? stale tool result? model regression?), plus an action.
- **Step 4 — Apply constraints:** No labelled offline set yet (must use production feedback); prompts contain PII (content capture is redacted, so reason from attributes + hashes); must not ship an unverified prompt change blind.
- **Step 5 — Diagnose & select:** Pull the thumbs-down traces (feedback → trace ID). Inspect spans top-down: retrieval span shows `hit=true` and correct doc IDs → retrieval is fine. The `get_order` **tool span** shows `status=OK` but returns a stale cached value → the wrong answer is *upstream of the LLM*; the model faithfully reported bad tool data. **Decision:** fix the tool's cache invalidation, not the prompt. Rationale vs alternatives: without per-step spans you'd have blamed the LLM and edited the prompt (fixing nothing); the trace localizes the fault to the tool step. Add an **alert** on tool-result staleness and add these traces to the eval set (`03-evaluation-and-quality-assurance-patterns.md`) as regression cases.

---

## Implementation

```python
# Scenario: a multi-step RAG+tool agent gives occasional wrong answers and we
# cannot tell which step is at fault. We instrument each step as a span under a
# shared trace so a bad run can be pulled up and replayed hop-by-hop. Using
# OpenTelemetry GenAI-style attributes so token/cost views fall out of the same trace.
from opentelemetry import trace

tracer = trace.get_tracer("support-agent")

def handle_request(query: str, user_id: str):
    # root span == the whole request; every child span shares its trace_id
    with tracer.start_as_current_span("support-agent request") as root:
        root.set_attribute("gen_ai.conversation.id", user_id)  # low-cardinality, no PII in value

        with tracer.start_as_current_span("retrieve") as s:
            docs = retrieve(query, k=6)
            s.set_attribute("retrieval.k", 6)
            s.set_attribute("retrieval.hit", bool(docs))

        with tracer.start_as_current_span("chat gpt-4") as s:   # GenAI span name: {operation} {model}
            resp = llm.chat(query, docs)
            # standardized GenAI usage attributes → token & cost dashboards for free
            s.set_attribute("gen_ai.operation.name", "chat")
            s.set_attribute("gen_ai.request.model", "gpt-4")
            s.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
            s.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
            s.set_attribute("gen_ai.usage.cache_read.input_tokens", resp.usage.cache_read_tokens)
            s.set_attribute("gen_ai.response.finish_reasons", resp.finish_reasons)
        return resp
```

```python
# Anti-pattern: log ONLY the final answer at the endpoint. When the answer is
# wrong you have zero visibility into retrieval, tool calls, or the LLM step —
# nothing to replay, no per-step latency, no token/cost attribution.
import logging

def handle_request_bad(query: str):
    answer = run_agent(query)          # retrieval + tools + LLM all invisible
    logging.info(f"answer: {answer}")  # unstructured, no trace_id, no per-step spans
    return answer                      # a wrong answer here is undebuggable

# Correct approach: one structured log line per step, all keyed by trace_id, so
# logs correlate with the trace and you can aggregate/query by step and user.
import json, logging

def handle_request_good(query: str, trace_id: str):
    docs = retrieve(query, k=6)
    logging.info(json.dumps({"trace_id": trace_id, "step": "retrieve",
                             "k": 6, "hit": bool(docs)}))
    resp = llm.chat(query, docs)
    logging.info(json.dumps({"trace_id": trace_id, "step": "llm.chat",
                             "input_tokens": resp.usage.input_tokens,
                             "output_tokens": resp.usage.output_tokens,
                             "latency_ms": resp.latency_ms}))
    return resp   # every step is now correlatable by trace_id
```

---

## Common Pitfalls & Misconceptions

- **Logging only the final answer** — beginners instrument the HTTP endpoint like a CRUD app, but an AI request is a multi-step pipeline, so the correct mental model is one span per step under a shared trace ID — instrument the pipeline, not just the boundary.
- **Treating "no errors + normal latency" as "healthy"** — people assume green dashboards mean quality is fine, but LLM quality decays silently with no exception and no latency change, so you must monitor output quality and input/output drift online, not only infrastructure metrics.
- **Capturing full prompts/responses everywhere by default** — it feels helpful to store everything, but prompts routinely contain PII and blow up storage, so make full content capture opt-in and redact/hash by default while keeping cheap structured attributes (tokens, status) always on.
- **Over-labelling metrics with high-cardinality fields** — tagging a metric with user ID or raw prompt seems informative, but it explodes the time-series count and cost, so label only low-cardinality dimensions (model, route, status) and keep per-request detail in traces/logs.

---

## Key Definitions

| Term | Definition |
|---|---|
| Trace | The end-to-end record of one request as a tree of spans, tied together by a shared trace ID; in LangSmith, a collection of runs for a single operation. |
| Span (run) | A single timed unit of work within a trace (a retrieval, tool call, or LLM call) with a status and typed attributes; LangSmith calls this a run. |
| Trace ID propagation | Passing the same identifier through every downstream step so all its spans/logs/metrics correlate to one request. |
| Structured logging | Emitting log events as machine-parseable records (e.g. JSON) with consistent fields, rather than free-text, so they can be queried and aggregated. |
| Telemetry pillars | Traces, metrics, and logs — the three complementary views of a running system. |
| Drift | A shift in the live input or output distribution away from the baseline the system was validated on (e.g. new query topics, rising refusal rate), often preceding quality decay. |
| Sampling (traces) | Recording only a fraction of requests (often keeping all errors + a percentage of successes) to bound telemetry cost. |
| Feedback | An explicit user/annotator signal (thumbs up/down, score, edit) bound to a run, used as a label to drive evaluation. |

---

## Summary / Quick Recall

- One trace per request, one span per step, all under a propagated trace ID — that is what makes an agent debuggable.
- The three pillars (traces, metrics, logs) apply directly; layer LLM-specific signals on top: tokens, cost, cache-hit, tool-call success, guardrail/hallucination flags.
- OpenTelemetry GenAI conventions standardize span/metric attributes (`gen_ai.usage.input_tokens`, etc.); LangSmith models traces as runs in a project, threads for multi-turn, feedback for labels.
- Alert on p95 latency, error rate, and cost spikes against an SLO; tune thresholds to avoid fatigue.
- Monitor quality and drift *online* — green infra dashboards do not prove quality is intact.
- Capture user feedback and route negatively-rated traces into your offline eval set (`03-evaluation-and-quality-assurance-patterns.md`).
- Sample and set retention to bound cost; make full prompt/response capture opt-in for PII; never label metrics with high-cardinality fields.

---

## Self-Check Questions

1. What are the three classic telemetry pillars, and which LLM-specific signals do you layer on top of them for an AI system?

   <details><summary>Answer</summary>

   The three pillars are traces, metrics, and logs. On top you add LLM-specific signals: token counts (input/output/cached), cost, cache-hit rate, tool-call success/failure, and quality signals (guardrail/hallucination flags, user feedback). Answering only "traces, metrics, logs" is incomplete because those pillars don't natively describe tokens, cost, or quality — the whole point of GenAI observability is attaching those as span/metric attributes.

   </details>

2. A production agent returns wrong answers but shows no errors and normal latency. What is your first diagnostic move, and why?

   <details><summary>Answer</summary>

   Pull the traces for thumbs-down (or otherwise flagged) runs and inspect the spans top-down — retrieval, then tool calls, then LLM calls — to localize which step produced the fault. This works because the failure is almost never observable from the final answer alone; per-step spans reveal whether retrieval missed, a tool returned stale data, or the model itself regressed. The tempting wrong move is editing the prompt immediately, which fixes nothing if (as is common) the fault is upstream in a tool or retrieval step.

   </details>

3. Your daily LLM spend doubled overnight with no traffic increase. Which telemetry would you inspect first, and what could it reveal?

   <details><summary>Answer</summary>

   Inspect per-span token-usage attributes (`gen_ai.usage.input_tokens`/`output_tokens`) and the cache-hit-rate metric. A cost spike at flat traffic usually means tokens per request rose (a prompt/context change inflated input tokens) or the cache-hit rate collapsed (a caching regression forcing full re-processing). The distractor is blaming raw request volume — but volume is flat, so cost/token or tokens/request must have changed, which only the token and cache metrics reveal.

   </details>

4. **Which TWO** of the following are correct trade-off decisions when instrumenting a high-volume production AI system?
   - A. Trace 100% of requests forever to never miss anything
   - B. Head/tail-sample traces — keep all errors plus a percentage of successes — to bound storage cost
   - C. Label latency metrics with the user ID so you can slice by user
   - D. Make full prompt/response capture opt-in and redact by default
   - E. Store full-content traces as long as you store aggregate metrics

   <details><summary>Answer</summary>

   **B and D.** Sampling (keep all errors + N% of successes) preserves debuggability while bounding cost, and opt-in/redacted content capture controls PII exposure and storage — both are standard trade-offs. A is wrong because 100%-forever full capture is prohibitively costly and PII-heavy at high volume. C is the most tempting wrong pick: user ID feels useful, but it is high-cardinality and explodes metric time-series cost — put per-user detail in traces/logs, not metric labels. E is wrong because full-content traces are expensive and should be retained *shorter* than cheap aggregate metrics.

   </details>

5. Two proposals to catch a silent quality regression after a model-provider version bump: (a) add more infrastructure alerts on p95 latency and error rate, or (b) add online quality monitoring (LLM-judge scoring on a live sample) plus input/output drift detection. Which addresses the root cause, and why?

   <details><summary>Answer</summary>

   Option (b). A silent quality regression produces no exceptions and no latency change, so infrastructure alerts (a) will stay green while users get worse answers — they cannot detect it by construction. Online quality monitoring scores live outputs and drift detection watches for shifts in output distribution (e.g. rising refusal rate or falling judge score), which is what actually surfaces a model-version quality drop. This is why you classify the symptom as quality vs infrastructure before choosing instrumentation, and route flagged runs into the offline eval set (`03-evaluation-and-quality-assurance-patterns.md`).

   </details>

---

## Further Reading

- [OpenTelemetry — GenAI Semantic Conventions (repository)](https://github.com/open-telemetry/semantic-conventions-genai) — *verified 2026-07-29* — the canonical home (moved from the main semconv site) for GenAI spans, metrics, and events conventions covering clients, MCP, and provider-specific attributes.
- [OpenTelemetry — GenAI client spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md) — *verified 2026-07-29* — span definitions and attributes for inference/embeddings/retrieval/tool spans, including `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.usage.cache_read.input_tokens`, and `gen_ai.response.finish_reasons`.
- [LangSmith — Observability](https://docs.langchain.com/langsmith/observability) — *verified 2026-07-29* — instrumenting an app, investigating traces, dashboards/alerts for production monitoring, online evaluations via rules, and user-feedback collection.
- [LangSmith — Observability concepts](https://docs.langchain.com/langsmith/observability-concepts) — *verified 2026-07-29* — how traces/runs/threads/projects, feedback, tags, metadata, sampling, and data retention (400-day SaaS default) are modeled.
- [OpenAI — Production best practices](https://platform.openai.com/docs/guides/production-best-practices) — *verified 2026-07-29* — usage/token accounting, cost-as-tokens×cost-per-token framing, spend alerts and hard spend limits, and model-monitoring guidance for production.
- [Anthropic — Messages API reference](https://docs.anthropic.com/en/api/messages) — *verified 2026-07-29* — the `usage` fields (input/output and cache-read/creation token counts) and `stop_reason` you record per LLM span for token, cost, and finish-reason telemetry.
