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
import { instrument } from '@traceverse/genai-otel-instrument-node/react-native';

instrument({
  serviceName: 'my-app',
  // OTLP/HTTP base URL; /v1/traces and /v1/metrics are added.
  endpoint: 'https://otel.example.com:4318',
  headers: { Authorization: 'Bearer …' },
  // Prompts and replies stay off the wire unless you opt in.
  enableContentCapture: false,
});
```

That is the only call (from `0.1.0-alpha.7`). `cactus-react-native` is instrumented when the app
has it installed, and React Native's `AppState` is used to send what is queued when the app
leaves the foreground; both are optional requires, so nothing breaks without them. A trace is
also sent as soon as it finishes, since the batch timer stops while the app is in the
background. Passing `appState`, or calling `instrumentCactus({ CactusLM, CactusSTT })`
yourself, still works.

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

An app's own components (its agent loop, tool dispatcher, search, pipelines) are not a library,
so no instrumentor can find them. Wrap each in one call; the conventions come with it:

```ts
import {
  traceAgent,
  traceRetrieval,
  traceTool,
  traceWorkflow,
} from '@traceverse/genai-otel-instrument-node/react-native';

const reply = await traceAgent(
  'assistant',
  { id: runId, input: text, output: (r) => r.reply, failure: (r) => (r.ok ? undefined : r.code) },
  () => runAgent(text),
);

await traceTool(
  'get_weather',
  { callId, description: 'Gets the weather for a city.', arguments: args, mcpServer: undefined },
  () => tools.call('get_weather', args),
);

const hits = await traceRetrieval(
  'notes',
  {
    query,
    system: 'sqlite',
    topK: 5,
    documents: (rows) => rows.map((r) => ({ id: r.id, score: r.score, content: r.text, metadata: { source: r.file } })),
  },
  () => search(query, 5),
);

await traceWorkflow('ingest', { resultAttributes: (r) => ({ 'app.pages': r.pages }) }, () => ingest(file));
```

| Helper | Span | Carries |
|---|---|---|
| `traceAgent` | `invoke_agent {name}` (AGENT) | agent name and id, conversation id, input and output (content) |
| `traceTool` | `execute_tool {name}` (TOOL) | call id, description, type; `mcp.server.name` and `mcp.method.name` for an MCP tool; arguments and result (content) |
| `traceRetrieval` | `retrieve {collection}` (RETRIEVER) | store, top-k, rows returned, `rag.result.score_*`, the documents in rank order (ids and scores always; text, metadata and query with content) |
| `traceWorkflow` | `{name}` (CHAIN) | input and output (content) |

Every helper nests what runs inside it, across `await`; records content only with
`enableContentCapture`; records an error's type always and its message only with content
capture; and takes `attributes` (at start), `resultAttributes` (from the result, not content)
and `parent`. They are in every entry, Node included.

The SDK registers the global tracer provider, so the rest of the app can also add spans with
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
| `graph.node.id`, `graph.node.parent_id`, `graph.node.name`, `graph.node.display_name`, `agent.name` | every span that names an agent | the agent graph (from `0.1.0-alpha.8`, as the Python library's 1.44.0): each agent is a node, its parent the nearest enclosing agent, so an agent started by another (a skill run by an assistant) is its child. From `0.1.0-alpha.9` the switch is parsed as the Python library parses it: on unless `GENAI_ENABLE_AGENT_GRAPH_ATTRIBUTES` is set to anything other than `true` |
| `session.id`, `user.id` | spans under a span that has them | from `0.1.0-alpha.9` (Python 1.45.0): a span without a session takes its parent span's, so every span of a traced run carries the run's session, not only the root. `GENAI_ENABLE_SESSION_PROPAGATION: 'false'` keeps only explicit sessions |
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

To send to more than one backend at once (your own collector and a hosted one), list the
others in `additionalDestinations` (from `0.1.0-alpha.7`). Each gets the same telemetry with its
own headers; the main endpoint's headers are never sent to them, and a destination that fails
is logged as a warning:

```ts
instrument({
  serviceName: 'my-app',
  endpoint: 'https://otel.example.com',
  headers: { Authorization: 'Bearer …' },
  additionalDestinations: [
    { endpoint: 'https://otlp.arize.com', headers: { space_id: '<space id>', api_key: '<api key>' } },
  ],
});
```

A failed export is logged as a warning, not at debug level: a backend that rejects every request
(a missing header, for example) would otherwise look the same as one that accepts them.

## Testing against a local collector

From a phone on USB, `adb reverse tcp:4318 tcp:4318` makes `http://localhost:4318` on the
phone reach a collector on the development machine. From the Android emulator, the host is
`10.0.2.2`.
