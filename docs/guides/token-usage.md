# Token usage

The kernel **measures** token usage and lets you **bound** a run by it — the same two
responsibilities it already has for latency/counts and `RunBudget`. It does **not**
price tokens or route by cost: that stays a host concern, derived from the counts this
exposes. (Added in 0.6.0.)

## How it flows

1. A **planner reports** usage for each model call via an optional `onUsage` /
   `on_token_usage` callback (non-breaking — planners that can't measure it never call it).
2. The **runtime records** each report as a `token_usage` event and aggregates a
   cumulative `state.tokenUsage` (`state.token_usage` in Python).
3. **Replay** re-aggregates from those events, so a replayed state matches live state.

`TokenUsage` = `{ inputTokens, outputTokens, totalTokens, model?, metadata? }`
(snake_case in Python).

## Reading it

=== "TypeScript"

    ```ts
    const result = await engine.run({ agent, task, host });
    console.log(result.state.tokenUsage);
    // { inputTokens: 108, outputTokens: 11, totalTokens: 119, model: "…" }
    ```

=== "Python"

    ```python
    result = await engine.run(agent=agent, task=task, host=host)
    print(result.state.token_usage)  # TokenUsage(input_tokens=108, output_tokens=11, ...)
    ```

## Budgeting by tokens

`RunBudget` gained token caps, enforced in the same pre-round check as
`maxToolCalls`/`maxWallClockMs` — the run stops with `budget_exhausted` once a cap is
reached:

=== "TypeScript"

    ```ts
    await engine.run({ agent, task, host, budget: { maxTotalTokens: 50_000 } });
    // also: maxInputTokens, maxOutputTokens
    ```

=== "Python"

    ```python
    await engine.run(agent=agent, task=task, host=host,
                     budget=RunBudget(max_total_tokens=50_000))
    ```

## Where the numbers come from

- **model-openai** reads the provider `usage` (Responses API `input_tokens` /
  `output_tokens`), blocking and streaming — exact counts.
- **model-ondevice** uses exact counts when the injected generator returns
  `{ text, usage }` (e.g. Ollama's `prompt_eval_count` / `eval_count`); otherwise it
  **estimates** from `countTokens` and marks `metadata.estimated = true`.

## Exporting & evaluating

- **observer-otel** emits the OpenTelemetry GenAI metric `gen_ai.client.token.usage`
  (a histogram split by `gen_ai.token.type` = input/output, with `gen_ai.request.model`)
  for each `token_usage` event.
- **evaluator** adds token totals to `EvalTrialMetrics` and `averageTotalTokens` /
  `totalTokens` to the summary. Cost stays via the host `estimateCost` hook — now it can
  be fed real tokens.

## Non-goals

No pricing tables, per-token rates, or cost-based routing in the kernel. Cost is a host /
evaluator concern computed from these counts.
