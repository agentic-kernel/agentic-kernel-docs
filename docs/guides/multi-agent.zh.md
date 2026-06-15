# 多 Agent 编排

`multi-agent` 通过注入的 `AgentRunner`(即 `AgentEngine`)协调父运行与子任务。子运行
沿用与单 agent 相同的 `Action`、`Observation`、策略、记忆与执行日志契约。

- **`SubagentDelegationManager`** 实现核心 `DelegationManager` 契约:`spawn_subagent`
  立即运行子任务或创建异步子任务,`join_subagent` 运行/观测它,`cancel_subagent` 停止
  它。子 agent 继承父的工具与能力、获得有界预算,并把结果作为父观测返回。
- **`TaskGraphCoordinator`** 增加只追加的交接、子运行 resume/cancel 钩子、observer 轨迹
  桥接、预算汇总,以及共享记忆写入提案 + 授权。

## 编排模式

在 `AgentRunner` 接缝上组合的可复用模式(不改内核)。每个模式接收一个 runner 与若干
`OrchestrationStep`(`{ agent, task, host, budget? }`):

| 模式 | 作用 |
|---|---|
| `runParallel(runner, steps, { concurrency })` | 有界并发扇出,结果按输入顺序;抛错的分支变为 `failed` 结果,不会拖垮整批。 |
| `mapReduce(runner, steps, reduce, { concurrency })` | 扇出后合并结果。 |
| `vote(runner, steps, { answerKey, concurrency })` | 对 N 次运行做辩论/多数投票(同一任务不同人设,或 N 次采样)→ `VoteResult { result, key, agreement, results }`。 |
| `plannerWorkerCritic(runner, { planner, decompose, synthesize })` | 一个 planner 运行拆解任务,workers 并行跑子步骤,critic 综合。`decompose`/`synthesize` 由宿主提供。 |

=== "TypeScript"

    ```ts
    import { vote } from "@agentic-kernel/multi-agent";

    const result = await vote(engine, [stepA, stepB, stepC]);
    console.log(result.key, result.agreement);
    ```

=== "Python"

    ```python
    from agentic_kernel.multi_agent import vote

    result = await vote(engine, [step_a, step_b, step_c])
    print(result.key, result.agreement)
    ```

这些模式在[真实 LLM 综合测试](../reports/2026-06-15-comprehensive-real-llm-campaign.md)中
用真实模型做了验证。
