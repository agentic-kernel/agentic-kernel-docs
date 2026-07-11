# 会话与多轮

自 **0.8.0** 起,一次运行可以进行对话。当 planner 发出 `ask_user`,运行停在
`waiting_for_user`——宿主通过携带回复的 resume 来回答它:

=== "TypeScript"

    ```ts
    const turn1 = await engine.run({ agent, task, host });
    // turn1.status === "waiting_for_user"

    const turn2 = await engine.resume({
      runId: turn1.state.runId,
      agent, task, host,
      userMessage: "用 staging 环境"
    });
    ```

=== "Python"

    ```python
    turn1 = await engine.run(agent=agent, task=task, host=host)
    # turn1.status == "waiting_for_user"

    turn2 = await engine.resume(
        agent=agent, task=task, host=host,
        run_id=turn1.state.run_id,
        user_message="用 staging 环境",
    )
    ```

内核把回复盖章为持久的用户 `Message`,发出 `user_message` 事件,并继续循环。
回复是 **replay 覆盖的**:`state.messages` 可以从事件日志精确重建(内核注入的
反思消息同样如此)。上下文驱动召回会自动使用最新的用户消息,召回随回答重新定向。

## Session 便利封装

`createSession` / `create_session` 封装一个任务的完整生命周期,包括澄清轮次:

=== "TypeScript"

    ```ts
    import { createSession } from "@agentic-kernel/core";

    const session = createSession(engine, { agent, host });
    const first = await session.send("部署这个服务");   // 可能反问
    const second = await session.send("用 staging");    // 回答它
    ```

=== "Python"

    ```python
    from agentic_kernel.core import create_session

    session = create_session(engine, agent=agent, host=host)
    first = await session.send("部署这个服务")
    second = await session.send("用 staging")
    ```

终态之后会话封存(`SESSION_COMPLETED`)——新任务开新会话。审批与调度等待属于
宿主工作流而非用户轮次,`send` 会拒绝(`SESSION_NOT_AWAITING_USER`)。

## 长期运行跨版本升级

以旧 schema 版本持久化的运行现在会在加载时迁移:持久化存储(`state-file`、
`state-postgres`)沿已知版本链升级快照(`migrate` 选项,默认开启),上个月停在
等待中的运行可以在这个月的内核下续跑。未知版本仍然硬失败——迁移是被设计的
路径,不是猜测。
