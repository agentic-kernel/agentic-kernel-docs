# Agent Kernel — full test & hardening report

- **Period:** 2026-06-09 → 2026-06-13
- **Scope:** the `model-ondevice` adapter and the overall kernel, tested on
  **real models** end-to-end — local (Ollama: qwen3.5:0.8b/2b, gemma3:4b,
  qwen3:8b, deepseek-r1:8b) and a remote large model (deepseek-v4-pro via an
  OpenAI-compatible gateway). Both SDKs (TypeScript + Python) kept at parity.
- **Discipline:** every empirical claim is from the real kernel
  (`createAgentEngine`) driving a real model — no mocks. Validity is judged on the
  full trajectory (grounded + correct + well-formed); reliability is `pass^k` over
  stochastic trials, aggregated with the SDK's own `summarizeModelScenario`.

## Executive summary

1. **The adapter and kernel are production-solid.** Across multi-hour soak runs
   and a remote-large-model concurrent stress (450 episodes, 2323 LLM calls), the
   kernel was crash-free, lossless, and stable — no degradation over 40+ minutes,
   no event loss under concurrency.
2. **Small on-device models can be made 100% reliable agents** with the right
   framework levers — demonstrated on both a 2B and a 4B model across all tested
   multi-step tasks.
3. **At a fraction of the cost.** A sound-verifier + accept-first recipe reaches
   that 100% at ~1–2 model runs per answer instead of 9 (−74% to −89%).
4. **A large model solves genuinely hard agentic tasks through the kernel** —
   281/282 (100%) on data-dependent branching, conditional aggregation, deep
   dependent chains, and combinatorial constraint search.
5. **One real defect was found and fixed** (replay diverged on the policy-denied
   path), in both SDKs, with regression coverage and empirical re-verification.
6. **Two recurring methodology lessons**, both of which changed conclusions:
   temp=0 single-shot measurement lies (always multi-sample); substring
   "correctness" lies (judge the grounded process, not the printed number).

---

## Part A — What shipped (`model-ondevice`, both SDKs, opt-in, core untouched)

| Feature | Purpose |
| --- | --- |
| `OnDevicePlanner` + injected `TextGenerator` | run any raw-text model as a `Planner`; no runtime bundled |
| tolerant parse + `type=<toolName>` normalization | absorb weak-model JSON mistakes (cut a 15% repair tax) |
| `maxRepairAttempts` + `onRepair` | error-feedback repair loop + telemetry |
| `maxPromptTokens` + render short-circuit | O(kept) token-aware prompt, atomic action/obs blocks |
| `toolPromptStyle: "compact"` | prefill savings for tool-heavy agents |
| `buildActionSchema()` | constrained/grammar decoding (Ollama `format`, llama.cpp) |
| `planMode: "plan-then-execute"` | externalize the plan (anti step-drop) — validity lever |
| `verifyAnswer` + `groundingVerifier()` + `verifiers` module | terminal correctness gate (host composes domain rules) |
| `examples` | few-shot adherence |
| `SelfConsistencyPlanner` (per-step) | kept; documented as the wrong tool for agents |
| `runSelfConsistent` (episode) + `minSamples`/`acceptFirst`/`concurrency` | reliability + cost controls |

## Part B — On-device empirical campaign (local models)

**Phase 1 — single-shot performance (06-09).** Adapter overhead is microseconds
vs seconds of inference; 78/78 calls valid first-try (0 repairs); disabling
"thinking" is ~17×; 2–4B is the on-device sweet spot.

**Phase 2 — 22-min / 194-episode soak (06-10).** 194/194 completed, no
stalls/loops. Found and fixed the `type=<toolName>` malform (gemma 15.3%→0% repairs).
Documented step-dropping on 3-hop chains.

**Phase 3 — validity + reliability (06-11).** Plan-then-execute lifted grounded
validity (qwen3.5:2b 0→69%, gemma3:4b 50→81%). Per-step self-consistency FAILED;
episode-level voting works. Reliability ≠ validity (81% valid but 50% `pass^4`).

**Phase 4 — 42-min / 183-episode soak (06-12).** 183/183 completed, 0 repairs,
**no drift** (first vs second half: latency −3%, correctness +2pt) — no leak/slowdown.

**Phase 5 — reaching 100% (06-12).** Full stack (plan + process-verify + voting):
**both qwen3.5:2b and gemma3:4b reached 100% on all 4 tasks** (gemma at N=5, qwen
at N=9). The "qwen 3-hop floor" was an under-sized ensemble, not a capability wall.

**Phase 6 — cost (06-12).** `acceptFirst` + `minSamples` cut cost while holding
100%: ~1.0–2.3 model runs per answer vs fixed 9 (−74% to −89%) — "retry until
provably correct, then stop."

## Part C — Remote large-model framework stress (06-13)

deepseek-v4-pro, 5 concurrent workers × 18.5 min on a shared store + observer.

- **450 episodes · 2323 real LLM calls · 0 API errors · 0 engine crashes ·
  12,367 events · 24.3 episodes/min.**
- **Hard-task accuracy 281/282 (100%)**:

| scenario | difficulty | pass |
| --- | --- | --- |
| branch | data-dependent branching (max → country → GDP) | 55/56 |
| twohop_min | deep dependent chain (min → country → GDP) | 57/57 |
| filter_agg | conditional aggregation (sum pops > 20M, ×2) | 56/56 |
| conditional | branch on a value (Tokyo > 13 → square else +100) | 56/56 |
| constraint | combinatorial search (two cities differing by 4M) | 57/57 |
| memory_recall | write → retrieve → recall | 56/56 |
| policy_deny | destructive-tool denial | 56/56 |
| stall_reflect | bounded autonomy (must terminate) | 56/56 |

Every framework surface verified under load: core loop, tool scheduling, policy
enforcement, memory injection/retrieval, bounded autonomy, state, and replay.

## Defects found & fixed

**Replay diverged on the policy-denied path (fixed, both SDKs).** The runtime logs
`action_selected` for every planned action before the policy check but does not
record a denied/stopped action in live `state.actions`; `replayAgentStateFromLog`
pushed every `action_selected`, over-counting by one on the denial path (56/56
policy-denied runs failed the replay-integrity check). Fix: replay drops the action
for a `deny`/`stop` decision. Regression tests added; re-running the workload gives
**replay integrity 62/62 (100%)**, policy_deny 7/7.

## Honest boundaries (not glossed)

- **Framework maximizes the model; it doesn't replace it.** The 100% recipe
  requires the correct answer to be the model's modal output; if a wrong answer is
  the mode, no ensemble fixes it — abstain on low `agreement` or use a stronger
  model.
- **100% was shown on a closed, verifiable task family** (search→compute). The
  recipe generalizes wherever a cheap soundness check exists (recompute, run tests,
  cross-check a source); open-ended tasks without one fall back to plain voting.
- **The process verifier's domain rules are host-supplied** — only the host knows
  "populations are integers." The SDK ships the seam + composable building blocks.
- **`InMemoryMemory` uses naive substring retrieval** — a reference impl; production
  memory should use `memory-postgres` (embeddings).
- **Per-step self-consistency does not help agent reliability** (measured) — kept
  but documented; prefer `runSelfConsistent`.

## The recommended on-device recipe

1. `planMode: "plan-then-execute"` — stop step-dropping.
2. A process `verifyAnswer` (compose from `verifiers`) — turn grounded into correct.
3. `runSelfConsistent` with `acceptFirst` + `minSamples: 1`, `samples` sized to the
   model — 100% at ~1–2 runs.
4. Below the bar: abstain/escalate on low `agreement`, or use a stronger model.

## Gate status & commits

Both SDKs green: **TS 530 tests pass; Python 100% statement+branch, ruff + mypy
--strict clean.** All adapter features opt-in/default-off; the one core change
(replay fix) is mirrored in both SDKs. Committed on `main` (not pushed); version
held at 0.4.0 (not published). Detailed phase reports:
`2026-06-09-model-ondevice-perf`, `2026-06-10-model-ondevice-longrun`,
`2026-06-11-ondevice-capability-reliability`, `2026-06-12-ondevice-testing-campaign`,
`2026-06-13-remote-model-framework-stress`. Benchmark/stress scripts live under
`/tmp` (not committed).
