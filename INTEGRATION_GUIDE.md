# Conviva Agent Tracing — Integration Guide for Python

Conviva Agent Tracing gives you end-to-end visibility into your AI agents. Add
one line to your application and every LLM call, tool call, retrieval and
agent-framework step is traced and sent to Conviva — grouped by conversation,
and joined to the end-user session your Conviva client sensor measures.

It is built on OpenTelemetry, so it works alongside any tracing you already
have. This guide covers Python; a Node.js edition is available as
`@convivainc/agent-tracing` on npm.

## Contents

1. [Integration options](#1-integration-options)
2. [Quick start — option 1, full SDK](#2-quick-start--option-1-full-sdk)
3. [Supported integrations](#3-supported-integrations)
4. [Tracing workflows, tasks, agents and tools](#4-tracing-workflows-tasks-agents-and-tools)
5. [Conversations and association properties](#5-conversations-and-association-properties)
6. [Connecting the Conviva client sensor — option 3](#6-connecting-the-conviva-client-sensor--option-3)
7. [Manual LLM instrumentation](#7-manual-llm-instrumentation)
8. [Privacy: prompts and completions](#8-privacy-prompts-and-completions)
9. [Exporter only — option 2](#9-exporter-only--option-2)
10. [Configuration reference](#10-configuration-reference)
11. [API reference](#11-api-reference)
12. [Verifying your integration](#12-verifying-your-integration)
13. [Limitations](#13-limitations)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Integration options

There are three ways to integrate. Pick **one** of the first two, and add the
third if your web or mobile app runs the Conviva client sensor.

| | Option | Use it when | You install |
|---|---|---|---|
| **1** | **Full SDK** | You want traces with the least work, or your agent has no tracing yet. | `conviva-agent-tracing` |
| **2** | **Exporter only** | Your agent already uses an OpenTelemetry-based tracing library (Langfuse, OpenInference, Arize, your own spans…) and you want to keep it. | `conviva-otlp-exporter` |
| **3** | **Traces connector**, added to 1 or 2 | Your front end runs the Conviva client sensor, and you want each agent trace joined to the user's session in Conviva. | Nothing more with option 1; `conviva-agent-traces-connector` with option 2 |

```text
Does your agent already create OpenTelemetry spans you want to keep?
├── No  → Option 1: Full SDK
└── Yes → Option 2: Exporter only

Does your web or mobile app run the Conviva client sensor?
└── Yes → also add Option 3: Traces connector
```

### Option 1 — Full SDK

One `init()` call instruments every supported LLM SDK, vector database and
agent framework automatically (§3), and sends the traces to Conviva. It
includes the exporter and the traces connector, so option 3 needs no extra
package.

```python
import conviva_agent_tracing
conviva_agent_tracing.init(app_name="travel-assistant")
```

Full walkthrough: §2. Annotating your own workflows, tools and conversations:
§4 – §5.

### Option 2 — Exporter only

Keeps your existing instrumentation and changes only where the spans go. You
add the Conviva exporter and span processors to the OpenTelemetry tracer
provider you already have; nothing else in your code changes.

```python
for processor in create_conviva_span_processors(config):
    provider.add_span_processor(processor)
provider.add_span_processor(BatchSpanProcessor(create_trace_exporter(config)))
```

Full walkthrough: §9.

### Option 3 — Traces connector (with option 1 or 2)

The Conviva client sensor sends three ids with every request to your backend:
the client, the conversation and the message. The traces connector copies them
onto every span of the agent work that request causes, so Conviva can join the
agent trace to the user's session, and uses the conversation id to group turns
when your code does not set one.

| Combined with | What you do |
|---|---|
| Option 1 | Nothing to install — the connector is part of the SDK. Make sure the client sensor's header reaches your agent (§6). |
| Option 2 | Install the connector and add its span processor ahead of the Conviva processors (§9), then the same wiring (§6). |

```python
# Option 2 + 3
provider.add_span_processor(BaggageSpanProcessor())   # the traces connector, first
for processor in create_conviva_span_processors(config):
    provider.add_span_processor(processor)
provider.add_span_processor(BatchSpanProcessor(create_trace_exporter(config)))
```

Full walkthrough: §6.

### What each option gives you

| | Option 1 | Option 2 | Option 3 (added) |
|---|:---:|:---:|:---:|
| Automatic tracing of LLM SDKs, vector DBs and frameworks | ✅ | from your own library | — |
| Workflow, task, agent and tool annotations (§4) | ✅ | from your own library | — |
| Export to Conviva, routed to your project | ✅ | ✅ | — |
| Conversation id on every span | ✅ | ✅ when your library writes one | ✅ from the client sensor |
| Client, conversation and message ids from the client sensor | ✅ with the wiring in §6 | — | ✅ |
| Content capture switch (§8) | ✅ | set in your own library | — |

---

## 2. Quick start — option 1, full SDK

The fastest path: the full SDK with automatic instrumentation. Using
option 2 instead? See §9.

### Prerequisites

| | |
|---|---|
| **Customer key** | Identifies your Conviva account. Traces go to `https://<customer-key>.agw.conviva.com/v1/traces`. |
| **Project id** *(optional)* | Files the traces under a specific project. Without one, they go to your account's default project. |
| **Runtime** | Python 3.10 or later (3.9 or later for the exporter and connector alone). |

Your Conviva contact provides these values. Keep them in your secret store or
environment, not in source code.

### Install the SDK

```bash
pip install conviva-agent-tracing
```

### Configure

```bash
export CONVIVA_CUSTOMER_KEY=<customer-key>
export CONVIVA_PROJECT_ID=<project-id>   # optional
```

### Initialize

Call `init()` once, at startup, **before** you import the LLM SDKs and
frameworks you want traced. Libraries are instrumented as they are loaded, so
anything imported earlier is not traced.

```python
import conviva_agent_tracing

conviva_agent_tracing.init(app_name="travel-assistant")

from openai import OpenAI  # instrumented libraries after init()
```

That's it. Every supported LLM, vector-database and framework call is now
traced and exported to Conviva. Long-running services need nothing else;
scripts and serverless functions should [flush before exit](#flush-before-exit).

---

## 3. Supported integrations

These libraries are instrumented automatically after `init()` — no code
changes. Each model call records the model, request parameters, prompts and
completions, token usage, and finish reasons.

### LLM providers

| Provider | | Provider | |
|---|:---:|---|:---:|
| OpenAI and Azure OpenAI (including OpenAI-compatible endpoints) | ✅ | Groq | ✅ |
| Anthropic | ✅ | Ollama | ✅ |
| AWS Bedrock | ✅ | Replicate | ✅ |
| AWS SageMaker | ✅ | IBM watsonx | ✅ |
| Google Vertex AI | ✅ | Aleph Alpha | ✅ |
| Google Gemini (Gen AI SDK) | ✅ | Writer | ✅ |
| Cohere | ✅ | Voyage AI | ✅ |
| Together AI | ✅ | Hugging Face Transformers | ✅ |
| Mistral AI | ✅ | LiteLLM | ✅ |

### Vector databases

| Database | | Database | |
|---|:---:|---|:---:|
| Pinecone | ✅ | Milvus | ✅ |
| Chroma | ✅ | LanceDB | ✅ |
| Qdrant | ✅ | Marqo | ✅ |
| Weaviate | ✅ | | |

### Agent frameworks

| Framework | | Framework | |
|---|:---:|---|:---:|
| LangChain and LangGraph | ✅ | Haystack | ✅ |
| LlamaIndex | ✅ | OpenAI Agents SDK | ✅ |
| CrewAI | ✅ | Agno | ✅ |

### Protocols

| Protocol | |
|---|:---:|
| Model Context Protocol (MCP) | ✅ |

`requests`, `urllib3`, Redis and PyMySQL are also instrumented when they are
installed, so outbound HTTP and database calls appear in the same trace.

Something missing? Any library that emits OpenTelemetry spans is exported as
well, and you can trace your own steps with the APIs in §4 and §7.

### Choosing what is instrumented

Allow-list or block-list instruments:

```python
from conviva_agent_tracing import Instruments, init

init(app_name="travel-assistant", instruments={Instruments.OPENAI, Instruments.LANGCHAIN})
# or
init(app_name="travel-assistant", block_instruments={Instruments.REQUESTS, Instruments.URLLIB3})
```

---

## 4. Tracing workflows, tasks, agents and tools

Auto-instrumentation records each model call on its own. To see the structure
of your agent — which user request a call belongs to, which tool ran, which
sub-agent handled it — annotate your own code. Spans nest automatically.

| Kind | Use it for |
|---|---|
| **workflow** | One end-to-end operation, usually a user turn |
| **task** | A step inside a workflow (retrieval, parsing, a chain) |
| **agent** | An autonomous component that decides what to do |
| **tool** | A function an agent or model calls |

The function's inputs and outputs are recorded on the span (see §8 to turn
that off).

```python
from conviva_agent_tracing import agent, task, tool, workflow

@tool(name="search_flights")
def search_flights(origin: str, destination: str) -> list[dict]:
    return inventory.search(origin, destination)

@task(name="rank_results")
def rank(flights: list[dict]) -> list[dict]:
    return sorted(flights, key=lambda f: f["price"])

@agent(name="booking_agent")
def booking_agent(request: str) -> str:
    flights = rank(search_flights("BLR", "GOI"))
    return llm_summarise(request, flights)

@workflow(name="chat_turn")
def chat_turn(message: str) -> str:
    return booking_agent(message)
```

For `async` functions use `aworkflow`, `atask`, `aagent` and `atool`. All four
accept `name` and `version`.

---

## 5. Conversations and association properties

Conviva groups traces into conversations so a multi-turn chat reads as one
story. Give each turn its conversation id in **one** of these ways.

### A — let the Conviva client sensor provide it (no code)

If your front end runs the Conviva client sensor, its conversation id arrives
with every request. Complete [§6](#6-connecting-the-conviva-client-sensor--option-3) and
nothing else is needed.

### B — set it in code

```python
from conviva_agent_tracing import set_association_properties, workflow

@workflow(name="chat_turn")
def chat_turn(message: str) -> str:
    return agent.run(message)

set_association_properties({"session_id": conversation_id})
reply = chat_turn(message)
```

For a fixed id, the `@conversation(conversation_id="...")` decorator does the
same.

An id you set in code wins over the client sensor's.

### Association properties

Attach your own identifiers — user, tenant, plan — to every span in a scope,
so you can filter on them in Conviva:

```python
set_association_properties({"user_id": user.id, "tenant": "acme"})
```

For values that are the same on every span — environment, region,
deployment — use the `attributes` option of `init()` instead.

---

## 6. Connecting the Conviva client sensor — option 3

With option 1, the traces connector is already installed; with option 2, add
it as shown in §9. Either way, the steps below are what get the client
sensor's ids to it.

This is what joins an end-user's session in Conviva to the agent work it
caused. With each request to your backend, the client sensor sends three ids
as a W3C `baggage` header:

```text
baggage: c3.client_id=<id>,c3.conv_id=<id>,c3.msg_id=<id>
```

The SDK copies `c3.client_id`, `c3.conv_id` and `c3.msg_id` onto every span,
and uses `c3.conv_id` as the conversation id when you have not set one. Three
things must hold for the ids to arrive.

**1. Your API accepts the header.** If the front end calls your API on another
origin, allow the header in CORS — otherwise the browser drops it silently:

```text
Access-Control-Allow-Headers: content-type, baggage
```

**2. The header is read into the trace context.** OpenTelemetry web-framework
instrumentation (`opentelemetry-instrumentation-fastapi`, `-flask`, `-django`,
…) does this. If your server has none, extract it per request:

```python
# FastAPI / Starlette
from opentelemetry import context
from opentelemetry.propagate import extract

@app.middleware("http")
async def conviva_baggage(request, call_next):
    token = context.attach(extract(dict(request.headers)))
    try:
        return await call_next(request)
    finally:
        context.detach(token)
```

**3. WebSockets carry it in the message.** Browsers cannot set headers on a
WebSocket. Send the same `baggage` value inside each message and extract it
around that message's handler:

```python
token = context.attach(extract({"baggage": message["baggage"]}))
try:
    await handle_message(message)
finally:
    context.detach(token)
```

Calls from your backend to other instrumented services forward the baggage, so
their spans carry the same ids.

To copy additional baggage keys onto spans:

```python
from conviva_agent_tracing import BaggageConfig, init

init(baggage_config=BaggageConfig(allowlist=("c3.", "tenant")))
```

An entry ending in `.` matches a prefix; any other entry matches exactly.

---

## 7. Manual LLM instrumentation

For a model client that is not in §3 — an in-house gateway, a raw HTTP call —
report the call yourself.

```python
from conviva_agent_tracing import LLMMessage, LLMUsage, track_llm_call

with track_llm_call(vendor="my-gateway", type="chat") as span:
    span.report_request(model="my-model", messages=[LLMMessage(role="user", content=prompt)])
    response = gateway.chat(model="my-model", prompt=prompt)
    span.report_response(response.model, [response.text])
    span.report_usage(LLMUsage(
        prompt_tokens=response.usage.input,
        completion_tokens=response.usage.output,
        total_tokens=response.usage.input + response.usage.output,
    ))
```

Messages you report this way are recorded as given, whatever the content
setting in §8. When content is off, report the model and usage only.

---

## 8. Privacy: prompts and completions

By default, prompts, completions, and the inputs and outputs of workflows,
tasks, agents and tools are recorded on spans. They are most of what makes an
agent trace useful — and they may contain personal data.

To keep content off spans, while still recording model, timing, token usage
and structure:

```bash
export CONVIVA_TRACE_CONTENT=false
```

or in code: `init(trace_content=False)`.

Other points:

- Only allow-listed baggage (`c3.*` by default) is copied onto spans.
- Your customer key is used only in the endpoint host name. It is never sent as
  a header or an attribute.
- There is no automatic PII masking. Redact sensitive values before they reach
  the model, or turn content off. See also [Limitations](#13-limitations).

---

## 9. Exporter only — option 2

Already tracing with an OpenTelemetry-based library — Langfuse, OpenInference,
Arize, or spans you create yourself? Keep it. Add the Conviva exporter and span
processors to the tracer provider you already have, and those spans go to
Conviva. `init()` is not used.

### Install

```bash
pip install conviva-otlp-exporter
```

Configure with the same environment variables as option 1
(`CONVIVA_CUSTOMER_KEY`, and optionally `CONVIVA_PROJECT_ID`).

### Wire it in

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

from conviva_otlp_exporter import create_conviva_span_processors, create_trace_exporter, resolve_config

config = resolve_config()

provider = TracerProvider()
for processor in create_conviva_span_processors(config):
    provider.add_span_processor(processor)
provider.add_span_processor(BatchSpanProcessor(create_trace_exporter(config)))
trace.set_tracer_provider(provider)
```

If your library builds its own provider and accepts only one span processor,
wrap the stack: `CompositeSpanProcessor(*processors)`.

The Conviva processors add your project id and the conversation id to every
span. The conversation id is read from what your library already writes —
`gen_ai.conversation.id` or `session.id` — or, with option 3, from the client
sensor.

### Adding the traces connector — option 2 + 3

Install the connector and put its span processor **first**, ahead of the
Conviva processors, so the client sensor's ids are on each span before the
conversation id is resolved.

```bash
pip install conviva-agent-traces-connector
```

```python
from conviva_agent_traces_connector import BaggageSpanProcessor

provider = TracerProvider()
provider.add_span_processor(BaggageSpanProcessor())
for processor in create_conviva_span_processors(config):
    provider.add_span_processor(processor)
provider.add_span_processor(BatchSpanProcessor(create_trace_exporter(config)))
trace.set_tracer_provider(provider)
```

Then make sure the client sensor's header reaches your agent: §6.

---

## 10. Configuration reference

### `init()` options

| Option | Default | Description |
|---|---|---|
| `app_name` | `OTEL_SERVICE_NAME`, else the process name | Service name on every span |
| `customer_key` | `CONVIVA_CUSTOMER_KEY` | Your customer key |
| `project_id` | `CONVIVA_PROJECT_ID` | Your project id. Optional: without it, traces go to your account's default project |
| `endpoint` | `https://<customer-key>.agw.conviva.com` | Send traces somewhere else, e.g. a local collector |
| `headers` | — | Extra export request headers |
| `attributes` | — | Attributes added to every span |
| `trace_content` | `True` | Record prompts, completions and payloads (§8) |
| `disable_batch` | `False` | Export each span as it ends; for scripts and debugging |
| `span_processors` | — | Your own span processors, run after Conviva's |
| `baggage_config` | `c3.*` | Which baggage entries are copied onto spans (§6) |
| `conversation_id`, `conversation_id_sources`, `conversation_id_baggage_keys` | — | A fixed conversation id, or other attribute / baggage names to read it from |
| `instruments`, `block_instruments` | all installed | What is auto-instrumented (§3) |
| `propagator` | W3C trace context + baggage | Context propagation format |
| `sampler` | always on | OpenTelemetry sampler |
| `resource_attributes` | — | Extra resource attributes |

`init()` raises `ConvivaTracingError` when there is neither a valid customer
key nor an endpoint. It does so before changing any global state.

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `CONVIVA_CUSTOMER_KEY` | — | Customer key. Required unless an endpoint is set. |
| `CONVIVA_PROJECT_ID` | your account's default project | Project id. Optional. |
| `CONVIVA_OTLP_ENDPOINT` | derived | Endpoint override. |
| `CONVIVA_TRACE_CONTENT` | `true` | `false` keeps prompts, completions and payloads off spans. |
| `OTEL_SERVICE_NAME` | — | Service name when none is passed. |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_ENDPOINT` | — | Standard OpenTelemetry endpoint overrides, honoured after `CONVIVA_OTLP_ENDPOINT`. |
| `OTEL_EXPORTER_OTLP_HEADERS` | — | Extra export request headers, e.g. for a proxy. |

Explicit `init()` arguments win over environment variables.

### Attributes added to every span

| Attribute | Value |
|---|---|
| `cnv.project_id` | Your project id, when one is configured |
| `cvx.conversation_id`, `gen_ai.conversation.id` | The conversation id (§5) |
| `c3.client_id`, `c3.conv_id`, `c3.msg_id` | The client sensor's ids, when present (§6) |

---

## 11. API reference

### `conviva_agent_tracing`

| API | Description |
|---|---|
| `init(**options) → ResolvedConfig` | Start tracing. A second call logs a warning and returns the first configuration. |
| `force_flush()` | Export everything still buffered. |
| `shutdown()` | Flush before the process exits. |
| `get_config() → ResolvedConfig \| None` | The active configuration. |
| `@workflow`, `@task`, `@agent`, `@tool` `(name=None, version=None)` | Decorators that run the function inside a span of that kind. |
| `@aworkflow`, `@atask`, `@aagent`, `@atool` | The same, for `async` functions. |
| `@conversation(conversation_id)` | Run the function with a fixed conversation id. |
| `set_association_properties(properties)` | Attach properties to every span in the current context. |
| `track_llm_call(vendor, type)`, `LLMMessage`, `LLMUsage` | Manual LLM call span (§7). |
| `Instruments` | Names for `instruments` / `block_instruments`. |
| `BaggageSpanProcessor`, `BaggageConfig`, `ensure_baggage_propagator` | The traces connector, re-exported. |
| `ConvivaTracingError` | Raised for invalid configuration. |

### `conviva_otlp_exporter` and `conviva_agent_traces_connector` — options 2 and 3

| API | Description |
|---|---|
| `resolve_config(customer_key=None, project_id=None, endpoint=None, headers=None)` | Reads the customer key, optional project id and endpoint. Raises `ConvivaTracingError` on invalid configuration. |
| `create_trace_exporter(config)` | The OTLP exporter for Conviva. |
| `create_conviva_span_processors(config, ...)` | The span processors that add the conversation id, project id and your own attributes, in the order they must run. |
| `CompositeSpanProcessor(*processors)` | Runs several processors as one, for libraries that accept a single processor. |
| `BaggageSpanProcessor(config=None)` | The traces connector: copies the client sensor's ids onto every span. Add it first. |

### Flush before exit

Spans are exported in batches in the background. Before a script, batch job or
serverless function returns:

```python
conviva_agent_tracing.force_flush()
```

---

## 12. Verifying your integration

1. **Locally first.** Run an OpenTelemetry collector or Jaeger all-in-one
   (OTLP on port 4318) and set `CONVIVA_OTLP_ENDPOINT=http://localhost:4318`.
2. Send one request with a test header:
   `baggage: c3.client_id=test,c3.conv_id=test-conv,c3.msg_id=test-conv:1`.
3. Check the trace:
   - LLM spans show the model, messages and token usage;
   - every span carries `cvx.conversation_id`, `gen_ai.conversation.id`,
     the three `c3.*` ids, and `cnv.project_id` if you set a project id.
4. Remove `CONVIVA_OTLP_ENDPOINT` and run again. The traces appear in your
   Conviva project — or your account's default project if you set none.

Export failures are silent by default. While integrating, turn on logging:

```python
import logging
logging.basicConfig(level=logging.INFO)
```

---

## 13. Limitations

| Limitation | What to do |
|---|---|
| **Initialize first.** Libraries imported before `init()` are not traced. | Call `init()` at the very top of your entry point. |
| **One `init()` per process.** Configuration cannot change after start-up. | Configure once, at startup. |
| **`shutdown()` only flushes.** Instrumentation stays installed; tracing cannot be restarted in the same process. | Treat tracing as process-lifetime. |
| **Every span is exported**, including HTTP and database spans from other OpenTelemetry instrumentation in the process. | Use `block_instruments`, or §9 with your own filtering processor. |
| **Content is on by default**, and the LangChain / LangGraph integration may record message content even with `CONVIVA_TRACE_CONTENT=false`. | Redact before the model call, and verify what your setup records (§12). |
| **No automatic PII masking.** | Redact sensitive values in your code, or turn content off. |
| **Streaming token usage.** Some providers report usage on streamed responses only when asked (for example OpenAI's `stream_options={"include_usage": True}`). | Request usage on streamed calls. |
| **No cost attribute.** Spans carry model and token counts, not prices. | Cost is derived from model and token usage. |
| **Traces only.** No metrics or logs are sent to Conviva. | — |
| **Context across process boundaries.** Context follows async code and threads in-process; it does not cross job queues or worker processes by itself. | Propagate it with OpenTelemetry's `inject` / `extract`. |
| **Custom span post-processing hooks** are not supported. | Use `span_processors`. |

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ConvivaTracingError` on start-up | No customer key, or one that is not a valid host name | Check `CONVIVA_CUSTOMER_KEY`. |
| No LLM spans | The library was imported before `init()` | Initialize first. |
| Spans arrive but are not grouped | No conversation id | §5 B, or connect the client sensor (§6). |
| No `c3.*` attributes | The `baggage` header is blocked by CORS or not extracted | §6 steps 1 and 2. |
| Last spans of a script are missing | The process exited before export | `force_flush()`. |
| Traces are not where you expect | No project id, so they went to your account's default project | Set `CONVIVA_PROJECT_ID`. |
| Gateway returns 200 but nothing is visible | Spans lack a conversation id | Verify locally (§12). |
