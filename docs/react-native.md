# React Native and Cactus

On-device models in a React Native app, traced with the same GenAI semantic conventions as
server-side calls. Verified on a phone (Nothing Phone 2a, React Native 0.81, Hermes,
`cactus-react-native` 1.13) in the [LiquidChat](https://github.com/Mandark-droid/LiquidChat)
app: spans for model download, load, completion and embedding reached an OTLP/HTTP collector
with token counts, timings and the model in use.

## Install

```bash
npm install @traceverse/genai-otel-instrument-node @opentelemetry/api
```

Import from the `react-native` entry: it has no Node APIs (`async_hooks`, `fs`, `http`) and
exports over `fetch`.

```ts
import { AppState } from 'react-native';
import { CactusLM, CactusSTT } from 'cactus-react-native';
import {
  instrument,
  instrumentCactus,
  flushTelemetry,
} from '@traceverse/genai-otel-instrument-node/react-native';

instrument({
  serviceName: 'my-app',
  // OTLP/HTTP base URL; /v1/traces and /v1/metrics are added.
  endpoint: 'https://otel.example.com:4318',
  headers: { Authorization: 'Bearer …' },
  // Prompts and replies stay off the wire unless you opt in.
  enableContentCapture: false,
  // Flush when the app goes to the background: a backgrounded app can be killed.
  appState: AppState,
});

// React Native cannot look packages up by name: hand the classes over.
instrumentCactus({ CactusLM, CactusSTT });
```

There is no default endpoint. Telemetry leaves the phone only for a collector you name, which
makes export something the user turns on (LiquidChat keeps it off until it is enabled in
Settings).

## Spans from Cactus

| Span | When | Main attributes |
|---|---|---|
| `cactus.download` | model weights fetched | `gen_ai.request.model`, `cactus.model_file` |
| `cactus.init` | model loaded into memory (not when it already is: the engine calls `init()` before every completion) | load time, model file |
| `cactus.complete` | a completion | input and output tokens, time to first token, decode tokens/s, memory, tool calls |
| `cactus.embed` | an embedding | dimensions, input count |
| `cactus.stt.download`, `cactus.stt.init`, `cactus.stt.transcribe` | speech to text | model, timing |

Each of these spans carries `openinference.span.kind`: `LLM` for completions and
transcriptions, `EMBEDDING` for embeddings, `CHAIN` for downloads and loads (see
[Backends](#backends)).

## Your own spans

The SDK registers the global tracer provider, so the rest of the app can add spans with
`@opentelemetry/api` and they go to the same collector.

React Native has no async context of its own, so the SDK registers a context manager that
keeps a context active until the promise passed to `context.with()` settles. Run your work
inside your span's context, and every Cactus call it awaits becomes a child of that span:
one trace per agent run or pipeline, not one per model call.

```ts
import { context, trace } from '@opentelemetry/api';

const tracer = trace.getTracer('my-app');
const run = tracer.startSpan('invoke_agent my-agent', {
  attributes: { 'gen_ai.operation.name': 'invoke_agent', 'gen_ai.agent.name': 'my-agent' },
});
await context.with(trace.setSpan(context.active(), run), async () => {
  const plan = await lm.complete({ messages });   // cactus.complete: a child of the run
  await runTools(plan);                            // spans started here are children too
});
run.end();
```

Only your own `context.with()` scopes become active. The SDK's model-call spans are
leaves and never do (from `0.1.0-alpha.5`): a call running in the background, such as an
embedding for a search index, does not become the parent of whatever the app starts
meanwhile. Use `0.1.0-alpha.5` or later; in alpha.4 such a call could become the root of
an unrelated trace.

The active context is the most recently entered of your scopes that has not finished.
When two of your pipelines run at the same time (a meeting being transcribed while the
user chats), a span started in the older one can attach to the newer one; where that
matters, pass the parent explicitly instead:

```ts
import { context, trace } from '@opentelemetry/api';

const tracer = trace.getTracer('my-app');
const run = tracer.startSpan('invoke_agent my-agent', {
  attributes: { 'gen_ai.operation.name': 'invoke_agent', 'gen_ai.agent.name': 'my-agent' },
});
const parent = trace.setSpan(context.active(), run);

const tool = tracer.startSpan(
  'execute_tool search',
  { attributes: { 'gen_ai.operation.name': 'execute_tool', 'gen_ai.tool.name': 'search' } },
  parent,
);
// … run the tool …
tool.end();
run.end();
```

LiquidChat uses both for agent runs (`invoke_agent`), tool and MCP calls (`execute_tool`, with
`mcp.server.name`), retrieval (`retrieve`), its vector store (`query <store>` with
`rag.result.score_*`) and its meeting pipeline.

For a model call the SDK cannot see (a remote API called with `fetch`), wrap it with
`traceGenAiCall` to get the same span, token and cost attributes:

```ts
import { traceGenAiCall } from '@traceverse/genai-otel-instrument-node/react-native';

const reply = await traceGenAiCall(
  { name: 'chat my-model', system: 'my-provider', model: 'my-model' },
  async () => {
    const r = await callMyApi();
    return { ...r, usage: { prompt_tokens: r.inputTokens, completion_tokens: r.outputTokens } };
  },
);
```

## Backends

Export is plain OTLP/HTTP, so any OpenTelemetry backend works: point `endpoint` at it and
pass whatever headers it requires. There is no vendor package and nothing
backend-specific to install.

Spans follow the OpenTelemetry GenAI semantic conventions. Two attributes from the
OpenInference conventions are added as well, because backends built on them (Arize AX,
Phoenix and others) need them and ignore the rest:

| Attribute | Where | Why |
|---|---|---|
| `openinference.span.kind` | every span | how those backends classify a span: `LLM`, `EMBEDDING`, `RETRIEVER`, `TOOL`, `AGENT`, `CHAIN`. Instrumented spans set it themselves; for your own spans it comes from `gen_ai.operation.name` or the span name (`invoke_agent ...` is `AGENT`, `execute_tool ...` `TOOL`, `retrieve ...` `RETRIEVER`, other internal spans `CHAIN`). One you set yourself is kept. Without it, an operation outside the GenAI registry (speech to text, a model load, a pipeline step) shows as "unknown" |
| `openinference.project.name` | the resource | the project the traces are filed under: the service name, or `OPENINFERENCE_PROJECT_NAME` passed to `instrument()`. Arize AX rejects a trace without one |

Content the span already carries (from `0.1.0-alpha.6`) is also exported under the names
those backends display: `input.value` and `output.value` from the GenAI messages, tool
arguments and results, or the retrieval query; `tool.name`, `tool.description` and
`tool.parameters` from the tool attributes. It is a copy of what you captured: with content
capture off there is nothing to copy.

Other backends store them like any other attribute. To leave all of this out, pass
`openInferenceSpanKinds: false`.

### Retrieved documents

Record what a search returned, in rank order, with `addRetrievalAttributes` (every entry;
a port of the Python library's `add_retrieval_attributes`, same names and limits):

```ts
import { addRetrievalAttributes } from '@traceverse/genai-otel-instrument-node/react-native';

const span = tracer.startSpan('retrieve memory', { attributes: { 'gen_ai.operation.name': 'retrieve' } });
const hits = await search(question);
addRetrievalAttributes(
  span,
  hits.map((h) => ({ id: h.id, score: h.score, content: h.text, metadata: { source: h.file } })),
  { query: question },
);
span.end();
```

| Attribute | Value |
|---|---|
| `retrieval.query` | the query, first 500 characters |
| `retrieval.documents.{i}.document.id` / `.score` / `.content` | the first five documents in rank order; content truncated to 500 characters |
| `retrieval.documents.{i}.document.metadata.<key>` | flattened metadata (keys 50, values 200 characters) |
| `retrieval.document_count` | how many documents came back |

The query and document text are your users' data: pass them only when you capture content.
Ids and scores alone still show the ranking (LiquidChat does exactly that).

For example, Arize AX takes OTLP directly:

```ts
instrument({
  serviceName: 'my-app',                 // also the project name
  endpoint: 'https://otlp.arize.com',
  headers: { space_id: '<space id>', api_key: '<api key>' },
});
```

To send to two backends at once (your own collector and a hosted one), export to the
collector and add an exporter per backend there, or give `instrument()` a `fetch` that
also posts each request to the second endpoint with its own headers (the
[LiquidChat](https://github.com/Mandark-droid/LiquidChat) app does the latter).

## Testing against a local collector

From a phone on USB, `adb reverse tcp:4318 tcp:4318` makes `http://localhost:4318` on the
phone reach a collector on the development machine. From the Android emulator, the host is
`10.0.2.2`.
