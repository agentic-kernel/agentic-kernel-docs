# Governance & long-run resilience

0.9.0 hardens the kernel for the agentic era: policy that guides instead of
kills, tiered authorization, opaque provider state that survives resume, and
runs that recover across crashes and schema upgrades.

## Denials are recoverable, not fatal

By default a policy `deny` is recorded as a **failed observation**
(`policy_denied: <reason>`, no tool executed, `failureCount + 1`) and the loop
continues — so the planner can adapt, bounded by `maxFailures` and
repeated-action detection. This matches how real agents recover from a blocked
step instead of aborting a long task over one misstep. For zero-tolerance paths
(security bypass attempts), opt back into hard termination:

=== "TypeScript"

    ```ts
    new AgentEngine({ planner, policy, denyBehavior: "fail" });
    ```

=== "Python"

    ```python
    create_agent_engine(planner=..., policy=..., deny_behavior="fail")
    ```

## Tiered policy

`createTieredPolicy` / `create_tiered_policy` maps a tool's `sideEffectLevel` to
a decision (default: `none`/`read` → allow, `write`/`destructive` →
require_approval), with per-tool overrides; unknown tools fail closed. It also
implements `filterTools`, so approval-/deny-tier tools can be shaped out of the
planner's offered toolset **before** planning — moving authorization up-front
for native tool-calling instead of rejecting after a wasted turn.

=== "TypeScript"

    ```ts
    const policy = createTieredPolicy({
      describeTools: registry.describe.bind(registry),
      tiers: { destructive: "deny" },
      overrides: { export_data: "require_approval" }
    });
    ```

=== "Python"

    ```python
    policy = create_tiered_policy(
        describe_tools=registry.describe,
        tiers={"destructive": "deny"},
        overrides={"export_data": "require_approval"},
    )
    ```

## Opaque provider reasoning state

Actions and messages carry an optional `providerMetadata` / `provider_metadata`
the kernel never inspects but persists in the event log and reconstructs on
replay — so a resumed run hands the provider back exactly what it emitted (e.g.
a model's encrypted extended-thinking signature for multi-turn continuity).

## Crash-window reconciliation

On resume, a `tool_call` whose observation never landed (a crash between
executing the tool and persisting its result) is reconciled into a failed
`tool_result_unknown_after_resume` observation, so the planner sees an explicit
unknown result rather than a silently dangling action (delivery is
at-least-once — verify rather than assume). Disable with
`reconcileDanglingToolCalls: false` when your tools carry exactly-once receipts.

## Reflection wording

The kernel owns stall *detection*; the nudge wording is yours:

=== "TypeScript"

    ```ts
    new AgentEngine({ planner, policy, reflectionMessage: (s) => "换个思路重试" });
    ```

=== "Python"

    ```python
    create_agent_engine(planner=..., policy=..., reflection_message=lambda s: "换个思路重试")
    ```

## Reliability measurement: pass^k

The evaluator reports the **pass^k curve** (`passHatK` / `pass_hat_k`,
unbiased `C(successes,k)/C(trials,k)`), not just an all-or-nothing scalar —
because `pass@1` overstates production reliability. The Markdown report shows
`pass^2` and worst-case `pass^k` so reliability decay with k is visible.
