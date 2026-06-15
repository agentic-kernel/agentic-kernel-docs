# Multi-agent orchestration

`multi-agent` coordinates parent runs and child subtasks through an injected
`AgentRunner` (the `AgentEngine`). Child runs use the same `Action`, `Observation`,
policy, memory, and execution-log contracts as any single-agent run.

- **`SubagentDelegationManager`** implements the core `DelegationManager` contract:
  `spawn_subagent` runs a child immediately or as an async subtask, `join_subagent`
  runs/observes it, `cancel_subagent` stops it. Children inherit parent tools and
  capabilities, get bounded budgets, and return their result as a parent observation.
- **`TaskGraphCoordinator`** adds append-only handoffs, child resume/cancel hooks,
  observer trace bridging, budget summaries, and shared-memory write proposals +
  authorization.

## Orchestration patterns

Reusable patterns composed purely over the `AgentRunner` seam (no kernel change). Each
takes a runner and `OrchestrationStep`s (`{ agent, task, host, budget? }`):

| Pattern | What it does |
|---|---|
| `runParallel(runner, steps, { concurrency })` | Bounded-concurrency fan-out, results in input order; a throwing branch becomes a `failed` result instead of sinking the batch. |
| `mapReduce(runner, steps, reduce, { concurrency })` | Fan out, then combine the results. |
| `vote(runner, steps, { answerKey, concurrency })` | Debate/plurality over N runs (same task with different personas, or N samples) → `VoteResult { result, key, agreement, results }`. |
| `plannerWorkerCritic(runner, { planner, decompose, synthesize })` | One planner run decomposes the task, workers run sub-steps in parallel, a critic synthesizes. The host supplies `decompose`/`synthesize`. |

=== "TypeScript"

    ```ts
    import { vote } from "@agentic-kernel/multi-agent";

    const result = await vote(engine, [stepA, stepB, stepC]);
    console.log(result.key, result.agreement);
    ```

=== "Python"

    ```python
    from agentic_kernel.multi_agent import vote

    result = await vote(engine, [step_a, step_b, step_c])
    print(result.key, result.agreement)
    ```

These patterns are validated under real models in the
[comprehensive real-LLM campaign](../reports/2026-06-15-comprehensive-real-llm-campaign.md).
