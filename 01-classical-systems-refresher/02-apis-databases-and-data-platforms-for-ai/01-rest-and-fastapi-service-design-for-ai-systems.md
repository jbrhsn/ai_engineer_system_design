# REST and FastAPI Service Design for AI Systems

**Section:** Classical Systems Refresher → APIs, Databases, and Data Platforms for AI | **Est. time:** 2 hrs | **Interview relevance:** High — the inference service is the seam between your model and every client; interviewers probe how you keep it fast, safe, and observable under concurrent LLM traffic.

---

## TL;DR

An AI inference service is a thin, resilient HTTP layer wrapped around a slow, expensive, failure-prone upstream (the model). FastAPI's job is to validate every request with Pydantic, hold thousands of connections open cheaply with `async`/ASGI while each waits on a model provider, stream tokens back over Server-Sent Events, and inject a shared model client so it is loaded once, not per request. The hard parts are not the routes — they are timeouts, retries, rate limits, backpressure, and readiness probes that keep a GPU-bound backend from taking the whole service down. **The one thing to remember: in an AI service the event loop is your scarcest resource — never let a blocking call sit on it while a model generates.**

---

## ELI5 — Explain It Like I'm 5

Imagine a busy restaurant with one waiter (the event loop) and a slow kitchen (the LLM). A bad waiter takes one order, then stands frozen at the kitchen window until that dish is cooked before serving anyone else — the whole restaurant grinds to a halt even though the waiter is doing nothing but waiting. A good waiter takes your order, hands the ticket to the kitchen, and immediately goes to take the next table's order; when a dish is ready the kitchen rings a bell and the waiter delivers it. FastAPI's `async` routes make the waiter behave the good way: while one request "awaits" the model, the same waiter serves hundreds of others. The common misconception is that `async` makes the model itself faster — it does not; the kitchen cooks at the same speed. It only stops one slow dish from blocking every other diner, which is exactly what you need when every request waits seconds on a model.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Design REST resource paths and Pydantic request/response schemas for an LLM inference endpoint that validate input and shape output.
- [ ] Explain how FastAPI's ASGI event loop handles `async def` vs `def` routes and why blocking calls in an async route are catastrophic for a model-serving API.
- [ ] Implement a token-streaming endpoint using Server-Sent Events with a correct completion sentinel and client-disconnect handling.
- [ ] Inject a shared model client with dependency injection and load it once via a lifespan context manager.
- [ ] Compare timeout, retry, and rate-limit strategies for an unreliable upstream model provider and select defaults for a given latency budget.

---

## Visual Overview

### Request Lifecycle Through a FastAPI Inference Service

```
Client
  │  POST /v1/chat/completions  {messages, stream:true}
  ▼
┌─────────────────────────── ASGI server (Uvicorn) ───────────────────────────┐
│  Middleware stack (top → bottom)                                             │
│   ServerError ──► CORS ──► RateLimit ──► RequestID/Timing ──► Exception      │
│        │                                                                     │
│        ▼   Routing matches path + method                                    │
│   Pydantic validates body ──► 422 if invalid (never reaches model)          │
│        │                                                                     │
│        ▼   Depends() injects shared model client (loaded at lifespan)       │
│   Route handler (async) ── await model_client.stream(...) ──► upstream LLM   │
│        │                                                                     │
│        ▼   StreamingResponse / EventSourceResponse yields tokens            │
└──────────────────────────────────────────────────────────────────────────────┘
  │  text/event-stream:  data: {"token":"Hello"} … data: [DONE]
  ▼
Client renders tokens as they arrive
```

### Blocking vs Non-Blocking Handling Under Concurrency

```
Anti-pattern: blocking sync call inside async def
  Loop ─► req A ─► [BLOCKED waiting on model 4s] ══════════► done
                    req B,C,D… all queued, loop frozen, latency explodes

Correct: await an async client (or offload to threadpool)
  Loop ─► req A ─► await… ┐
       ─► req B ─► await… │  loop serves all while each awaits
       ─► req C ─► await… ┘  → high concurrency on one worker
```

### SSE Token-Streaming Flow

```
Route (async gen)          EventSourceResponse            Client (EventSource)
  yield token "Hel"  ──►  data: "Hel"\n\n            ──►  onmessage → append
  yield token "lo"   ──►  data: "lo"\n\n             ──►  onmessage → append
  (every 15s idle)   ──►  : keep-alive comment       ──►  (ignored, holds conn)
  yield [DONE]       ──►  data: [DONE]\n\n            ──►  close stream
  client closes tab  ──►  http.disconnect ──► cancel generator ──► free upstream
```

---

## Key Concepts

### REST Resource Design for Inference Endpoints

**What is it?** A discipline for naming and structuring the HTTP surface of a model service so clients can predict behaviour: resource-oriented paths, correct verbs, and versioned, mode-aware responses.

**How does it work under the hood?** FastAPI matches an incoming `(method, path)` against registered *path operations* and dispatches to the handler; the decorator (`@app.post("/v1/chat/completions")`) is what registers the route and its OpenAPI schema. Inference services are mostly "action on a resource" rather than pure CRUD, so a `POST` that returns a generated completion is idiomatic — `POST` because the request body is large and non-idempotent (the model may return different text each call), and it must not be cached by intermediaries. Versioning in the path (`/v1/`) lets you evolve schemas without breaking deployed clients.

**Where does it appear in real systems?** OpenAI's `POST /v1/chat/completions`, Anthropic's `POST /v1/messages`, and most self-hosted vLLM/TGI deployments all expose a versioned `POST` with a `stream: bool` field that switches between a single JSON response and a `text/event-stream`. The same route path serves both modes; the handler branches on the flag.

### Pydantic Request/Response Validation

**What is it?** Pydantic `BaseModel` classes declare the expected shape and types of the request body and the response; FastAPI uses them to parse, validate, document (OpenAPI), and serialize.

**How does it work under the hood?** When a request arrives, FastAPI reads the JSON body and calls Pydantic, which instantiates the model — parsing, coercing (`"123"` → `123`), and validating against the field types and constraints in Rust-backed `pydantic-core`. If any field fails, Pydantic raises `ValidationError` and FastAPI returns `422 Unprocessable Entity` with a per-field error list, *before your handler runs* — so the model is never invoked on garbage input. On the way out, declaring a return type or `response_model` lets Pydantic serialize directly (fastest path) and strip fields not in the schema.

**Where does it appear in real systems?** A `ChatRequest(messages: list[Message], max_tokens: int = Field(gt=0, le=4096), temperature: float = Field(ge=0, le=2))` bounds cost and blocks prompt-injection-via-oversized-input at the edge. `Field(le=4096)` is a concrete guardrail: it stops a client from requesting a 1M-token generation that would exhaust the GPU. `model_validate_json()` is the method FastAPI uses internally to validate an incoming JSON payload.

### Async/Await and the ASGI Event Loop

**What is it?** ASGI is the async server interface FastAPI (via Starlette) runs on; `async def` routes are coroutines that can pause at `await` points and let the single-threaded event loop serve other requests meanwhile.

**How does it work under the hood?** FastAPI runs each `async def` *path operation* directly on the event loop. At an `await` (e.g. `await client.post(...)` to a model provider), the coroutine suspends and the loop picks up other ready work; when the network I/O completes, the loop resumes it. Crucially, a `def` (non-async) route is run in an *external threadpool* so a blocking call inside it does **not** freeze the loop — but a blocking call inside an `async def` route *does*, because the loop cannot preempt CPU-bound or blocking-I/O code that never `await`s. For I/O-bound LLM calls (the common case), `async def` + an async HTTP client gives the highest concurrency per worker.

**Where does it appear in real systems?** One Uvicorn worker with an `async` route calling `httpx.AsyncClient` can hold thousands of concurrent open connections to an upstream model while each awaits generation — the same pattern that makes Node.js/Go efficient. If instead you called the blocking `openai` SDK synchronously inside `async def`, throughput collapses to roughly one request at a time per worker.

### Streaming Responses (SSE) for Token Streaming

**What is it?** Server-Sent Events is a one-way `text/event-stream` protocol where the server pushes a sequence of small text events over a single long-lived HTTP response — ideal for streaming LLM tokens so users see output as it is generated.

**How does it work under the hood?** The handler is a generator (`yield`) instead of a `return`. FastAPI wraps it in a streaming response that sends each chunk as an ASGI `http.response.body` message with `more_body=True`, flushing to the client immediately rather than buffering the whole answer. FastAPI 0.135.0 added a native `fastapi.sse` module: setting `response_class=EventSourceResponse` and `yield`ing data (or `ServerSentEvent` objects) encodes each item into the SSE wire format, sends a keep-alive `:` comment every 15s to defeat proxy timeouts, and sets `Cache-Control: no-cache` and `X-Accel-Buffering: no` automatically. *(Note: `fastapi.sse` is new as of 0.135.0 — verify your installed version; older code uses `StreamingResponse` with manually formatted `data:` lines.)*

**Where does it appear in real systems?** Every streaming chat UI. The conventional end marker is a sentinel `data: [DONE]` (sent with `raw_data` so it is not JSON-quoted). On `http.disconnect` (user closes the tab), the async generator is cancelled at its next `await`, which lets you stop and release the expensive upstream generation instead of paying for tokens no one will read.

### Dependency Injection for Model Clients

**What is it?** FastAPI's `Depends()` system lets a route declare what it needs (a model client, a rate limiter, an authenticated user) and have FastAPI construct and supply it, without the route creating it directly.

**How does it work under the hood?** You annotate a parameter as `Annotated[ModelClient, Depends(get_model_client)]`; on each request FastAPI resolves the dependency (calling the provider, caching within the request, honouring sub-dependencies) and injects the result. A dependency declared with `def` runs in the threadpool; with `async def` it is awaited. Combined with a **lifespan** context manager, the heavy object (a loaded model or a pooled HTTP client) is created **once at startup** and shared across all requests, not rebuilt per call.

**Where does it appear in real systems?** `lifespan` loads a local transformer model or opens a pooled `httpx.AsyncClient` to a provider before the app accepts traffic, and closes it on shutdown. Tests override the dependency (`app.dependency_overrides`) to inject a fake client — no network, deterministic. This is the seam that makes an inference service testable.

### Middleware for Cross-Cutting Concerns

**What is it?** Middleware wraps every request/response to apply behaviour that is not route-specific: CORS, request IDs, timing/metrics, auth, and rate limiting.

**How does it work under the hood?** Middleware forms an onion evaluated top-to-bottom on the way in and bottom-to-up on the way out; each layer can inspect/modify the request `scope`, short-circuit with a response, or wrap the downstream `send`. Starlette's `BaseHTTPMiddleware` gives a friendly `async def dispatch(request, call_next)` interface, but it has a known limitation: it breaks `contextvars` propagation from endpoint back up the stack, so for request-ID/tracing you often want **pure ASGI middleware** (`async def __call__(self, scope, receive, send)`). Middleware must be **stateless** — per-connection state belongs inside `__call__`/`dispatch`, never on `self`, or concurrent requests corrupt each other.

**Where does it appear in real systems?** A `RequestID` middleware stamps every log line so a single slow generation is traceable; a rate-limit middleware rejects with `429` before a request ever reaches the GPU. Note streaming: `GZipMiddleware` explicitly skips `text/event-stream` so it does not buffer and break your token stream.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Uvicorn `--workers` | Number of OS processes (each its own event loop) | Set to CPU count for CPU-bound local inference; keep low (1–2) and rely on async concurrency when the model is a remote provider (I/O-bound). Never use workers to "add concurrency" for async I/O — that is what the loop already does. |
| Upstream request `timeout` | Max seconds to wait on the model provider before aborting | Set to just above your p99 generation time (e.g. 30–60s for long completions); too low kills valid long generations, too high lets hung upstreams pile up connections. Set **connect** and **read** timeouts separately. |
| Max retries + backoff | Retries on transient upstream errors (`429`, `5xx`, timeouts) | Retry only idempotent-safe transient failures with exponential backoff + jitter, cap at 2–3; never retry a `400`/`422` (deterministic failure) or a non-idempotent streamed generation mid-stream. |
| Rate limit (req/s per key) | Requests admitted before `429` | Set to protect GPU throughput, not client convenience — derive from tokens/sec the backend can sustain, and return `Retry-After`. |
| Request body size limit | Max prompt/payload bytes accepted | Cap below the model's context window in bytes; enforce at middleware/proxy so oversized bodies are rejected before allocation. |
| `max_tokens` (Pydantic `Field(le=...)`) | Upper bound on generated tokens per request | Bound to control latency and cost per call; set the ceiling to your budget, let clients request less. |

### Worked Example: Requirement → Decision

**Given:** Your team must expose a chat endpoint that streams tokens to a web UI. The model is served behind a remote provider that averages 800ms to first token and 6s to full completion; the UI must feel responsive, and a single misbehaving client must not degrade others.

- **Step 1 — Identify the goal:** A low-perceived-latency chat endpoint that streams tokens, validates input, survives upstream flakiness, and isolates noisy clients.
- **Step 2 — Define inputs:** `POST /v1/chat/completions` with a Pydantic `ChatRequest{messages: list[Message], max_tokens: int = Field(gt=0, le=2048), temperature: float = Field(ge=0, le=2), stream: bool = True}`; an API key header used by the rate-limit middleware.
- **Step 3 — Define outputs:** When `stream=true`, a `text/event-stream` where each event is `data: {"delta": "<token>"}` ending with `data: [DONE]`; when `stream=false`, a single JSON `ChatResponse`. Errors: `422` for bad input, `429` with `Retry-After` when rate-limited, `504` on upstream timeout.
- **Step 4 — Apply constraints:** Latency budget — first token must reach the client fast, so buffering the full 6s answer is unacceptable (rules out a plain JSON `return`). Concurrency — many users wait simultaneously on an I/O-bound provider. Cost — a client that disconnects should stop the upstream generation. Isolation — one client cannot starve the GPU.
- **Step 5 — Select the approach:** Use an **`async def` route** (concurrency on one worker for the I/O-bound call) returning an **`EventSourceResponse`** that `yield`s tokens as they arrive (streaming beats first-byte latency), with the shared async model client provided by **`Depends` + lifespan** (loaded once), an **async HTTP client with per-request read timeout + capped retries** on the upstream, and a **rate-limit middleware** returning `429` before the GPU. Rationale vs alternatives: a synchronous `def` route in a threadpool would work but caps per-worker concurrency at the thread count; a WebSocket is overkill for one-way token push and complicates proxies; returning full JSON fails the first-token latency budget.

---

## Implementation

```python
# Scenario: Serve an LLM chat endpoint that must (1) reject invalid/oversized
# requests before touching the GPU, and (2) load the expensive model client
# exactly once so it is shared across all requests, not rebuilt per call.
from contextlib import asynccontextmanager
from typing import Annotated

import httpx
from fastapi import Depends, FastAPI
from pydantic import BaseModel, Field


class Message(BaseModel):
    role: str
    content: str


class ChatRequest(BaseModel):
    messages: list[Message]
    # Field bounds are edge guardrails: they cap cost/latency and block a
    # client from requesting an unbounded generation. Invalid input -> 422,
    # and the model is never called.
    max_tokens: int = Field(default=512, gt=0, le=2048)
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    stream: bool = True


# Loaded once at startup, closed at shutdown — shared connection pool to the
# upstream provider. This is the "load the ML model before serving" pattern.
clients: dict[str, httpx.AsyncClient] = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    clients["model"] = httpx.AsyncClient(
        base_url="https://provider.example/v1",
        timeout=httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=5.0),
    )
    yield
    await clients["model"].aclose()


app = FastAPI(lifespan=lifespan)


def get_model_client() -> httpx.AsyncClient:
    return clients["model"]


ModelClient = Annotated[httpx.AsyncClient, Depends(get_model_client)]


@app.post("/v1/chat/completions")
async def chat(req: ChatRequest, client: ModelClient) -> dict:
    # async def + async client => one worker serves many concurrent waits.
    resp = await client.post("/generate", json=req.model_dump())
    resp.raise_for_status()
    return resp.json()
```

```python
# Anti-pattern: a BLOCKING synchronous SDK call inside an async def route.
# The event loop cannot preempt code that never awaits, so this one request
# freezes the single-threaded loop for the entire multi-second generation —
# every other concurrent request stalls behind it. Throughput collapses to
# ~one request at a time per worker, the exact opposite of what async buys you.
import time
from openai import OpenAI  # synchronous client

sync_client = OpenAI()


@app.post("/v1/bad-chat")
async def bad_chat(req: ChatRequest):
    # Blocking network call (no await) sitting on the event loop:
    result = sync_client.chat.completions.create(  # blocks the loop for seconds
        model="gpt-x", messages=[m.model_dump() for m in req.messages]
    )
    return result


# Correct approach: stream via SSE with an async generator, a shared async
# client, and a [DONE] sentinel. Tokens flush immediately (low first-token
# latency); on client disconnect the generator is cancelled at its next await,
# stopping the upstream generation so you stop paying for unread tokens.
from collections.abc import AsyncIterable

from fastapi.sse import EventSourceResponse, ServerSentEvent  # FastAPI >= 0.135.0


@app.post("/v1/chat/completions/stream", response_class=EventSourceResponse)
async def chat_stream(req: ChatRequest, client: ModelClient) -> AsyncIterable[ServerSentEvent]:
    async with client.stream("POST", "/generate", json=req.model_dump()) as upstream:
        async for token in upstream.aiter_text():   # await point -> loop stays free
            yield ServerSentEvent(data={"delta": token}, event="token")
    # raw_data avoids JSON-quoting the sentinel: emits `data: [DONE]`
    yield ServerSentEvent(raw_data="[DONE]", event="done")
```

---

## Common Pitfalls & Misconceptions

- **Blocking the event loop inside `async def`** — Beginners assume marking a route `async def` magically makes any code inside it concurrent, so they call a synchronous SDK or a blocking DB driver directly. The correct model: `async` only helps at `await` points; a blocking call with no `await` freezes the single loop for its whole duration — use an async client, or put the blocking call in a plain `def` route (threadpool) or `run_in_threadpool`.
- **Buffering a streaming response** — Beginners return the full completion as JSON "for simplicity," then wonder why the UI feels frozen for 6 seconds. The correct model: stream with `yield`/SSE so first token latency, not total latency, is what the user perceives; and remember GZip/compression middleware must skip `text/event-stream` or it re-buffers your stream.
- **Rebuilding the model client per request** — Beginners instantiate the client/model inside the handler because it "just works" locally, adding seconds and exhausting connections under load. The correct model: build it once in a `lifespan` context manager and inject it with `Depends`, so it is shared and closed cleanly on shutdown.
- **Retrying non-idempotent or deterministic failures** — Beginners wrap every upstream call in blanket retries, so a `422` (bad request) is retried three times and a mid-stream failure double-charges tokens. The correct model: retry only transient, idempotent-safe errors (`429`, `5xx`, timeouts) with backoff + jitter and a small cap; surface `4xx` immediately.
- **Confusing liveness with readiness** — Beginners point Kubernetes' readiness probe at a route that returns `200` as soon as the process starts, so traffic arrives before the model is loaded and every early request errors. The correct model: readiness must reflect that the model/client is actually loaded (flip a flag at the end of `lifespan` startup); liveness only checks the process is alive.

---

## Key Definitions

| Term | Definition |
|---|---|
| ASGI | The asynchronous server-gateway interface FastAPI runs on; defines `scope`/`receive`/`send` and supports long-lived connections and streaming, unlike synchronous WSGI. |
| Path operation | A FastAPI route: the function plus its decorator binding a `(method, path)`; the unit FastAPI validates, injects into, and documents. |
| Pydantic `BaseModel` | A typed schema class FastAPI uses to parse, validate, document, and serialize request bodies and responses; failure yields `422`. |
| Server-Sent Events (SSE) | A one-way `text/event-stream` protocol pushing a sequence of small text events over one HTTP response; used for token streaming. |
| `EventSourceResponse` | FastAPI 0.135.0+ response class (`fastapi.sse`) that encodes yielded data into SSE wire format with keep-alive pings and anti-buffering headers. |
| Dependency injection (`Depends`) | FastAPI's mechanism to declare and auto-supply a route's requirements (e.g. a shared model client) per request. |
| Lifespan | An async context manager passed to `FastAPI(lifespan=...)` running startup code (load model) before serving and cleanup after. |
| Readiness probe | A health check reporting whether the instance can serve traffic *now* (model loaded), distinct from liveness (process alive). |

---

## Summary / Quick Recall

- An inference service is a thin resilient HTTP layer over a slow, expensive upstream — the design work is validation, streaming, concurrency, and resilience, not the routes.
- Pydantic validates and bounds every request at the edge (`Field(le=...)`) so the model is never invoked on invalid or oversized input.
- `async def` + an async client gives huge per-worker concurrency for I/O-bound model calls; a blocking call inside `async def` freezes the whole event loop.
- Stream tokens with SSE (`EventSourceResponse`, `[DONE]` sentinel) to optimise first-token latency and cancel upstream work on client disconnect.
- Load the model/client once in `lifespan` and inject it with `Depends`; override it in tests for deterministic, network-free testing.
- Add timeouts, capped-with-backoff retries on transient errors only, and rate-limit middleware returning `429` before the GPU; separate readiness (model loaded) from liveness.

---

## Self-Check Questions

1. In FastAPI, what happens to a `def` (non-async) path operation function versus an `async def` one when it contains a blocking call?

   <details><summary>Answer</summary>

   A `def` path operation is run in an **external threadpool**, so a blocking call inside it does not freeze the event loop — other requests continue on the loop. An `async def` path operation runs **directly on the single event loop**, so a blocking call with no `await` freezes the loop for its entire duration, stalling every other concurrent request. The tempting wrong answer is "they behave the same because FastAPI is always async" — FastAPI *is* async overall, but where the function runs (threadpool vs loop) is exactly what differs and determines whether blocking is safe.

   </details>

2. You expose `POST /v1/chat/completions` and a client sends `max_tokens: -5` against a schema declaring `max_tokens: int = Field(gt=0, le=2048)`. What does the client receive and when does the model get called?

   <details><summary>Answer</summary>

   The client receives a `422 Unprocessable Entity` with a per-field error, and the model is **never called** — Pydantic validation runs before your handler executes, so invalid input is rejected at the edge. This is why field bounds are a cost/latency guardrail. The tempting wrong answer is that the handler runs and you must check `max_tokens` manually; FastAPI/Pydantic already reject it and return `422` for you.

   </details>

3. **Which TWO** of the following are correct practices when building an SSE token-streaming endpoint for an LLM?
   - A. Return the full completion as a single JSON object for simplicity.
   - B. Send a terminal sentinel such as `data: [DONE]` so the client knows the stream ended.
   - C. Enable GZip compression on the `text/event-stream` response to save bandwidth.
   - D. Handle client disconnect so the async generator is cancelled and upstream generation stops.
   - E. Instantiate a fresh model client inside the generator on every request.

   <details><summary>Answer</summary>

   **B and D.** B is correct because clients need an explicit end marker; the conventional `[DONE]` sentinel signals completion (sent as `raw_data` so it is not JSON-quoted). D is correct because on `http.disconnect` the generator is cancelled at its next `await`, letting you stop paying for tokens no one will read. A is wrong — it defeats the entire purpose of streaming and hurts first-token latency. C is the most tempting wrong answer: GZip must **not** be applied to `text/event-stream` (Starlette's `GZipMiddleware` explicitly skips it) because buffering breaks the stream. E is wrong — the client should be loaded once via lifespan and injected, never rebuilt per request.

   </details>

4. Your service calls a remote model provider. Design your timeout and retry policy for these upstream errors: `422 Bad Request`, `429 Too Many Requests`, a read timeout, and `503 Service Unavailable`. Which do you retry and why?

   <details><summary>Answer</summary>

   Retry `429`, the read timeout, and `503` — all transient and safe to retry — using exponential backoff with jitter, capped at 2–3 attempts, honouring any `Retry-After` on the `429`. Do **not** retry the `422`: it is a deterministic client error that will fail identically every time, so retrying wastes latency and load. Set connect and read timeouts separately (short connect, generous read tuned just above p99 generation). The trade-off: too-aggressive retries amplify load during an upstream incident (retry storm), so cap attempts and add jitter; too-low read timeouts kill valid long generations.

   </details>

5. A teammate proposes scaling concurrency for your remote-provider-backed inference API by raising Uvicorn `--workers` to 32. Evaluate this against relying on `async` concurrency, and state when each is right.

   <details><summary>Answer</summary>

   For an **I/O-bound** remote-provider call, `async` concurrency on a single loop already lets one worker hold thousands of connections while each awaits — so jumping to 32 workers mostly multiplies memory (32 model clients/pools) and process overhead without proportional throughput gain; 1–2 workers plus async is usually right. Workers (parallel processes) are the correct lever for **CPU-bound** work — e.g. local model inference or heavy pre/post-processing that actually burns CPU and cannot be interleaved on one loop — where you want roughly one worker per core. The nuanced trade-off: many teams run a small worker count for redundancy and to use multiple cores, but the key insight is that workers add *parallelism* (for CPU-bound work) while async adds *concurrency* (for I/O-bound waiting); choosing the wrong one wastes resources or caps throughput.

   </details>

---

## Further Reading

- [Concurrency and async / await — FastAPI](https://fastapi.tiangolo.com/async/) — *verified 2026-07-28* — how `async def` vs `def` routes run on the event loop vs threadpool, and why it matters for ML/web workloads.
- [Server-Sent Events (SSE) — FastAPI](https://fastapi.tiangolo.com/tutorial/server-sent-events/) — *verified 2026-07-28* — streaming tokens with `EventSourceResponse`/`ServerSentEvent`, the `[DONE]` sentinel, and built-in keep-alive/anti-buffering behaviour.
- [Server-Sent Events — `EventSourceResponse` and `ServerSentEvent` (Reference) — FastAPI](https://fastapi.tiangolo.com/reference/sse/) — *verified 2026-07-28* — API reference for the SSE response class and event fields (`data`, `raw_data`, `event`, `id`, `retry`, `comment`).
- [Custom Response - HTML, Stream, File, others — FastAPI](https://fastapi.tiangolo.com/advanced/custom-response/) — *verified 2026-07-28* — `StreamingResponse` and response-class mechanics for non-JSON and streamed responses.
- [Dependencies — FastAPI](https://fastapi.tiangolo.com/tutorial/dependencies/) — *verified 2026-07-28* — the `Depends`/`Annotated` dependency-injection system for supplying shared clients to routes.
- [Lifespan Events — FastAPI](https://fastapi.tiangolo.com/advanced/events/) — *verified 2026-07-28* — loading a shared ML model/client once at startup with an async context manager and cleaning up on shutdown.
- [Models — Pydantic](https://docs.pydantic.dev/latest/concepts/models/) — *verified 2026-07-28* — defining `BaseModel` request/response schemas, validation behaviour, and `model_validate`/`model_dump`.
- [Middleware — Starlette](https://www.starlette.dev/middleware/) — *verified 2026-07-28* — `BaseHTTPMiddleware` vs pure ASGI middleware, statelessness requirements, and the `contextvars` limitation relevant to request tracing.
