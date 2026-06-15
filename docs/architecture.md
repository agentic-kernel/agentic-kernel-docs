# Architecture

Agentic Kernel is a **microkernel**: the core runs one bounded "plan → authorize →
execute → observe" loop, and every variable part is an injected seam. The two SDKs
mirror each other (TypeScript interfaces ↔ Python `@dataclass` + `Protocol`; camelCase
↔ snake_case; Promises ↔ `async/await`).

## Layering

```
Integration   sdk-validation (compose all adapters, e2e + benchmarks)
                  │
Adapters      planner: model-openai / model-ondevice
(depend only   memory:  memory-postgres / EmbeddingMemory
 on core)      state:   state-file / state-postgres
               observer: observer-otel
               extensions: multi-agent (delegation + orchestration) / distributed
                  │
Quality       testing (fakes) · conformance (contract suites) · evaluator (pass^k)
                  │
Microkernel   core — only external dep is a JSON-Schema validator
               runtime + contracts + ~12 seams + in-memory reference impls
```

**Invariant:** core depends on no adapter; every adapter depends only on core. Any
implementation can be swapped, and the `conformance` contract suites prove the swap is
safe.

## The run loop

`createAgentEngine(options)` returns an `AgentEngine` with `run` / `step` / `resume` /
`runStream`. Each step:

1. **Check abort** signal.
2. **Reflect** if stalled (`onNonProgress: "reflect"` injects a corrective message).
3. **Check budget** — every `RunBudget` dimension; a hard `KERNEL_DEFAULT_MAX_ITERATIONS`
   (1000) backstops even when no budget is configured.
4. **Compact** context if a compactor is configured.
5. **Build context** — retrieve memory, inject seed memory, derive goals, resolve capabilities.
6. **Plan** — the planner returns one `Action`, a parallel tool batch, or a stream.
7. **Authorize** — the policy returns `allow | deny | stop | require_approval`.
8. **Route** by decision × action type: tool call (via the scheduler), delegation,
   thought/goal, `final_answer` (→ completed), `ask_user` / `schedule` (→ waiting),
   stop/deny (→ terminal).

This collapses logic that ad-hoc harnesses scatter into a single auditable state machine.

## Data model — three concepts

| Concept | What it is | Key point |
|---|---|---|
| **AgentState** | Persisted run state | Holds `variables` (host scratch, **not** reconstructable from the log); optimistic `version` |
| **ExecutionLogEntry** | Immutable audit event stream | Monotonic `sequence` per run; 25+ event types; append before resolve |
| **AgentContext** | Per-step view built for the planner | Retrieved memory, available tools, goals, remaining budget; not stored |

**State ≠ Log.** `replayAgentStateFromLog()` reconstructs actions/observations/status
read-only (and drops policy-denied/stopped actions to match live state), but cannot
rebuild `variables` — so resuming a run always loads from the `StateStore`, never replay.

## The injection seams

Each seam is an interface/`Protocol` with an in-kernel reference implementation plus
production adapters:

| Seam | Reference impl | Production adapter |
|---|---|---|
| Planner | FakePlanner / RuleBasedPlanner | model-openai, model-ondevice |
| PolicyManager | BasicPolicy / PermissionPolicy / PolicyPipeline | + risk classifier |
| ToolRegistry / ToolScheduler | in-memory + eval-free validator | inject ajv, etc. |
| MemoryManager | Noop / InMemory / **EmbeddingMemory** | memory-postgres |
| StateStore | InMemoryStateStore | state-file, state-postgres |
| Observer | Noop / InMemory / Console / Streaming / Composite | observer-otel |
| DelegationManager | (host-specific) | multi-agent |
| ApprovalProvider | InMemoryApprovalStore | host transport |
| Clock / IdGenerator | defaults | inject for determinism |
| Compactor / MemoryGovernor | SessionMemoryGovernor | host policy |
| CapabilityRegistry | InMemoryCapabilityRegistry | — |

## Bounded autonomy

Budgets are enforced **inside** the kernel before each planning round —
`maxIterations`, `maxToolCalls`, `maxFailures`, `maxConsecutiveNonProgress`,
`maxGoalDepth`, `maxWallClockMs` — and a stalled planner triggers reflection with a
per-round growing allowance. There is no unbounded loop.

## Extensions above the kernel

- **multi-agent** — `SubagentDelegationManager` + `TaskGraphCoordinator`, plus reusable
  orchestration patterns (`runParallel` / `mapReduce` / `vote` / `plannerWorkerCritic`)
  over the `AgentRunner` seam. See [Multi-agent orchestration](guides/multi-agent.md).
- **distributed** — durable job queue with exclusive leases, retry/backoff, idempotent
  side-effects, and dead-letter inspection.

All of this is validated end-to-end against real models — see [Reports](reports/index.md).
