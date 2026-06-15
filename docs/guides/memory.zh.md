# 记忆

记忆通过 `MemoryManager` 接缝注入。`ContextBuilder` 检索作用域内的条目并注入到
planner 上下文(发出 `memory_retrieved` / `memory_injected` 事件)。core 自带三个参考
实现。

## InMemoryMemory —— 词项重叠相关性

检索按**查询词在内容中出现的多少**对作用域内条目排序(问题能命中与之共享词语的
事实),而**非**子串包含。`preference`(偏好)条目总会被托底,使长期指令始终浮现。
平分时按条目 score、再按时间倒序。适用于测试与本地示例。

## EmbeddingMemory —— 语义召回

按查询向量与每个条目内容向量的余弦相似度排序。embedder 是**注入式**的——不绑定任何
模型,可在任意运行时使用。无文本的查询回退到 score/时间排序。

=== "TypeScript"

    ```ts
    import { EmbeddingMemory, type Embedder } from "@agentic-kernel/core";

    const embed: Embedder = async (text) => myEmbeddingModel(text); // number[]
    const memory = new EmbeddingMemory({ embed });
    // new AgentEngine({ planner, policy, memory, ... })
    ```

=== "Python"

    ```python
    from agentic_kernel.core import EmbeddingMemory

    async def embed(text: str) -> list[float]:
        return await my_embedding_model(text)

    memory = EmbeddingMemory(embed=embed)
    # engine = create_agent_engine(planner=..., policy=..., memory=memory)
    ```

## 作用域

记忆按租户命名空间隔离:作用域形如 `tenant/[workspace/][agent]`。种子条目的 `scope`
必须**位于** agent 作用域前缀之内才可被检索——例如租户 `acme`、agent `researcher`,
则 scope 为 `acme/researcher` 的条目匹配。

## 生产与治理

生产存储用 `memory-postgres`(或任意 `MemoryManager`)。长期记忆有治理生命周期——
`proposeWrite` / `supersede` / `tombstone` / `explain` / `export`——而
`SessionMemoryExtractor` 或记忆治理器可在完成时自动抽取可溯源的会话摘要。
