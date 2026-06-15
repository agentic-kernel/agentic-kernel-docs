# 开发记录

真实 LLM 的评测、能力与压测记录——每条结论都有对真实模型(远端网关 + 端侧 Ollama)的
实跑支撑,以 `pass^k` 与有据的过程校验评分。

> 报告正文为英文(开发记录原文存档)。

- [2026-06-15 — 真实 LLM 综合测试(TS + Python)](2026-06-15-comprehensive-real-llm-campaign.md) — 17 个领域覆盖、功能 pass^k、两 SDK 各 9/9 流程、并发压测零错误、跨模型对比。
- [2026-06-13 — 完整测试报告](2026-06-13-agent-kernel-full-test-report.md) — 战役汇总、发现、修复与边界。
- [2026-06-13 — 远端大模型框架压测](2026-06-13-remote-model-framework-stress.md) — 450 episode、并发、策略/记忆/replay。
- [2026-06-12 — 端侧测试战役](2026-06-12-ondevice-testing-campaign.md) — 端侧六阶段总报告。
- [2026-06-11 — 端侧能力与可靠性](2026-06-11-ondevice-capability-reliability.md) — plan-then-execute + 投票 → 100%。
- [2026-06-10 — 端侧长跑](2026-06-10-model-ondevice-longrun.md) — 22 分钟、194 episode 多步运行。
- [2026-06-09 — 端侧性能](2026-06-09-model-ondevice-perf.md) — 5 个本地模型的单步延迟/修复。
- [2026-05-30 — 官方基准运行](2026-05-30-official-agent-benchmark-run.md) — BFCL / tau-bench / tau2。
