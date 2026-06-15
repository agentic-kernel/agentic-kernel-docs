# Getting started

Build your first agent in either SDK. The shape is the same: construct an engine from
a **planner** (the model) and a **policy** (the guardrails), register **tools**, then
`run`.

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

## Next

- [Architecture](../architecture.md) — how the loop and seams fit together.
- [Memory](memory.md) · [On-device models](on-device.md) · [Multi-agent orchestration](multi-agent.md)
- [Consuming packages](consuming-packages.md) — install from the registry.
