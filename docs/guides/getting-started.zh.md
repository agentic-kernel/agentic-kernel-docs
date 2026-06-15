# 快速上手

在任一 SDK 写出第一个 agent。套路一致:用 **planner**(模型)和 **policy**(护栏)
构建引擎,注册**工具**,然后 `run`。

## TypeScript

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

const tools = new ToolRegistry();
tools.register({
  name: "search_city",
  description: "查询城市人口。",
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
  agent: { id: "a", name: "Analyst", role: "analyst", goal: "用工具作答", instructions: "使用 search_city。", tools: ["search_city"], maxSteps: 8 },
  task: { id: "t1", input: "巴黎的人口是多少?" },
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
    name="search_city", description="查询城市人口。",
    input_schema={"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"], "additionalProperties": False},
    side_effect_level="read", required_scopes=[],
    execute=lambda args, ctx: {"city": args["city"], "population": 2_100_000},
))

engine = create_agent_engine(
    planner=my_planner,                       # OpenAI / 端侧 / 自定义 Planner
    policy=BasicPolicy(BasicPolicyConfig(max_steps=8, allowed_tools=["search_city"])),
    tool_registry=tools,
)

result = await engine.run(
    agent=Agent(id="a", name="Analyst", role="analyst", goal="用工具作答", instructions="使用 search_city。", tools=["search_city"], max_steps=8),
    task=Task(id="t1", input="巴黎的人口是多少?"),
    host=HostContext(tenant_id="acme", principal=Principal(id="u", type="user", tenant_id="acme"), effective_scopes=[], trace_id="t1"),
)
print(result.status, result.output)
```

## 下一步

- [架构](../architecture.md) — 循环与接缝如何协作。
- [记忆](memory.md) · [端侧模型](on-device.md) · [多 Agent 编排](multi-agent.md)
- [安装与引入](consuming-packages.md) — 从仓库安装。
