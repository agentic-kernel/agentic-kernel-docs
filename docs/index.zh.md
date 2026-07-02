# Agentic Kernel

一个**微内核 Agent 运行时**:核心只跑一个受治理的最小循环「规划 → 鉴权 → 执行 →
观测」,其余一切都通过注入接缝(seam)可替换。提供两个**功能对等**的 SDK:

- **TypeScript** — `@agentic-kernel/*` monorepo(core + 适配器)
- **Python** — `agentic-kernel`(单分发包;`agentic_kernel.core` + 适配器)

两个 SDK 均为 **v0.7.0**,保持功能对等。

## 为什么是「内核」?

多数 Agent 框架把 harness(模型调用、记忆、策略、重试、日志)直接写进循环里。Agentic
Kernel 把循环保持**最小且受治理**,把 harness 组合在明确的接缝上,从而获得:

- **可复现** — 每次运行都是不可变、可重放的执行日志。
- **可治理** — 策略、审批、预算、能力/凭证检查都是一等接缝,而非散落的分支判断。
- **有界自治** — 硬性迭代上限 + 停滞→反思恢复内建于内核,没有无界运行。
- **可移植** — 换模型、存储、记忆或可观测后端只需换适配器,内核不变。
- **可测试** — 契约测试套件(`conformance`)证明任意适配器都是安全替换。

## 从这里开始

- **[架构](architecture.md)** — 运行循环、数据模型与注入接缝。
- **[快速上手](guides/getting-started.md)** — 在任一 SDK 写出第一个 agent。
- **[安装与引入](guides/consuming-packages.md)** — 从仓库安装。
- **指南:** [记忆](guides/memory.md) · [端侧模型](guides/on-device.md) · [多 Agent 编排](guides/multi-agent.md) · [Token 用量](guides/token-usage.md)
- **手册:** [开发者指南](manuals/developer-guide.md) · [完整手册](manuals/manual.md)
- **[开发记录](reports/index.md)** — 真实 LLM 评测与压测报告。

## 状态

**公开**且 **Apache-2.0**:TypeScript 在 **npmjs**(`@agentic-kernel/*`),Python 在
**PyPI**(`agentic-kernel-runtime`;导入包名 `agentic_kernel`)。见
[安装](guides/consuming-packages.md)。
