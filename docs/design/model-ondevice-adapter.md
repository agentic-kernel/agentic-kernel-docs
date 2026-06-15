# Design Spec: `model-ondevice` adapter

- **Date:** 2026-06-08
- **Status:** Draft — awaiting review
- **Scope:** new `Planner` adapter package, both SDKs (TS + Python). **No core change.**

## Context & goal

We run a **hybrid** deployment where remote and on-device models go **separate
lanes** ("各走各的"): the host starts an independent `engine.run()` per lane —
remote tasks with `OpenAIPlanner`, on-device tasks with the new on-device
planner. There is **no** routing/fallback/escalation in the SDK; combining lanes
is a host concern.

On-device / small / local models differ from remote models on several axes —
most painfully **unreliable structured output** and **small context windows**.
This adapter contains all of that resilience **inside the planner**, so:

- **`core` is untouched** — it only sees the `Planner` interface (one successful
  plan, or one failure).
- **The remote path (`core` + `model-openai`) does not change at all** — it does
  not know on-device exists. The large-model capability cannot regress.

The retry/repair lives **inside `OnDevicePlanner.plan()`**, invisible to core.
This is why the earlier idea of adding `maxPlanRetries`/`repairHint` to `core`
was the wrong layer and was dropped.

## Non-goals

- Bundling any model runtime (WebLLM / Ollama / transformers.js / llama.cpp) —
  the runtime is **injected**, exactly like `model-openai` injects `fetch`.
- Routing / fallback / escalation between lanes (host concern).
- Replacing `model-openai`.
- Any change to `core` or other packages.

## Package

- **TS:** `@agentic-kernel/model-ondevice` (peer of `model-openai`).
- **Python:** `agentic_kernel.model_ondevice` (sibling subpackage).
- Implements the existing `Planner` seam (`plan` + optional `planStream`).

## Public API (locked decisions)

Injected text generator — the only required dependency (no runtime bundled):

```ts
type ChatMessage = { role: "system" | "user" | "assistant"; content: string };

type GenerateRequest = {
  messages: ChatMessage[];
  schema?: JsonObject; // for runtimes that support constrained decoding; else ignored
  signal?: AbortSignal;
};

// Blocking or streaming; the planner adapts to either.
type TextGenerator = (req: GenerateRequest) => Promise<string> | AsyncIterable<string>;
```

Planner:

```ts
new OnDevicePlanner({
  generate: TextGenerator,            // required: WebLLM / Ollama / transformers.js / ...
  actions?: ActionKind[],             // restricted vocabulary; DEFAULT below
  maxRepairAttempts?: number,         // default 2
  maxPromptTokens?: number,           // default 3500
  countTokens?: (s: string) => number,// default ~ ceil(chars / 4)
  clock?: Clock,                      // to stamp action.createdAt if model omits it
  idGenerator?: IdGenerator           // to stamp action.id if model omits it
});
```

Locked decisions:

1. **Default action vocabulary (restricted):** `tool_call`, `final_answer`,
   `ask_user`, `stop`, `thought`. Host opts into more (`spawn_subagent`,
   `join_subagent`, `cancel_subagent`, `push_goal`, `complete_goal`, `schedule`)
   via `actions`. Fewer choices ⇒ higher reliability for small models.
2. **Streaming in v1:** implements `planStream`; reasoning/text deltas surface as
   `PlannerChunk`s. Falls back to blocking when the injected generator returns a
   string.
3. **`messages[]` chat interface** (not a single prompt string).
4. **Name:** `model-ondevice`. (Capability is broader — works for any raw-text
   model — but we name by deployment per the product framing.)

## Internal pipeline (per `plan()`)

1. **Render** `AgentContext` → compact chat messages, trimmed to
   `maxPromptTokens`:
   - **system**: agent role/goal/instructions; "Return exactly ONE JSON action";
     the allowed action shapes (compact); available tools (name + 1-line desc +
     input schema).
   - **transcript**: recent messages, recent action→observation pairs as terse
     lines, retrieved memory (compact), `compactedSummary` if present.
   - **trim order** when over budget: drop oldest transcript/memory first; always
     keep the task input, the tool list + action schema, and the most recent
     observation.
2. **Generate** via the injected `generate` (pass `schema` for constrained-decode
   runtimes; stream chunks to `onChunk`/`planStream` when async-iterable).
3. **Tolerant parse** → `Action`:
   - strip ```` ```json ```` fences and surrounding prose; extract the first
     balanced JSON object; `JSON.parse`.
   - require `type` ∈ allowed vocabulary; fill `id` (idGenerator) and `createdAt`
     (clock) if missing; validate required fields per type (`tool_call` →
     `toolName`+`arguments`, `final_answer` → `content`, etc.).
4. **Repair loop** on parse/validation failure: append the bad output + a strict
   correction turn ("Your last reply was not a valid action: <error>. Reply with
   ONLY one JSON action, no prose."), retry up to `maxRepairAttempts`.
5. **Give up** → throw `OnDevicePlannerError`. Core's runtime catches a planner
   throw and fails the run **exactly as today** — no core change, no behavior
   change for any other planner.

## Failure & observability semantics

- All retries are internal; core sees success or one terminal failure.
- The planner MAY surface repair attempts via `onChunk` text or its own logging;
  it does **not** emit core events (it has no access to the event log — correct
  separation). Hosts that want repair telemetry wrap `generate`.

## Parity (TS ↔ Python)

- Identical design: `OnDevicePlanner` + `TextGenerator` (Python: a `Callable`
  protocol), same options (snake_case), same render/parse/repair/token pipeline.
- The injected generator is platform-specific and host-provided (TS browser:
  WebLLM; Python local: Ollama/llama.cpp). The package ships **no** binding.
- Both at full gate: TS build + typecheck + prettier + vitest thresholds;
  Python ruff + mypy --strict + pytest 100% + CI.

## Testing (no real model needed)

Inject a **scripted `TextGenerator`** returning canned strings to exercise:

- clean JSON → action; fenced JSON; prose-wrapped JSON;
- malformed-then-fixed sequence → repair loop succeeds; always-malformed →
  throws after `maxRepairAttempts`;
- each allowed action type; a disallowed type → rejected/repaired;
- missing `id`/`createdAt` → stamped;
- token-budget trimming (long context → oldest dropped, task/tools/last-obs
  kept);
- streaming generator → chunks surfaced + final action;
- 100% statement+branch (Python) / thresholds (TS).

## Example: two independent lanes (host)

```ts
// remote lane — unchanged
const remote = createAgentEngine({ planner: new OpenAIPlanner({ fetch, apiKey, model }), policy, toolRegistry });
await remote.run({ agent, task: remoteTask, host });

// on-device lane — independent run, same kernel
const onDevice = createAgentEngine({
  planner: new OnDevicePlanner({ generate: webllmGenerate }),
  policy, toolRegistry
});
await onDevice.run({ agent, task: localTask, host });
// Optional: inject the same MemoryManager into both to share lessons.
```

## Future (not v1)

- Constrained/grammar decoding helpers per runtime (currently just pass `schema`
  through; tolerant parse + repair is the safety net).
- Multi-action batch output (weak models struggle; single-action only in v1).
- Token-aware compaction in `core` (optional; the adapter trims its own prompt,
  so not required).
