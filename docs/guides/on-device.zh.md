# 端侧模型

`model-ondevice` 是面向端侧/小型/本地原始文本模型的 `Planner` 适配器。它作为**独立
通道**与远端 `model-openai` planner 并存——SDK 内不做路由或回退;宿主按通道分别发起
运行。所有弱模型的鲁棒性都在 `plan()` 内:容错 JSON 提取、错误反馈式**修复重试**、
按 token 预算裁剪提示。模型运行时(Ollama / llama.cpp / transformers / WebLLM)以
`TextGenerator` **注入**,不绑定任何运行时。

## 一键预设

最简方式是用预设把 `OnDevicePlanner` 按已验证的默认配方(`plan-then-execute`,可覆盖)
接入引擎:

=== "TypeScript"

    ```ts
    import { createOnDeviceAgentEngine } from "@agentic-kernel/model-ondevice";

    const engine = createOnDeviceAgentEngine({
      planner: { generate },           // 你注入的 TextGenerator
      engine: { policy, toolRegistry } // createAgentEngine 所需的其余项
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

## 调优(对真实本地模型实测)

- 最大的端到端收益是配置:在交互通道上**关闭「思考」/ 避免推理模型**(实测约 17×),
  并优先 2–4B 模型——适配器自身开销是微秒级,推理才是秒级。
- `toolPromptStyle: "compact"` 把工具渲染为 `name(p1, p2): description`,为工具众多的
  agent 削减 prefill。
- `buildActionSchema()` 导出动作 JSON Schema,用于约束/语法解码(Ollama `format`、
  llama.cpp `json_schema`)。
- 解析器会把常见错误 `{"type":"<工具名>"}` 归一化为 `tool_call`;传 `onRepair` 可观测
  修复频率。
- **保持工具链短**——小模型能稳定完成 ≤2 跳任务,但在 3 跳链上会丢步;请拆解长任务。

## 达到 100% 可靠性

两个已验证的杠杆(见[开发记录](../reports/index.md)):

1. **`planMode: "plan-then-execute"`** —— 把计划外化为可重放的 `thought`;提升有据
   有效性(如 qwen3.5:2b 0→69%,gemma3:4b 50→81%)。
2. **`runSelfConsistent(run, { samples })`** —— 整体跑 N 次并对最终答案多数投票
   (`pass^k`)。plan-then-execute + 宿主 `verifyAnswer`(用 `verifiers` 模块组合)+
   回合投票,使 **qwen3.5:2b 与 gemma3:4b 在所有测试任务上达到 100%**。

成本控制让 100% 不必每次付满 N:`acceptFirst`(接受首个通过可靠校验的运行 → N→1)、
`minSamples`(领先无法被追上即停)、`concurrency`(并行波次)——实测某简单任务在 100%
不变下降本约 89%。
