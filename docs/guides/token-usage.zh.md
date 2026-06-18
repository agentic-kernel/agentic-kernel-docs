# Token 用量

内核**测量** token 用量,并允许你按它**设界**——这正是它对延迟/计数与 `RunBudget`
已有的两项职责。它**不**定价、不按成本路由:那是宿主的事,基于这里暴露的计数推导。
(0.6.0 引入。)

## 数据如何流动

1. **planner 上报**:每次模型调用通过可选回调 `onUsage` / `on_token_usage` 上报用量
   (非破坏性——无法测量的 planner 永不调用它)。
2. **运行时记录**:把每次上报作为 `token_usage` 事件,并聚合累计的 `state.tokenUsage`
   (Python 为 `state.token_usage`)。
3. **重放**:从这些事件重新聚合,使重放状态与 live 状态一致。

`TokenUsage` = `{ inputTokens, outputTokens, totalTokens, model?, metadata? }`
(Python 为 snake_case)。

## 读取

=== "TypeScript"

    ```ts
    const result = await engine.run({ agent, task, host });
    console.log(result.state.tokenUsage);
    // { inputTokens: 108, outputTokens: 11, totalTokens: 119, model: "…" }
    ```

=== "Python"

    ```python
    result = await engine.run(agent=agent, task=task, host=host)
    print(result.state.token_usage)  # TokenUsage(input_tokens=108, output_tokens=11, ...)
    ```

## 按 token 设预算

`RunBudget` 新增 token 上限,与 `maxToolCalls`/`maxWallClockMs` 在同一轮前检查中强制
——达到上限即以 `budget_exhausted` 停止:

=== "TypeScript"

    ```ts
    await engine.run({ agent, task, host, budget: { maxTotalTokens: 50_000 } });
    // 还有:maxInputTokens、maxOutputTokens
    ```

=== "Python"

    ```python
    await engine.run(agent=agent, task=task, host=host,
                     budget=RunBudget(max_total_tokens=50_000))
    ```

## 数字从哪里来

- **model-openai** 读取 provider 的 `usage`(Responses API 的 `input_tokens` /
  `output_tokens`),阻塞与流式均可——精确计数。
- **model-ondevice** 在注入的 generator 返回 `{ text, usage }` 时用精确计数
  (如 Ollama 的 `prompt_eval_count` / `eval_count`);否则用 `countTokens` **估算**,
  并标记 `metadata.estimated = true`。

## 导出与评测

- **observer-otel** 为每个 `token_usage` 事件发出 OpenTelemetry GenAI 指标
  `gen_ai.client.token.usage`(按 `gen_ai.token.type` = input/output 分桶的 histogram,
  带 `gen_ai.request.model`)。
- **evaluator** 在 `EvalTrialMetrics` 加入 token 总量,在汇总中加入 `averageTotalTokens`
  / `totalTokens`。成本仍走宿主的 `estimateCost` 钩子——现在可以喂真实 token。

## 非目标

内核内不含价目表、单 token 费率或按成本路由。成本是宿主/evaluator 基于这些计数计算的。
