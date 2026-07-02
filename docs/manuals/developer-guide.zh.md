# AgentEngine 开发者大师指南

使用 Agentic Kernel SDK 构建健壮、实时、可扩展的 Agent 应用的权威指南。

---

## 1. 架构哲学：微内核方法

AgentEngine 被设计为一个**确定性状态机**和**智能路由器**。它不拥有你的模型、数据库或工具；相反，它提供一个高保真的“内核（Kernel）”，通过严格的契约来协调这些组件。

### 双轨执行模型

为了解决“原子级可靠性”与“实时反馈”之间的冲突，AgentEngine 采用双轨架构：

1.  **可靠轨道（原子性）**：
    *   **组件**：`StateStore`、`PolicyManager`、`State`。
    *   **目标**：确保每一个动作和 observation 都被持久记录并受治理。如果运行崩溃，可以 100% 保真地恢复。
    *   **行为**：只有当一个步骤完全通过校验并提交后，事件才会被写入。

2.  **观测轨道（实时性）**：
    *   **组件**：`Observer`、`runStream`、`PlannerChunk`。
    *   **目标**：为 UI 提供零延迟反馈。
    *   **行为**：即时发射“思考”令牌和“实时状态”更新，绕过持久化写锁，确保 UI 永远不会显得“卡住”。

---

## 2. 核心组件与安装

### 包选择

| 包 | 用途 | 是否必需？ |
| :--- | :--- | :--- |
| `@agentic-kernel/core` | 核心运行时、契约与基础适配器。 | **是** |
| `@agentic-kernel/model-openai` | 面向 OpenAI 兼容 API（Responses API）的 Planner 实现。 | 否 |
| `@agentic-kernel/multi-agent` | 子代理与任务图的协调逻辑。 | 否 |
| `@agentic-kernel/state-postgres` | 面向生产环境的持久化 Postgres 存储。 | 推荐 |

### 最小实时示例

运行 AgentEngine 的现代方式是通过 `runStream`，它提供一等公民的 `AsyncIterable` 事件流。

```typescript
import { AgentEngine, BasicPolicy, StreamingObserver } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

const engine = new AgentEngine({
  planner: new OpenAIPlanner({ apiKey: "..." }),
  policy: new BasicPolicy({ maxSteps: 10 }),
  // stateStore: new PostgresStateStore(...), // For production
});

// Create the run
const runInput = {
  agent: { id: "expert-1", role: "Researcher", ... },
  task: { id: "task-101", input: "Research current AI trends" },
  host: { tenantId: "org-a", workspaceId: "team-1", ... }
};

// Consume the event stream
for await (const event of engine.runStream(runInput)) {
  switch (event.type) {
    case "reasoning_chunk":
      process.stdout.write(event.payload.content); // Stream thinking tokens
      break;
    case "action_selected":
      console.log(`\nDecision: ${event.payload.actionType}`);
      break;
    case "task_completed":
      console.log(`\nFinal Answer: ${event.payload.output}`);
      break;
  }
}
```

---

## 3. 高级流式与 UI 路由

### UI 路由对齐（Root-ID 模式）

在多代理或并发场景下，宿主应用需要精确知道一个令牌应路由到哪里（例如：哪个聊天气泡？）。AgentEngine 通过在每个事件的**根级别注入路由 ID** 来解决这个问题：

```json
{
  "id": "ev-123",
  "type": "reasoning_chunk",
  "runId": "run-456",
  "agentId": "specialist-a",  // Root-level for fast routing
  "taskId": "subtask-789",    // Root-level for UI mapping
  "step": 2,                  // Current execution step
  "payload": { ... }
}
```

**开发者守则**：在你的 UI 分发器/Electron IPC 桥中，始终使用根级别的 `agentId` 和 `taskId`，以确保“思考”令牌出现在正确的组件中。

### 智能批处理

为防止“IPC 洪泛”（可能导致 Electron 或 Web 浏览器卡顿），AgentEngine 会自动对高频令牌进行批处理：
*   **异步 Planner**：每 10 个令牌 Flush 一次（约 500ms 延迟）。
*   **同步 Planner**：通过微任务（Microtask）聚合（在下一个 tick 之前完成聚合）。

---

## 4. 多智能体编排

### 分形委托

AgentEngine 中的子代理是“分形”的：一个子代理只是另一次 `AgentEngine` 运行。

**关键特性**：
1.  **Observer 继承**：当 Orchestrator 派生一个子代理时，父级的 `runStream` 会自动捕获子代理的事件。用户看到的是单一、统一的事件流：“Orchestrator 正在思考……Specialist 正在思考……Specialist 已完成……Orchestrator 正在作答。”
2.  **种子记忆（Seed Memory）**：你可以通过向子代理的任务规格注入 `seedMemory` 来为其“预热”，使其无需新的检索阶段即可获得上下文。

```typescript
// Subagent execution automatically propagates events back to the parent stream
const result = await engine.run({
  ...parentContext,
  observer: parentObserver // Inheritance happens here
});
```

---

## 5. 主动式记忆治理

AgentEngine 区分**环境上下文（Ambient Context）**（被动检索）与**主动知识（Proactive Knowledge）**（主动写入）。

### 记忆工具模式

永远不要允许 Agent 直接写入数据库。相反，应将记忆操作暴露为**工具**。这确保它们受 `Policy` 治理，并可要求`人工审批`。

```typescript
// Recommended pattern: Capture MemoryManager in a tool closure
export function createMemoryWriteTool(memory: LongTermMemoryManager) {
  return {
    name: "propose_fact",
    execute: async (args, context) => {
      // Logic to propose a new fact to the user's permanent memory
      return await memory.proposeWrite({ ... }, [context.state.runId]);
    }
  };
}
```

---

## 6. 生产环境硬化清单

1.  **状态隔离**：确保你的 `StateStore` 实现使用 `runId` 作为主分区键。
2.  **并发安全**：使用**运行级 Observer**（将 `observer` 传入 `.run()`），而不是实例级 Observer，以避免多租户环境下的事件串扰。
3.  **人机协同**：为任何 `sideEffectLevel: "destructive"` 的工具实现 `ApprovalProvider`。
4.  **契约校验**：在部署前使用 `@agentic-kernel/conformance` 验证你的自定义 Postgres 或 VectorDB 适配器。

---

## 7. 排查“白屏”问题

如果你的 UI 显示“正在思考……”却始终不更新：
1.  **检查 ID**：确保你的 UI 组件监听的是 `AgentEvent` 中提供的那个特定 `taskId`。
2.  **验证 Flush**：确保你的 Planner 产出 `PlannerChunk` 事件。如果使用自定义 Planner，请高频调用 `onChunk` 回调。
3.  **继承**：如果缺失的令牌来自子代理，请验证 `DelegationManager` 是否收到了父级的 `Observer`。

---

*由 AgentEngine 架构团队出品，2026 年 5 月。*
