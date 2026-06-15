# On-device models

`model-ondevice` is a `Planner` adapter for on-device / small / local raw-text models.
It runs as an **independent lane** beside the remote `model-openai` planner — no routing
or fallback; a host starts a separate run per lane. All weak-model resilience lives
inside `plan()`: tolerant JSON extraction, error-feedback **repair retries**, and
token-aware prompt trimming. The model runtime (Ollama / llama.cpp / transformers /
WebLLM) is **injected** as a `TextGenerator`; nothing is bundled.

## Turnkey preset

For a one-call setup, the preset wires an `OnDevicePlanner` into the engine with the
validated default recipe (`plan-then-execute`, overridable):

=== "TypeScript"

    ```ts
    import { createOnDeviceAgentEngine } from "@agentic-kernel/model-ondevice";

    const engine = createOnDeviceAgentEngine({
      planner: { generate },           // your injected TextGenerator
      engine: { policy, toolRegistry } // everything createAgentEngine needs
    });
    ```

=== "Python"

    ```python
    from agentic_kernel.model_ondevice import (
        OnDevicePlannerOptions, create_on_device_agent_engine,
    )

    engine = create_on_device_agent_engine(
        OnDevicePlannerOptions(generate=generate),
        policy=..., tool_registry=...,
    )
    ```

## Tuning (measured against real local models)

- **Disable "thinking" / avoid reasoning models** on an interactive lane (~17× faster);
  prefer a 2–4B model — adapter overhead is microseconds vs seconds of inference.
- `toolPromptStyle: "compact"` renders tools as `name(p1, p2): description` to cut
  prefill for tool-heavy agents.
- `buildActionSchema()` exports the action JSON Schema for constrained/grammar decoding
  (Ollama `format`, llama.cpp `json_schema`).
- The parser normalizes the common `{"type":"<toolName>"}` mistake into a `tool_call`;
  pass `onRepair` to observe repair frequency.
- **Keep tool chains short** — small models reliably handle ≤2-hop tasks but drop steps
  on 3-hop chains; decompose long tasks.

## Reaching 100% reliability

Two validated levers (see [reports](../reports/index.md)):

1. **`planMode: "plan-then-execute"`** — externalize the plan as a replayable `thought`;
   lifts grounded validity (e.g. qwen3.5:2b 0→69%, gemma3:4b 50→81%).
2. **`runSelfConsistent(run, { samples })`** — run the whole agent N times and
   majority-vote the final answer (`pass^k`). Plan-then-execute + a host
   `verifyAnswer` (compose from the `verifiers` module) + episode voting reached **100%
   on qwen3.5:2b and gemma3:4b across all tested tasks**.

Cost controls keep 100% cheap: `acceptFirst` (accept the first verifier-passing run →
N→1), `minSamples` (early stop), `concurrency` (parallel waves) — measured −89% on an
easy task at unchanged 100%.
