# 治理与长运行韧性

0.9.0 为 Agentic 时代加固内核:策略从"处决"改为"引导"、分级授权、可跨 resume 保真的
不透明供应商状态,以及可跨崩溃与 schema 升级续跑的运行。

## 拒绝是可恢复的,不是致命的

默认情况下,策略 `deny` 被记录为一条**失败 observation**(`policy_denied: <原因>`,
不执行工具,`failure_count + 1`),循环继续——让 planner 得以调整,受 `max_failures`
与重复动作检测约束。这符合真实 agent 从被挡的一步中恢复、而不是因一次失误放弃整个长
任务的方式。对零容忍路径(安全绕过尝试),可退回硬终止:

=== "TypeScript"

    ```ts
    new AgentEngine({ planner, policy, denyBehavior: "fail" });
    ```

=== "Python"

    ```python
    create_agent_engine(planner=..., policy=..., deny_behavior="fail")
    ```

## 分级策略

`createTieredPolicy` / `create_tiered_policy` 把工具的 `side_effect_level` 映射到
决策(默认:`none`/`read` → 允许,`write`/`destructive` → 需审批),支持按工具覆盖;
未知工具默认拒绝(fail closed)。它还实现 `filter_tools`,可在规划**之前**把审批级/
拒绝级工具从 planner 可见工具集中裁掉——为原生工具调用把授权前移,而不是浪费一轮后再拒。

## 不透明的供应商推理状态

Action 与 Message 携带可选的 `provider_metadata`,内核从不检视,但会持久化进事件日志
并在 replay 时重建——resume 后能把模型发出的内容(如加密的扩展思维签名)原样回传给
供应商,保持多轮连续性。

## 崩溃窗口调解

resume 时,若一个 `tool_call` 的 observation 从未落盘(在执行工具与持久化结果之间崩溃),
会被调解为一条失败的 `tool_result_unknown_after_resume` observation——让 planner 看到
明确的"结果未知"而不是悄然悬挂的动作(投递是 at-least-once——应验证而非假设)。工具带
exactly-once 回执时可用 `reconcile_dangling_tool_calls=False` 关闭。

## 反思文案

内核负责停滞*检测*;提示语归你:

=== "Python"

    ```python
    create_agent_engine(planner=..., policy=..., reflection_message=lambda s: "换个思路重试")
    ```

## 可靠性度量:pass^k

评测器报告 **pass^k 曲线**(`pass_hat_k`,无偏 `C(successes,k)/C(trials,k)`),而不只是
全有全无的标量——因为 `pass@1` 会高估生产可靠性。Markdown 报告展示 `pass^2` 与最差
`pass^k`,使可靠性随 k 的衰减可见。
