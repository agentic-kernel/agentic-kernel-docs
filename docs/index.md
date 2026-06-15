# Agentic Kernel

A **microkernel agent runtime**: a minimal, governed "plan → authorize → execute →
observe" loop with everything pluggable behind injection seams. Available as two
feature-paired SDKs:

- **TypeScript** — `@agentic-kernel/*` monorepo (core + adapters)
- **Python** — `agentic-kernel` (single distribution; `agentic_kernel.core` + adapters)

Both SDKs are at **v0.5.0** and kept at feature parity.

## Why a kernel?

Most agent frameworks bake the harness — model calls, memory, policy, retries,
logging — directly into the loop. Agentic Kernel keeps the loop **minimal and
governed**, and composes the harness at well-defined seams instead. That buys:

- **Reproducibility** — every run is an immutable, replayable execution log.
- **Governance** — policy, approvals, budgets, and capability/credential checks are
  first-class seams, not scattered conditionals.
- **Bounded autonomy** — a hard iteration ceiling and stall→reflect recovery are
  built into the kernel; no run is unbounded.
- **Portability** — swap the model, store, memory, or observability backend by
  swapping an adapter; the kernel never changes.
- **Testability** — contract test suites (`conformance`) prove any adapter is a safe
  substitute.

## Start here

- **[Architecture](architecture.md)** — the run loop, data model, and the injection seams.
- **[Getting started](guides/getting-started.md)** — first agent in either SDK.
- **[Consuming packages](guides/consuming-packages.md)** — install from the registry.
- **Guides:** [Memory](guides/memory.md) · [On-device models](guides/on-device.md) · [Multi-agent orchestration](guides/multi-agent.md)
- **Manuals:** [Developer guide](manuals/developer-guide.md) · [Comprehensive manual](manuals/manual.md)
- **[Development records](reports/index.md)** — real-LLM evaluation & stress campaigns.

## Status

Packages are currently **private / publishing paused** (Apache-2.0, intended for an
eventual public release). This documentation site is public.
