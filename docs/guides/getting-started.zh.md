# 快速上手

在任一 SDK 写出第一个 agent。套路一致:用 **planner**(模型)和 **policy**(护栏)
构建引擎,注册**工具**,然后 `run`。

!!! tip "先安装"
    按 [安装](consuming-packages.md) 添加 `@agentic-kernel/core`(npm)或
    `agentic-kernel`(pip)。下面的示例还会用到 planner 适配器——
    `@agentic-kernel/model-openai` / `model-ondevice`(TS)或对应的 Python 子包。

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

## 无需 API key —— 跑本地模型

不需要远端密钥:把端侧适配器指向本地 [Ollama](https://ollama.com)
(`ollama pull qwen3.5:2b`)。`createOnDeviceAgentEngine` 会为你接好已验证的配方
(`plan-then-execute`)。

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
    // 用法同上:engine.run({ agent, task, host })
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

小模型请参阅[端侧模型](on-device.md)中能达到 100% 的可靠性配方
(`plan-then-execute` + 投票)。

## `run` 返回什么

`engine.run(...)` 解析为带 `status` 的 `AgentResult`:

- `completed` → `output` 是最终答案。
- `failed` → `error`;`stopped` → `reason`。
- `waiting_for_user` / `waiting_for_approval` / `waiting_for_schedule` → 运行已暂停;
  用 `engine.resume(...)`(或 `resolveApproval(...)`)继续。完整状态在 `result.state`,
  `result.state.runId` 可用于从 `StateStore` 重新加载。

## 下一步

- [架构](../architecture.md) — 循环与接缝如何协作。
- [记忆](memory.md) · [端侧模型](on-device.md) · [多 Agent 编排](multi-agent.md)
- [安装](consuming-packages.md) — TypeScript(npm)与 Python(pip)详解。
