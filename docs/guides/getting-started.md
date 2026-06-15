# Getting started

Build your first agent in either SDK. The shape is the same: construct an engine from
a **planner** (the model) and a **policy** (the guardrails), register **tools**, then
`run`.

!!! tip "Install first"
    Follow [Installation](consuming-packages.md) to add `@agentic-kernel/core` (npm)
    or `agentic-kernel` (pip). The examples below also use a planner adapter —
    `@agentic-kernel/model-openai` / `model-ondevice` (TS) or the matching Python
    subpackage.

## TypeScript

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

const tools = new ToolRegistry();
tools.register({
  name: "search_city",
  description: "Look up a city's population.",
  inputSchema: { type: "object", properties: { city: { type: "string" } }, required: ["city"], additionalProperties: false },
  sideEffectLevel: "read",
  requiredScopes: [],
  execute: async ({ city }) => ({ city, population: 2_100_000 })
});

const engine = createAgentEngine({
  planner: new OpenAIPlanner({ apiKey: process.env.OPENAI_API_KEY!, model: "gpt-5" }),
  policy: new BasicPolicy({ maxSteps: 8, allowedTools: ["search_city"] }),
  toolRegistry: tools
});

const result = await engine.run({
  agent: { id: "a", name: "Analyst", role: "analyst", goal: "answer with tools", instructions: "Use search_city.", tools: ["search_city"], maxSteps: 8 },
  task: { id: "t1", input: "What is the population of Paris?" },
  host: { tenantId: "acme", principal: { id: "u", type: "user", tenantId: "acme" }, effectiveScopes: [], traceId: "t1" }
});
console.log(result.status, result.status === "completed" && result.output);
```

## Python

```python
from agentic_kernel.core import create_agent_engine, BasicPolicy, ToolRegistry
from agentic_kernel.core.policy import BasicPolicyConfig
from agentic_kernel.core.tools import ToolDefinition
from agentic_kernel.core.contracts import Agent, Task, HostContext, Principal

tools = ToolRegistry()
tools.register(ToolDefinition(
    name="search_city", description="Look up a city's population.",
    input_schema={"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"], "additionalProperties": False},
    side_effect_level="read", required_scopes=[],
    execute=lambda args, ctx: {"city": args["city"], "population": 2_100_000},
))

engine = create_agent_engine(
    planner=my_planner,                       # an OpenAI / on-device / custom Planner
    policy=BasicPolicy(BasicPolicyConfig(max_steps=8, allowed_tools=["search_city"])),
    tool_registry=tools,
)

result = await engine.run(
    agent=Agent(id="a", name="Analyst", role="analyst", goal="answer with tools", instructions="Use search_city.", tools=["search_city"], max_steps=8),
    task=Task(id="t1", input="What is the population of Paris?"),
    host=HostContext(tenant_id="acme", principal=Principal(id="u", type="user", tenant_id="acme"), effective_scopes=[], trace_id="t1"),
)
print(result.status, result.output)
```

## Without an API key — run a local model

No remote key needed: point the on-device adapter at a local
[Ollama](https://ollama.com) (`ollama pull qwen3.5:2b`). `createOnDeviceAgentEngine`
wires the validated recipe (`plan-then-execute`) for you.

=== "TypeScript"

    ```ts
    import { createOnDeviceAgentEngine } from "@agentic-kernel/model-ondevice";
    import { BasicPolicy, ToolRegistry } from "@agentic-kernel/core";

    const generate = async ({ messages }) => {
      const r = await fetch("http://127.0.0.1:11434/api/chat", {
        method: "POST",
        body: JSON.stringify({ model: "qwen3.5:2b", messages, stream: false })
      });
      return (await r.json()).message.content;
    };

    const engine = createOnDeviceAgentEngine({
      planner: { generate },
      engine: { policy: new BasicPolicy({ maxSteps: 8, allowedTools: ["search_city"] }), toolRegistry: tools }
    });
    // engine.run({ agent, task, host }) as above
    ```

=== "Python"

    ```python
    import httpx
    from agentic_kernel.model_ondevice import OnDevicePlannerOptions, create_on_device_agent_engine

    async def generate(req):
        async with httpx.AsyncClient() as c:
            r = await c.post("http://127.0.0.1:11434/api/chat",
                             json={"model": "qwen3.5:2b", "messages": req.messages, "stream": False})
        return r.json()["message"]["content"]

    engine = create_on_device_agent_engine(
        OnDevicePlannerOptions(generate=generate),
        policy=BasicPolicy(BasicPolicyConfig(max_steps=8, allowed_tools=["search_city"])),
        tool_registry=tools,
    )
    ```

For small models, see [On-device models](on-device.md) for the reliability recipe
(`plan-then-execute` + voting) that reaches 100%.

## What `run` returns

`engine.run(...)` resolves to an `AgentResult` with a `status`:

- `completed` → `output` holds the final answer.
- `failed` → `error`; `stopped` → `reason`.
- `waiting_for_user` / `waiting_for_approval` / `waiting_for_schedule` → the run paused;
  resume it with `engine.resume(...)` (or `resolveApproval(...)`). The full state is on
  `result.state`, and `result.state.runId` lets you reload from the `StateStore`.

## Next

- [Architecture](../architecture.md) — how the loop and seams fit together.
- [Memory](memory.md) · [On-device models](on-device.md) · [Multi-agent orchestration](multi-agent.md)
- [Installation](consuming-packages.md) — TypeScript (npm) and Python (pip) in detail.
