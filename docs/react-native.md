# React Native and Cactus

On-device models in a React Native app, traced with the same GenAI semantic conventions as
server-side calls. Verified on a phone (Nothing Phone 2a, React Native 0.81, Hermes,
`cactus-react-native` 1.13) in the [LiquidChat](https://github.com/Mandark-droid/LiquidChat)
app: spans for model download, load, completion and embedding reached an OTLP/HTTP collector
with token counts, timings and the model in use.

## Install

```bash
npm install @traceverse/genai-otel-instrument-node@next @opentelemetry/api
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
| `cactus.init` | model loaded into memory | load time, model file |
| `cactus.complete` | a completion | input and output tokens, time to first token, decode tokens/s, memory, tool calls |
| `cactus.embed` | an embedding | dimensions, input count |
| `cactus.stt.transcribe` | speech to text | audio duration, model |

## Your own spans

The SDK registers the global tracer provider, so the rest of the app can add spans with
`@opentelemetry/api` and they go to the same collector. React Native has no async context
manager: pass the parent context explicitly when a child should nest under it.

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

LiquidChat uses this for agent runs (`invoke_agent`), tool and MCP calls (`execute_tool`, with
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

## Testing against a local collector

From a phone on USB, `adb reverse tcp:4318 tcp:4318` makes `http://localhost:4318` on the
phone reach a collector on the development machine. From the Android emulator, the host is
`10.0.2.2`.
