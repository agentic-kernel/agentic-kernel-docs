# Agentic Kernel SDK 使用文档

面向接入方和宿主服务的实用指南：如何把 AgentEngine 作为可复用 SDK 嵌入应用，而不是作为独立平台部署。

生成时间: 2026-05-29 23:01 CST
分支: `main`
提交: `f19bf3a`
仓库: `/Users/chengqing/Projects/github/agent-kernel`
来源: `docs/reports/2026-05-29-agent-engine-sdk-usage-guide.html`

## 目录

1. 定位与集成边界
2. 包选择
3. 安装与运行脚本
4. 最小可运行示例
5. HostContext 与租户边界
6. 工具接入
7. 策略与审批
8. 状态、恢复与回放
9. 记忆与压缩
10. 观测与 trace
11. OpenAI Planner
12. 多 Agent / Subagent
13. 分布式运行
14. 测试与 conformance
15. 生产接入清单
16. 小范围发布

## 1. 定位与集成边界

AgentEngine 是一个 TypeScript SDK。它提供自治 agent runtime kernel，但不拥有你的模型网关、数据库、队列、凭证、审批 UI、业务工具或多租户系统。

### SDK 负责

- Agent runtime loop
- structured Action / Observation
- tool execution governance
- state and execution log contracts
- memory injection and compaction helpers
- policy, approval, observer contracts
- multi-agent and distributed extension packages



### 宿主服务负责

- LLM provider credentials and routing
- domain tools and business permissions
- database migrations and connection pools
- approval UI and audit workflow
- queue workers and deployment topology
- tenant, workspace, user identity



### 推荐模式

- 先单 Agent 单任务接入
- 先使用 deterministic planner 做集成测试
- 再替换 OpenAIPlanner 或自定义 Planner
- 所有高风险工具先走 approval
- 生产环境必须接 durable StateStore



## 2. 包选择

| 包                               | 何时使用                                                              | 是否运行时必需 |
|---------------------------------|-------------------------------------------------------------------|---------|
| `@agentic-kernel/core`            | 所有宿主应用都需要。提供 runtime、contracts、state、tool、memory、policy、observer。 | 是       |
| `@agentic-kernel/model-openai`    | 使用 OpenAI Responses API 作为 Planner 时。                             | 否       |
| `@agentic-kernel/state-file`      | 本地服务、开发环境、文件持久化参考实现。                                              | 否       |
| `@agentic-kernel/state-postgres`  | 用 Postgres 持久化 AgentState 和 execution log。                        | 否       |
| `@agentic-kernel/memory-postgres` | 用 Postgres 保存长期记忆。                                                | 否       |
| `@agentic-kernel/observer-otel`   | 把 AgentEvent 导出到 OpenTelemetry-compatible tracer/meter。           | 否       |
| `@agentic-kernel/multi-agent`     | 需要自主生成、运行、join、cancel subagent 时。                                 | 否       |
| `@agentic-kernel/distributed`     | 需要 worker lease、retry、dead-letter、job runner 时。                   | 否       |
| `@agentic-kernel/evaluator`       | 比较不同 model planner，固定 scenarios，deterministic judges，`pass^k`，JSON/Markdown reports。 | 否       |
| `@agentic-kernel/testing`         | 宿主应用写测试时使用 fake 和 trace assertion。                                | 测试依赖    |
| `@agentic-kernel/conformance`     | 宿主实现自定义 adapter 时验证契约。                                            | 测试依赖    |



## 3. 安装与运行脚本

当前仓库是 monorepo workspace。发布到 registry 后，宿主应用按需安装对应包。
```
npm install @agentic-kernel/core
npm install @agentic-kernel/model-openai
npm install @agentic-kernel/state-postgres @agentic-kernel/memory-postgres
npm install @agentic-kernel/evaluator
npm install -D @agentic-kernel/testing @agentic-kernel/conformance
```

在本仓库内开发和验证:
```
npm install
npm run validate:sdk
npm run test:e2e
npm run test:e2e:faults
npm run test:e2e:live
npm run bench:sdk
```

## 4. 最小可运行示例

最小接入只需要 `Planner`、`PolicyManager`、`ToolRegistry`、`StateStore` 和 `HostContext`。
```
import {
  AgentEngine,
  BasicPolicy,
  InMemoryObserver,
  InMemoryStateStore,
  RuleBasedPlanner,
  ToolRegistry,
  type Agent,
  type Task,
  type HostContext
} from "@agentic-kernel/core";

const tools = new ToolRegistry();

tools.register({
  name: "echo",
  description: "Echo input text",
  inputSchema: {
    type: "object",
    properties: { text: { type: "string" } },
    required: ["text"],
    additionalProperties: false
  },
  outputSchema: {
    type: "object",
    properties: { echoed: { type: "string" } },
    required: ["echoed"],
    additionalProperties: false
  },
  sideEffectLevel: "none",
  requiredScopes: [],
  concurrencySafe: true,
  idempotent: true,
  execute: async (args) => ({ echoed: String(args.text) })
});

const planner = new RuleBasedPlanner({
  rules: [
    {
      when: (context) => context.recentActions.length === 0,
      action: () => ({
        id: "action-1",
        type: "tool_call",
        toolName: "echo",
        arguments: { text: "hello" },
        createdAt: new Date().toISOString()
      })
    }
  ],
  fallback: () => ({
    id: "final-1",
    type: "final_answer",
    content: "done",
    createdAt: new Date().toISOString()
  })
});

const observer = new InMemoryObserver();

const engine = new AgentEngine({
  planner,
  toolRegistry: tools,
  stateStore: new InMemoryStateStore(),
  policy: new BasicPolicy({
    maxSteps: 4,
    allowedTools: ["echo"]
  }),
  observer
});

const agent: Agent = {
  id: "agent-1",
  name: "Example Agent",
  role: "assistant",
  goal: "Run an echo tool and finish",
  instructions: "Use tools only when needed. Stop with final_answer.",
  tools: ["echo"],
  maxSteps: 4
};

const task: Task = {
  id: "task-1",
  input: "Echo hello"
};

const host: HostContext = {
  tenantId: "tenant-1",
  workspaceId: "workspace-1",
  principal: {
    id: "user-1",
    type: "user",
    tenantId: "tenant-1"
  },
  effectiveScopes: [],
  traceId: "trace-1"
};

const result = await engine.run({ agent, task, host });
console.log(result.status);
console.log(observer.events.map((event) => event.type));
```

## 5. HostContext 与租户边界

`HostContext` 是 SDK 复用的关键。AgentEngine 不猜测用户是谁、权限是什么、当前环境是否安全，所有这些都由宿主服务显式注入。
```
const host: HostContext = {
  tenantId: "tenant-1",
  workspaceId: "workspace-1",
  principal: {
    id: "service-user-1",
    type: "user",
    tenantId: "tenant-1",
    roles: ["researcher"]
  },
  effectiveScopes: ["tool.read", "memory.write"],
  credentialRefs: [
    {
      id: "cred-search",
      provider: "internal-search",
      scope: "read",
      tenantId: "tenant-1",
      workspaceId: "workspace-1"
    }
  ],
  traceId: "trace-abc",
  correlationId: "http-request-abc",
  idempotencyKey: "request-abc",
  environment: {
    id: "dev",
    kind: "local",
    tenantId: "tenant-1",
    workspaceId: "workspace-1"
  }
};
```

**规则:** 工具权限、memory scope、state ownership、trace attribution 都应该从 `HostContext` 推导。不要在工具 handler 内绕过 SDK 的 context 直接读全局用户状态。

## 6. 工具接入

工具是宿主应用的外设。每个工具必须声明 schema、side effect level、required scopes 和 handler。
```
tools.register({
  name: "create_ticket",
  description: "Create a ticket in the host issue system",
  inputSchema: {
    type: "object",
    properties: {
      title: { type: "string" },
      body: { type: "string" }
    },
    required: ["title", "body"],
    additionalProperties: false
  },
  outputSchema: {
    type: "object",
    properties: {
      ticketId: { type: "string" }
    },
    required: ["ticketId"],
    additionalProperties: false
  },
  sideEffectLevel: "write",
  requiredScopes: ["ticket.write"],
  timeoutMs: 10_000,
  concurrencySafe: false,
  idempotent: true,
  sanitizeInput: async (args) => ({
    title: String(args.title).trim(),
    body: String(args.body)
  }),
  beforeExecute: async (_args, context) => {
    context.reportProgress?.({ phase: "creating_ticket" });
  },
  execute: async (args, context) => {
    const ticketId = await hostTicketClient.create({
      title: String(args.title),
      body: String(args.body),
      idempotencyKey: context.idempotencyKey
    });
    return { ticketId };
  }
});
```

建议把工具按风险分级:

| 级别            | sideEffectLevel | 策略                            |
|---------------|-----------------|-------------------------------|
| 纯读/计算         | `none` 或 `read` | 可在 headless/auto 模式下允许，适合并发调度 |
| 业务写入          | `write`         | 需要 scope，通常需要幂等键，关键写入建议审批     |
| 删除/发布/支付/生产系统 | `destructive`   | 必须审批，并保留审计记录                  |



## 7. 策略与审批

`BasicPolicy` 适合 MVP 和测试。生产服务通常在它外面包一层 `PolicyPipeline` 或自定义 `PolicyManager`。
```
const policy = new BasicPolicy({
  maxSteps: 8,
  maxFailures: 3,
  allowedTools: ["search", "create_ticket"],
  requireApprovalForTools: ["create_ticket"],
  maxChildRuns: 3,
  maxDelegationDepth: 1,
  requireApprovalForDelegation: false
});
```

处理审批结果:
```
const first = await engine.run({ agent, task, host });

if (first.status === "waiting_for_approval") {
  const approved = await engine.resolveApproval({
    agent,
    task,
    host,
    runId: first.state.runId,
    approvalId: first.approvalRequest.id,
    decision: "approved",
    approver: host.principal
  });

  console.log(approved.status);
}
```

**生产建议:** 审批 UI、通知、超时、审批人权限和审计归档应由宿主服务实现。SDK 负责 pending action、approval id、resume safety 和事件记录。

## 8. 状态、恢复与回放

生产环境不要使用 `InMemoryStateStore`。至少使用 file-backed store，服务端建议接 Postgres 或自定义 durable store。
```
import { FileStateStore } from "@agentic-kernel/state-file";

const stateStore = new FileStateStore({
  directory: "/var/lib/my-service/agent-runs"
});

const engine = new AgentEngine({
  planner,
  policy,
  toolRegistry,
  stateStore
});
```

恢复运行:
```
const resumed = await engine.resume({
  agent,
  task,
  host,
  runId: "run-123"
});
```

回放用于审计和恢复检查:
```
import { replayAgentStateFromLog } from "@agentic-kernel/core";

const log = await stateStore.listLog("run-123");
const replayedState = replayAgentStateFromLog(log);
```

## 9. 记忆与压缩

MemoryManager 是接口，不绑定向量库。你可以先用 `InMemoryMemory`，生产接 Postgres、vector DB 或内部知识库。
```
import { InMemoryMemory, ContextCompactor, SessionMemoryExtractor } from "@agentic-kernel/core";

const memory = new InMemoryMemory();

await memory.write({
  id: "memory-1",
  scope: "tenant-1/workspace-1/agent-1",
  kind: "fact",
  content: "The user prefers concise engineering reports.",
  createdAt: new Date().toISOString(),
  confidence: 0.9
});

const engine = new AgentEngine({
  planner,
  policy,
  toolRegistry,
  memory,
  stateStore
});
```

从 session 中抽取长期记忆:
```
const extractor = new SessionMemoryExtractor({
  minObservations: 1,
  summarize: async () => ({
    kind: "summary",
    content: "User approved creating SDK delivery reports.",
    confidence: 0.8
  })
});

const result = await extractor.extract({
  agent,
  task,
  host,
  state,
  sourceEventIds: log.map((entry) => entry.id)
});
```

## 10. 观测与 trace

Observer 接收 runtime 事件。开发时可使用 `InMemoryObserver`，生产可接 OpenTelemetry adapter。
```
import { OpenTelemetryObserver } from "@agentic-kernel/observer-otel";

const observer = new OpenTelemetryObserver({
  tracer,
  meter
});

const engine = new AgentEngine({
  planner,
  policy,
  toolRegistry,
  stateStore,
  observer
});
```

关键事件包括: `task_started`、`context_built`、`action_selected`、`policy_decision`、`approval_requested`、`tool_started`、`tool_completed`、`observation_recorded`、`task_completed`。

## 11. OpenAI Planner

OpenAI adapter 只负责把 `AgentContext` 转成 Responses API 请求，并把模型输出验证为结构化 SDK Action。模型路由、API key、base URL 由宿主服务注入。
```
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

const planner = new OpenAIPlanner({
  apiKey: process.env.OPENAI_API_KEY!,
  model: process.env.AGENT_ENGINE_OPENAI_MODEL ?? "gpt-4.1",
  baseUrl: process.env.OPENAI_BASE_URL,
  fetch: globalThis.fetch
});

const engine = new AgentEngine({
  planner,
  policy,
  toolRegistry,
  stateStore,
  memory,
  observer
});
```

**规则:** Planner 必须返回结构化 `Action`。不要让宿主服务从自由文本中解析工具调用。

## 12. 多 Agent / Subagent

单 Agent runtime 是底座。多 Agent 能力通过 `@agentic-kernel/multi-agent` 注入，不污染 core。
```
import { SubagentDelegationManager } from "@agentic-kernel/multi-agent";

const childEngine = new AgentEngine({
  planner: childPlanner,
  policy: childPolicy,
  toolRegistry,
  stateStore: childStateStore
});

const delegationManager = new SubagentDelegationManager({
  engine: {
    run: (input) => childEngine.run(input)
  }
});

const parentEngine = new AgentEngine({
  planner: parentPlanner,
  policy: new BasicPolicy({
    maxSteps: 8,
    allowedTools: ["search"],
    maxChildRuns: 3,
    maxDelegationDepth: 1
  }),
  toolRegistry,
  stateStore: parentStateStore,
  delegationManager
});
```

Planner 可以返回 `spawn_subagent`、`join_subagent`、`cancel_subagent`。每个 child run 仍然使用同一套 Action、Observation、Policy、Memory、Trace 合约。

## 13. 分布式运行

`@agentic-kernel/distributed` 提供 scheduler 和 job runner 的参考实现。它适合把长任务、审批恢复、记忆抽取、compaction、tool reconciliation 放进 worker。
```
import { InMemoryDistributedScheduler, RuntimeJobRunner } from "@agentic-kernel/distributed";

const scheduler = new InMemoryDistributedScheduler();

await scheduler.enqueue({
  id: "job-1",
  type: "continue_run",
  runId: "run-1",
  expectedStateVersion: 1,
  attempt: 0,
  notBefore: new Date().toISOString(),
  deadline: new Date(Date.now() + 60_000).toISOString(),
  idempotencyKey: "job-1",
  traceId: "trace-1"
});

const runner = new RuntimeJobRunner({
  scheduler,
  workerId: "worker-1",
  leaseMs: 30_000,
  maxAttempts: 3,
  backoffMs: 1_000,
  runtime: {
    step: (input) => engine.step(input),
    resolveApproval: (input) => engine.resolveApproval(input)
  },
  resolver: async (job) => ({
    agent,
    task,
    host,
    runId: job.runId,
    currentStateVersion: job.expectedStateVersion
  })
});

await runner.runNext();
```

## 14. 测试与 conformance

宿主服务接入 SDK 时，建议三层测试。

| 层级       | 目标                                                                  | 推荐包                         |
|----------|---------------------------------------------------------------------|-----------------------------|
| Unit     | 测试自定义 tool、policy、planner wrapper、memory mapper。                    | `@agentic-kernel/testing`     |
| Contract | 验证自定义 StateStore、MemoryManager、Observer、Tool、Scheduler 是否符合 SDK 契约。 | `@agentic-kernel/conformance` |
| E2E      | 用真实宿主依赖跑一个完整 run，包括审批、恢复、trace 和审计。                                 | 宿主应用测试 + SDK scenarios 参考   |



本仓库的交付验证:
```
npm run validate:sdk
```

该命令覆盖 workspace tests、typecheck、build、coverage、exports、audit、diff check、deterministic E2E 和 benchmark gate。

### Model planner evaluation

`@agentic-kernel/evaluator` 是运行时可选包，用于评估和回归比较不同 Planner。它采用 OpenAI-Evals/Inspect-style scenario runner、BFCL-style tool trajectory checks、tau-bench-style `pass^k` reliability、HELM-style multi-metric report，以及 MLCommons-style safety/policy metrics。

SDK 方式可以用固定场景比较两个 OpenAI 模型:

```ts
import { OpenAIPlanner } from "@agentic-kernel/model-openai";
import { DeterministicJudge, runEvalSuite } from "@agentic-kernel/evaluator";

const report = await runEvalSuite(
  {
    id: "planner-regression",
    name: "Planner regression suite",
    scenarios: [
      {
        id: "refund-policy",
        name: "Refund policy answer",
        agent: {
          id: "support-agent",
          name: "Support Agent",
          role: "support",
          goal: "Resolve customer requests within policy",
          instructions: "Use tools only when required by the task.",
          tools: []
        },
        task: {
          id: "task-refund",
          input: "Can this customer receive a refund for order A123?"
        },
        expected: {
          status: "completed",
          finalAnswerIncludes: ["refund"],
          maxToolCalls: 2,
          denyPolicyViolations: true
        }
      }
    ],
    models: [
      {
        id: "gpt-4.1-mini",
        provider: "openai",
        model: "gpt-4.1-mini",
        createPlanner: () =>
          new OpenAIPlanner({
            apiKey: process.env.OPENAI_API_KEY ?? "",
            model: "gpt-4.1-mini"
          })
      },
      {
        id: "gpt-4.1",
        provider: "openai",
        model: "gpt-4.1",
        createPlanner: () =>
          new OpenAIPlanner({
            apiKey: process.env.OPENAI_API_KEY ?? "",
            model: "gpt-4.1"
          })
      }
    ],
    judges: [new DeterministicJudge()]
  },
  { trials: 5 }
);
```

CLI 方式生成 JSON 和 Markdown 报告:

```bash
agent-engine-eval run --config ./agent-engine.eval.mjs --json ./eval/report.json --markdown ./eval/report.md --trials 5
```

## 15. 生产接入清单

### 必做

- 使用 durable StateStore
- 所有 write/destructive 工具配置 approval
- 配置 maxSteps、maxFailures、maxChildRuns
- 每个 request 注入 HostContext
- 接入 Observer 到日志/trace/metrics
- 用 conformance 验证自定义 adapter



### 建议

- 所有工具 handler 支持 idempotencyKey
- 工具返回大对象走 artifact 引用
- 长期记忆采用 proposal/approval 写策略
- Planner 输出启用严格 schema
- 分布式 worker 记录 dead-letter 原因



### 不要做

- 不要在工具中绕过 HostContext 授权
- 不要从自由文本解析工具调用
- 不要让 LLM 自己决定高风险动作是否审批
- 不要在生产只用 InMemoryStateStore
- 不要把 SDK 改造成全功能平台



**推荐接入顺序:** deterministic planner + fake tools -> real tools with BasicPolicy -> durable StateStore -> observer -> approval UI -> real Planner -> memory -> multi-agent -> distributed worker。

## 16. 小范围发布

小范围使用不建议一开始公开发布到 npm latest。推荐先用 private registry 或 tarball canary，让 1-3 个宿主服务接入，收集 API 反馈后再决定是否公开发布。

### 推荐路线

| 阶段             | 方式                                    | 适用场景                                        | 限制                                                          |
|----------------|---------------------------------------|---------------------------------------------|-------------------------------------------------------------|
| Canary         | `npm pack` tarball                    | 1 个服务、本机或内部共享路径试用，验证 API 和打包内容              | 不适合多服务长期使用；lockfile 会记录 tarball 路径                          |
| Internal alpha | Verdaccio / 私有 npm registry           | 小团队、内网、包名不想改、需要正常 semver 和 dist-tag         | 需要维护 registry 和访问控制                                         |
| Harbor alpha   | Harbor OCI artifact + ORAS            | 已有内网 Harbor，希望先小范围分发 tgz 包，不新增 npm registry | 不是原生 npm registry；消费方需要先 `oras pull` 再 `npm install` 本地 tgz |
| Private org    | npm private package 或 GitHub Packages | 已有 npm/GitHub 组织和 token 管理流程                | scope 必须可控；GitHub Packages 通常要求 package scope 与 owner 匹配    |
| Public beta    | 公开 npm + `alpha`/`beta` dist-tag      | 准备让外部试用，但还不设为 latest                        | 代码和 API 立即公开，不适合私密评估                                        |



**建议:** 如果内网已有 Harbor，则先走 tarball canary 验证打包内容，再把 tarball 作为 OCI artifact 推到 Harbor 发布 `0.1.0-alpha.0`。需要 npm 原生安装体验时，再补 Verdaccio 或其它 private npm registry。不要直接发布 `latest`。

### 发布包清单

建议发布这些包:

- `@agentic-kernel/core`
- `@agentic-kernel/model-openai`
- `@agentic-kernel/memory-postgres`
- `@agentic-kernel/state-file`
- `@agentic-kernel/state-postgres`
- `@agentic-kernel/observer-otel`
- `@agentic-kernel/multi-agent`
- `@agentic-kernel/distributed`
- `@agentic-kernel/evaluator` 作为可选评估包
- `@agentic-kernel/testing` 和 `@agentic-kernel/conformance` 作为测试/开发支持包



不要发布:

- `@agentic-kernel/sdk-validation`: 私有交付验证 workspace
- `@agentic-kernel/example-embedded-service`: 示例应用



### 发布前必须补齐

当前包已经有 `main`、`types` 和 `exports`，但正式发布前建议每个可发布 package 增加:
```
{
  "files": [
    "dist",
    "README.md",
    "package.json"
  ],
  "sideEffects": false,
  "publishConfig": {
    "access": "restricted"
  }
}
```

如果使用 GitHub Packages 或内部 registry，还需要配置 registry:
```
{
  "publishConfig": {
    "registry": "https://npm.pkg.github.com",
    "access": "restricted"
  }
}
```

如果 `@agent-engine` scope 不属于你的 npm/GitHub 组织，先创建对应组织或改成内部可控 scope，例如 `@your-org/agent-engine-core`。

### Canary tarball 发布

先在 SDK 仓库中完成验证和打包:
```
npm ci
npm run validate:sdk

mkdir -p artifacts/npm
npm pack --workspace @agentic-kernel/core --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/model-openai --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/state-file --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/state-postgres --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/memory-postgres --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/observer-otel --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/multi-agent --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/distributed --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/testing --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/conformance --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/evaluator --pack-destination artifacts/npm
```

消费方服务用 tarball 安装所需包。需要把依赖包放在同一条 install 命令里，确保 workspace 内部依赖能被本地 tarball 满足:
```
npm install \
  /path/to/artifacts/npm/agent-engine-core-0.1.0.tgz \
  /path/to/artifacts/npm/agent-engine-model-openai-0.1.0.tgz \
  /path/to/artifacts/npm/agent-engine-state-postgres-0.1.0.tgz
```

### Harbor 内网发布

现有 Harbor 地址是 `https://harbor.plab.work/harbor/projects`。这是 Harbor UI 地址；OCI registry endpoint 使用主机名 `harbor.plab.work`。建议在 Harbor UI 中创建一个项目，例如 `agent-engine`，然后用 ORAS 推送 npm tarball artifact。

本机已使用 `oras` 完成拆包发布验证。新机器如果没有安装 ORAS，macOS 可安装:
```
brew install oras
```

登录 Harbor。生产环境建议安装内网 CA，不建议长期使用 insecure 选项:
```
export HARBOR_REGISTRY=harbor.plab.work
export HARBOR_PROJECT=agent-engine
export AGENT_ENGINE_VERSION=0.1.0-alpha.0

oras login "$HARBOR_REGISTRY" -u "$HARBOR_USER" --password-stdin
```

仓库内已提供脚本化入口。默认 Harbor project 是 `agent-engine`，默认 registry 是 `harbor.plab.work`。默认发布方式是拆包发布，每个 npm package 对应一个 Harbor artifact:
```
npm run release:pack

export HARBOR_USER=your-user
export HARBOR_PASSWORD=your-password-or-robot-token
export AGENT_ENGINE_RELEASE_TAG=0.1.0-alpha.0
npm run release:harbor
```

拆包发布目标示例:
```
harbor.plab.work/agent-engine/npm/core:0.1.0-alpha.0
harbor.plab.work/agent-engine/npm/model-openai:0.1.0-alpha.0
harbor.plab.work/agent-engine/npm/state-postgres:0.1.0-alpha.0
harbor.plab.work/agent-engine/npm/evaluator:0.1.0-alpha.0
```

如需一次性发布 bundle，可运行:
```
npm run release:harbor:bundle
npm run release:harbor:all
```

注意: `AGENT_ENGINE_RELEASE_TAG` 是 Harbor artifact tag。当前 package.json 版本仍是 `0.1.0`，因此 tarball 文件名和消费方 npm 解析出的包版本仍是 `0.1.0`。如果希望消费方 lockfile 也体现 alpha 版本，需要先统一 bump workspace package version。

生成 npm tarball:
```
npm ci
npm run validate:sdk

mkdir -p artifacts/npm
npm pack --workspace @agentic-kernel/core --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/model-openai --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/state-file --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/state-postgres --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/memory-postgres --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/observer-otel --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/multi-agent --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/distributed --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/testing --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/conformance --pack-destination artifacts/npm
npm pack --workspace @agentic-kernel/evaluator --pack-destination artifacts/npm
```

方式 A: 拆包推送，每个 package 单独一个 artifact，推荐作为默认发布方式:
```
npm run release:harbor
```

方式 B: 推一个 SDK bundle artifact，适合一次性拉全包:
```
oras push "$HARBOR_REGISTRY/$HARBOR_PROJECT/npm/sdk-bundle:$AGENT_ENGINE_VERSION" \
  --artifact-type application/vnd.agent-engine.npm-bundle.v1 \
  artifacts/npm/agent-engine-core-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-model-openai-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-state-file-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-state-postgres-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-memory-postgres-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-observer-otel-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-multi-agent-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-distributed-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-testing-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-conformance-0.1.0.tgz:application/vnd.npm.package.tgz \
  artifacts/npm/agent-engine-evaluator-0.1.0.tgz:application/vnd.npm.package.tgz
```

等价的单包 ORAS 命令示例:
```
oras push "$HARBOR_REGISTRY/$HARBOR_PROJECT/npm/core:$AGENT_ENGINE_VERSION" \
  --artifact-type application/vnd.agent-engine.npm-package.v1 \
  artifacts/npm/agent-engine-core-0.1.0.tgz:application/vnd.npm.package.tgz

oras push "$HARBOR_REGISTRY/$HARBOR_PROJECT/npm/model-openai:$AGENT_ENGINE_VERSION" \
  --artifact-type application/vnd.agent-engine.npm-package.v1 \
  artifacts/npm/agent-engine-model-openai-0.1.0.tgz:application/vnd.npm.package.tgz
```

消费方按需拉取单包:
```
mkdir -p vendor/agent-engine
oras pull harbor.plab.work/agent-engine/npm/core:0.1.0-alpha.0 -o vendor/agent-engine
oras pull harbor.plab.work/agent-engine/npm/model-openai:0.1.0-alpha.0 -o vendor/agent-engine

npm install \
  ./vendor/agent-engine/agent-engine-core-0.1.0.tgz \
  ./vendor/agent-engine/agent-engine-model-openai-0.1.0.tgz
```

消费方也可以从 Harbor 拉取 bundle:
```
export HARBOR_REGISTRY=harbor.plab.work
export HARBOR_PROJECT=agent-engine
export AGENT_ENGINE_VERSION=0.1.0-alpha.0

oras login "$HARBOR_REGISTRY" -u "$HARBOR_USER" --password-stdin
mkdir -p vendor/agent-engine
oras pull "$HARBOR_REGISTRY/$HARBOR_PROJECT/npm/sdk-bundle:$AGENT_ENGINE_VERSION" \
  -o vendor/agent-engine

npm install \
  ./vendor/agent-engine/agent-engine-core-0.1.0.tgz \
  ./vendor/agent-engine/agent-engine-model-openai-0.1.0.tgz \
  ./vendor/agent-engine/agent-engine-state-postgres-0.1.0.tgz
```

如果消费方项目要提交 lockfile，建议不要提交 `vendor/agent-engine/*.tgz` 到业务仓库；用 CI 在 install 前从 Harbor 拉取，并在 lockfile 中固定版本或 artifact digest。

上一轮已发布的内网 alpha 拆包 artifacts:
```
harbor.plab.work/agent-engine/npm/conformance:0.1.0-alpha.0
sha256:6f67687d0cb8f16ed106c76f4a9386bd372021237ff444ae4529aa38e7cc64e2

harbor.plab.work/agent-engine/npm/core:0.1.0-alpha.0
sha256:1765a6bcb66e76ed9464d9fb568c03339f3c148a4ba263d0c5a368ea389ea17e

harbor.plab.work/agent-engine/npm/distributed:0.1.0-alpha.0
sha256:bf0929a074ef1c4ae510a035547ee106afe1a1b2b14c5bd785a821dacaeadc3a

harbor.plab.work/agent-engine/npm/memory-postgres:0.1.0-alpha.0
sha256:88a44f2370b7d5e924eb778b5608c6c98035cf098f342ea2c2d455f15ab14660

harbor.plab.work/agent-engine/npm/model-openai:0.1.0-alpha.0
sha256:551bfb6d29aafde047aca02eeaaca418770a00092b5cad9ce0d0d30059a73607

harbor.plab.work/agent-engine/npm/multi-agent:0.1.0-alpha.0
sha256:4aaf4db61e9121fe6fafff33efcc41941c9258a7357cac6c65e9675f46a24a70

harbor.plab.work/agent-engine/npm/observer-otel:0.1.0-alpha.0
sha256:5b8692dd89d04993dcc5def670db113f71e225cd2b1d9c9da515f1c563563ad4

harbor.plab.work/agent-engine/npm/state-file:0.1.0-alpha.0
sha256:a4a856cfd4d5a8be3b6640812729fee32555978caf889db666e719c7ff90bde8

harbor.plab.work/agent-engine/npm/state-postgres:0.1.0-alpha.0
sha256:58c341ed8ed00eb9cbeb4a87b2e255d65823c500056659ab50e69bc2e922e69d

harbor.plab.work/agent-engine/npm/testing:0.1.0-alpha.0
sha256:1aeacaca56af8b20474f2bccedcf355db13ce8d6e32810c88e019363a734a413
```

本分支新增 `@agentic-kernel/evaluator`，`npm run release:pack` 已生成本地 tarball。执行 `npm run release:harbor` 后补录 Harbor digest:
```
harbor.plab.work/agent-engine/npm/evaluator:0.1.0-alpha.0
sha256:<publish-output-digest>
```

上一轮辅助 bundle 也已发布，可用于一次性拉全包；重新发布包含 evaluator 的 bundle 后需要更新 digest:
```
harbor.plab.work/agent-engine/npm/sdk-bundle:0.1.0-alpha.0
sha256:6207f6d475b4b82cf05a71cbb334ac8cda42f12d9c3d13712657bf06584b911c
```

### Private registry 发布

使用 Verdaccio 或私有 npm registry 时，推荐发布到 `alpha` tag:
```
npm run validate:sdk

npm publish --workspace @agentic-kernel/core --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/model-openai --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/state-file --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/state-postgres --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/memory-postgres --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/observer-otel --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/multi-agent --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/distributed --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/evaluator --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/testing --registry http://localhost:4873 --tag alpha
npm publish --workspace @agentic-kernel/conformance --registry http://localhost:4873 --tag alpha
```

消费方配置 `.npmrc`:
```
@agent-engine:registry=http://localhost:4873
//localhost:4873/:_authToken=${NPM_TOKEN}
```

然后安装:
```
npm install @agentic-kernel/core@alpha
npm install @agentic-kernel/model-openai@alpha
npm install @agentic-kernel/state-postgres@alpha @agentic-kernel/memory-postgres@alpha
```

### 版本策略

小范围试用建议使用 prerelease 版本:
```
npm version 0.1.0-alpha.0 --workspaces --no-git-tag-version
```

每次破坏性 API 调整递增 prerelease:
```
npm version prerelease --workspaces --preid alpha --no-git-tag-version
```

稳定后再发布 `0.1.0` 并切换 dist-tag:
```
npm dist-tag add @agentic-kernel/core@0.1.0 latest
```

### 消费方验收脚本

每个小范围接入服务至少跑以下验收:
```
npm install
npm run typecheck
npm test
node ./scripts/agent-engine-smoke-test.js
```

smoke test 应该覆盖:

- 创建一个 AgentEngine
- 注册一个 read-only tool
- 执行一个 deterministic planner run
- 验证 state 已保存
- 验证 observer 收到 `task_started` 和 `task_completed`
- 如果启用审批，验证 `waiting_for_approval` 和 `resolveApproval()`



### 小范围发布准入标准

| 检查项    | 要求                                                        |
|--------|-----------------------------------------------------------|
| SDK 验证 | `npm run validate:sdk` 通过                                 |
| 打包检查   | `npm pack --dry-run` 确认只包含 `dist`、README、package metadata |
| 安装检查   | 在一个空消费方项目中安装并能 import                                     |
| 类型检查   | 消费方 TypeScript 编译通过                                       |
| 运行检查   | 消费方 smoke run 能完成、失败、审批、trace 至少各覆盖一个路径                   |
| 回滚     | 保留上一个 alpha tag，并能在消费方锁回旧版本                               |



## 相关文档

- [测试报告总览](../reports/index.md)
- [架构总览](../architecture.md)
- [agentic-kernel 源码仓库](https://github.com/agentic-kernel/agentic-kernel)
