# 原生工具调用

自 **0.8.0** 起,`OpenAIChatPlanner` 是复杂任务与长任务的推荐 planner。它走
`/chat/completions`——因此对 OpenAI 及一切 OpenAI 兼容网关(vLLM、Ollama、
Groq、DeepSeek 等)都可用——并采用 provider 原生工具调用而非"单动作 JSON 输出":

- **每工具独立 schema**:`ToolDefinition.inputSchema` 作为原生 `tools` 条目直达
  provider,参数按工具约束。
- **并行工具调用**:`tool_calls[]` 映射为内核的并行 `ToolCallAction[]` 批次——
  扇出只花一轮而不是 N 轮。
- **prompt cache 友好**:提示词是追加式消息转录、前缀稳定,长任务下缓存持续生效。
- **真正的流式输出**:答案以内容增量流出,不再从半截 JSON 里抠字段。
- **内核盖章身份**:模型不再编造 `id`/`createdAt`——由内核统一盖章并全程去重。

=== "TypeScript"

    ```ts
    import { OpenAIChatPlanner } from "@agentic-kernel/model-openai";

    const planner = new OpenAIChatPlanner({
      apiKey: process.env.OPENAI_API_KEY!,
      model: "gpt-5.1",
      // baseUrl: "http://localhost:11434/v1",  // 任意 OpenAI 兼容网关
      parallelToolCalls: true,
      temperature: 0.2
    });
    ```

=== "Python"

    ```python
    import httpx
    from agentic_kernel.model_openai import OpenAIChatPlanner, OpenAIChatPlannerOptions
    from agentic_kernel.model_openai.httpx_fetch import httpx_fetch

    client = httpx.AsyncClient()
    planner = OpenAIChatPlanner(
        OpenAIChatPlannerOptions(
            api_key="...",
            model="gpt-5.1",
            fetch=httpx_fetch(client),   # 参考 HttpFetch——随 [openai] extra 开箱可用
            parallel_tool_calls=True,
        )
    )
    ```

控制动作以一小组合成工具提供给模型调用:`ask_user`(停在
`waiting_for_user`——见[会话](sessions.md))与 `stop` 恒定可用;
`push_goal`/`complete_goal` 由 `enableGoalTools` 开启。控制调用永远优先于同批的
工具调用,因为内核批次必须是纯工具批次。

Responses-API planner(`OpenAIPlanner`)仍然保留,供需要严格单动作结构化输出的
宿主使用。
