# 架构

Agentic Kernel 是一个**微内核**:核心只跑一个有界的「规划 → 鉴权 → 执行 → 观测」
循环,其余可变部分都是注入接缝。两个 SDK 互为镜像(TS interface ↔ Python `@dataclass`
+ `Protocol`;camelCase ↔ snake_case;Promise ↔ `async/await`)。

## 分层

```
集成层    sdk-validation(组装全部适配器,e2e + benchmark)
            │
适配器层   planner: model-openai / model-ondevice
(仅依赖    memory:  memory-postgres / EmbeddingMemory
 core)     state:   state-file / state-postgres
           observer: observer-otel
           扩展:    multi-agent(委派 + 编排) / distributed
            │
质量层    testing(fakes) · conformance(契约套件) · evaluator(pass^k)
            │
微内核    core — 唯一外部依赖是 JSON-Schema 校验器
           运行时 + 契约 + 约 12 个接缝 + 内存级参考实现
```

**不变量:** core 不依赖任何适配器;每个适配器只依赖 core。任何实现都可替换,且
`conformance` 契约套件证明替换是安全的。

## 运行循环

`createAgentEngine(options)` 返回带 `run` / `step` / `resume` / `runStream` 的
`AgentEngine`。每一步:

1. **检查中止** 信号。
2. **反思**(停滞时,`onNonProgress: "reflect"` 注入纠偏消息)。
3. **检查预算** — `RunBudget` 各维度;即使未配预算,也有硬上限
   `KERNEL_DEFAULT_MAX_ITERATIONS`(1000)兜底。
4. **压缩** 上下文(若配置了 compactor)。
5. **构建上下文** — 检索记忆、注入种子记忆、派生目标、解析能力。
6. **规划** — planner 返回单个 `Action`、并行工具批次,或流式。
7. **鉴权** — 策略返回 `allow | deny | stop | require_approval`。
8. **路由** — 按「决策 × 动作类型」:工具调用(经调度器)、委派、思考/目标、
   `final_answer`(→ 完成)、`ask_user` / `schedule`(→ 等待)、stop/deny(→ 终态)。

这把临时 harness 里散落各处的逻辑收敛成一条可审计的状态机。

## 数据模型——三个概念

| 概念 | 是什么 | 关键点 |
|---|---|---|
| **AgentState** | 持久化运行状态 | 含 `variables`(宿主草稿,**无法**从日志重建);乐观锁 `version` |
| **ExecutionLogEntry** | 不可变审计事件流 | 每个 run 内单调递增的 `sequence`;25+ 事件类型;先落盘再 resolve |
| **AgentContext** | 每步为 planner 构建的视图 | 检索到的记忆、可用工具、目标、剩余预算;不持久化 |

**状态 ≠ 日志。** `replayAgentStateFromLog()` 只读重建动作/观测/状态(并剔除被
拒绝/停止的动作以对齐 live 状态),但无法重建 `variables`——所以恢复运行始终从
`StateStore` 加载,而非用 replay。

## 注入接缝

每个接缝都是一个接口/`Protocol`,内核自带参考实现,另有生产适配器:

| 接缝 | 参考实现 | 生产适配器 |
|---|---|---|
| Planner | FakePlanner / RuleBasedPlanner | model-openai、model-ondevice |
| PolicyManager | BasicPolicy / PermissionPolicy / PolicyPipeline | + 风险分类器 |
| ToolRegistry / ToolScheduler | 内存实现 + eval-free 校验器 | 注入 ajv 等 |
| MemoryManager | Noop / InMemory / **EmbeddingMemory** | memory-postgres |
| StateStore | InMemoryStateStore | state-file、state-postgres |
| Observer | Noop / InMemory / Console / Streaming / Composite | observer-otel |
| DelegationManager | (宿主特定) | multi-agent |
| ApprovalProvider | InMemoryApprovalStore | 宿主传输 |
| Clock / IdGenerator | 默认 | 注入以获得确定性 |
| Compactor / MemoryGovernor | SessionMemoryGovernor | 宿主策略 |
| CapabilityRegistry | InMemoryCapabilityRegistry | — |

## 有界自治

预算在内核**内部**于每轮规划前强制执行——`maxIterations`、`maxToolCalls`、
`maxFailures`、`maxConsecutiveNonProgress`、`maxGoalDepth`、`maxWallClockMs`——
停滞的 planner 会触发反思,且每轮放宽额度。没有无界循环。

## 内核之上的扩展

- **multi-agent** — `SubagentDelegationManager` + `TaskGraphCoordinator`,以及在
  `AgentRunner` 接缝上的可复用编排模式(`runParallel` / `mapReduce` / `vote` /
  `plannerWorkerCritic`)。见[多 Agent 编排](guides/multi-agent.md)。
- **distributed** — 持久任务队列:独占租约、重试/退避、幂等副作用、死信检视。

以上全部都用真实模型做了端到端验证——见[开发记录](reports/index.md)。
