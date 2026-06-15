# Agentic Kernel SDK 全方位使用手册 (中文版)

**版本**: 1.0 (2026年5月)  
**定位**: 企业级自治 Agent 运行时内核  
**核心特性**: 双轨执行模型、全链路流式响应、分形多代理调度、精准 UI 路由对齐。

---

## 目录

1. [设计哲学：微内核与双轨模型](#1-设计哲学微内核与双轨模型)
2. [环境准备与安装](#2-环境准备与安装)
3. [快速上手：10分钟运行第一个流式 Agent](#3-快速上手10分钟运行第一个流式-agent)
4. [核心概念深度解析](#4-核心概念深度解析)
5. [实时流（Streaming）实战：从内核到 UI](#5-实时流streaming实战从内核到-ui)
6. [多智能体协作：分形代理与任务分发](#6-多智能体协作分形代理与任务分发)
7. [长期记忆管理：环境上下文与主动学习](#7-长期记忆管理环境上下文与主动学习)
8. [生产环境硬化：持久化、安全与审批](#8-生产环境硬化持久化安全与审批)
9. [测试、评估与质量保障](#9-测试评估与质量保障)
10. [常见问题与故障排查 (Troubleshooting)](#10-常见问题与故障排查)

---

## 1. 设计哲学：微内核与双轨模型

### 1.1 微内核设计 (Microkernel)
Agentic Kernel SDK **不绑定**任何特定的 LLM、数据库或前端框架。它只负责最核心的“思考循环 (Sense-Plan-Act Loop)”。
*   **宿主提供 (Host-Provided)**: 模型网关、数据库连接、业务工具、身份认证。
*   **SDK 治理 (SDK-Governed)**: 状态转移、安全性校验、任务编排、流式分发。

### 1.2 双轨执行模型 (Dual-Track)
为了平衡“金融级可靠性”与“互联网级实时性”，SDK 内部采用双轨制：
*   **可靠轨道 (Reliable Track)**: 负责 `StateStore` 写入。只有当一个步骤完全成功并经过策略校验后，才会持久化。
*   **观测轨道 (Observability Track)**: 负责 `Observer` 事件发射。实时将 Planner 产生的“思考令牌 (Thinking Tokens)”和“中间状态”推送到 UI，无需等待数据库写入。

---

## 2. 环境准备与安装

### 2.1 依赖安装
```bash
# 核心包
npm install @agentic-kernel/core

# 可选：OpenAI 适配器
npm install @agentic-kernel/model-openai

# 可选：Postgres 持久化
npm install @agentic-kernel/state-postgres @agentic-kernel/memory-postgres
```

---

## 3. 快速上手：10分钟运行第一个流式 Agent

以下代码展示了如何使用最新的 `runStream` 接口实现实时思考输出。

```typescript
import { AgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

async function main() {
  // 1. 注册工具
  const tools = new ToolRegistry();
  
  // 2. 初始化引擎 (双轨模式)
  const engine = new AgentEngine({
    planner: new OpenAIPlanner({ apiKey: "sk-..." }),
    policy: new BasicPolicy({ maxSteps: 10 }),
    toolRegistry: tools
  });

  // 3. 使用 runStream 获取实时异步流
  const runInput = {
    agent: { id: "assistant-1", role: "AI 助手", goal: "解答问题", instructions: "简洁明了", tools: [] },
    task: { id: "task-001", input: "请用一句话解释什么是量子纠缠。" },
    host: { tenantId: "demo-org", principal: { id: "user-1", type: "user", tenantId: "demo-org" }, effectiveScopes: [] }
  };

  console.log("Agent 正在思考...\n");

  for await (const event of engine.runStream(runInput)) {
    // 自动处理路由：event.agentId 和 event.taskId 已在根级别注入
    if (event.type === "reasoning_chunk") {
      process.stdout.write(event.payload.content); // 实时打印思维链令牌
    } else if (event.type === "task_completed") {
      console.log("\n\n回答完毕:", event.payload.output);
    }
  }
}

main();
```

---

## 4. 核心概念深度解析

### 4.1 Agent (谁在执行)
Agent 是逻辑身份，定义了其能力边界（Tools）和行为准则（Instructions）。

### 4.2 Task (做什么)
Task 定义了输入参数、初始变量和种子记忆（Seed Memory）。

### 4.3 HostContext (在什么环境下)
**这是 SDK 最重要的安全边界**。它包含了租户 ID、工作区 ID 和当前用户的权限 Scope。
*   **原则**: 工具实现中应始终通过 `context.host` 派生权限，严禁绕过 SDK 访问全局 Session。

---

## 5. 实时流（Streaming）实战：从内核到 UI

### 5.1 UI 路由对齐 (Root-ID Pattern)
在复杂的 UI（如多窗口、多 Bubble）中，前端需要知道一个 Token 属于谁。
AgentEngine 所有的 `AgentEvent` 均在根级别提供：
*   `agentId`: 产生该事件的 Agent。
*   `taskId`: 该事件关联的原子任务（对应前端的一个 Message Bubble）。
*   `step`: 当前执行到第几步。

**前端分发伪代码 (Electron/Web)**:
```typescript
window.ipc.on('agent-event', (event) => {
  const bubble = chatUI.findBubble(event.taskId);
  if (event.type === 'reasoning_chunk') {
    bubble.updateThinkingStream(event.payload.content);
  }
});
```

### 5.2 智能批处理 (Intelligent Batching)
为了防止高频事件（每秒 50+ 令牌）拖慢 Electron 渲染进程或造成网络拥堵，SDK 内部实现了：
*   **每 10 个令牌自动 Flush 一次**（平衡了 500ms 左右的感知延迟与系统负载）。
*   **微任务合并**: 针对非流式 Planner，自动在下一个事件循环滴答前合并所有同步输出。

---

## 6. 多智能体协作：分形代理与任务分发

### 6.1 分形委托 (Fractal Delegation)
子代理（Sub-agent）不是特殊的实体，它只是由父代理通过工具调用的另一个 `AgentEngine` 实例。

### 6.2 观察者继承 (Observer Inheritance)
这是解决“子代理工作时 UI 空白”的关键。
*   **原理**: 当父代理 Spawn 一个子代理时，SDK 会自动将当前的 `Observer` 传递下去。
*   **结果**: 子代理的所有“思考过程”会直接出现在父级的 `runStream` 中，UI 只需根据 `agentId` 自动切换显示即可。

---

## 7. 长期记忆管理：环境上下文与主动学习

### 7.1 被动检索 (Passive Retrieval)
每一轮对话前，SDK 自动调用 `MemoryManager.retrieve`，将相关背景（如用户偏好）注入 Planner 上下文。

### 7.2 主动写入 (The Tool Pattern)
**Agent 严禁直接访问数据库**。必须将“写入记忆”封装为工具，由 `Policy` 审计：
```typescript
// 推荐模式：闭包注入 MemoryManager
const memoryTool = createMemoryWriteTool(postgresMemory);
toolRegistry.register(memoryTool);
```

---

## 8. 生产环境硬化：持久化、安全与审批

### 8.1 强一致性持久化
生产环境必须使用 `PostgresStateStore`。
*   **断点恢复**: 如果服务器重启，只需调用 `engine.resume({ runId: "..." })`。
*   **完美审计**: `Reliable Track` 记录了每一条经过验证的指令，可用于合规性回溯。

### 8.2 人机协同 (HITL)
对于 `sideEffectLevel: "destructive"` 的工具，应在 `Policy` 中配置 `requireApproval: true`。
引擎会返回 `waiting_for_approval` 状态并挂起，直到调用 `resolveApproval`。

---

## 9. 测试、评估与质量保障

### 9.1 契约测试 (Conformance)
如果你实现了自定义的存储或 Planner，请务必运行契约测试：
```typescript
import { runStateStoreConformance } from "@agentic-kernel/conformance";
runStateStoreConformance(() => new MyCustomStore());
```

---

## 10. 常见问题与故障排查 (Troubleshooting)

*   **问题**: UI 上只显示“正在思考...”但文字不更新。
    *   **排查**: 检查 `AgentEvent` 根级别的 `taskId` 是否与前端组件绑定的 ID 一致。
*   **问题**: 复杂的任务跑一半报错 `PLANNER_STREAM_NO_ACTION`。
    *   **排查**: 这通常是因为模型输出被截断或 JSON 格式错误。建议在 `OpenAIPlanner` 中开启 `strict: true` 并使用高版本模型。

---

*AgentEngine 架构团队 荣誉出品*
