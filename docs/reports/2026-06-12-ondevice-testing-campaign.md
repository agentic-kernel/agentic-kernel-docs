# On-device agent testing campaign — master report

- **Date:** 2026-06-09 → 2026-06-12
- **Subject:** `@agentic-kernel/model-ondevice` (TS) / `agentic_kernel.model_ondevice` (Python)
- **Question:** can small/local models run reliable, cost-efficient agents on the
  kernel — and what does the framework need to make that true?
- **Method:** every result is from the **real kernel** (`createAgentEngine`) driving
  **real local models** via Ollama (`qwen3.5:0.8b/2b`, `gemma3:4b`, `qwen3:8b`,
  `deepseek-r1:8b`). No mocks for the empirical claims. Validity is judged on the
  full trajectory (grounded + correct + well-formed); reliability is `pass^k` /
  voted-correct over stochastic trials, aggregated with the SDK's own
  `summarizeModelScenario`.

## Headline

Starting from "small models are unreliable agents," the campaign ended at:
**100% reliability on both a 2B and a 4B model across all tested multi-step tasks,
at ~1–2 model runs per answer instead of 9.** Maximum accuracy and minimum cost,
together — the on-device thesis, demonstrated.

The path required six phases; each surfaced a problem, fixed it in the right
layer, and verified the fix on real models. Two methodology lessons recurred and
shaped everything: **temp=0 single-shot measurement lies** (always multi-sample),
and **substring "correctness" lies** (judge the grounded process, not the number).

---

## Phase 1 — Single-shot performance (2026-06-09)

`docs/reports/2026-06-09-model-ondevice-perf.md`. Five models, single planner calls.

- **The adapter is not the bottleneck — by 3–4 orders of magnitude.** Its own work
  (render + token-trim + parse) is microseconds; end-to-end latency is seconds and
  is ~100% model inference.
- **78/78 calls produced a valid action first-try (0 repairs)** — including a
  reasoning model whose answer was wrapped in `<think>…</think>`.
- **Biggest latency lever is configuration, not code:** disabling "thinking" is
  ~**17×** (deepseek-r1 33s vs qwen3:8b 1.9s). 2–4B is the on-device sweet spot.
- Shipped here: `toolPromptStyle: "compact"`, `buildActionSchema()` (constrained
  decoding), render short-circuit (O(kept) not O(history); ~9× faster at 600 blocks).

## Phase 2 — Long-run multi-step soak (2026-06-10)

`docs/reports/2026-06-10-model-ondevice-longrun.md`. 22 minutes, 194 episodes of the
real plan→tool→observe loop.

- **194/194 completed — no crashes, stalls, or loops.** Infrastructure is
  production-solid.
- **Discovered a real malform**: gemma3:4b put the tool name in `type`
  (`{"type":"calculator",…}`) ~15% of the time. Fixed by **tolerant normalization**
  (when `type` matches an available tool) → repairs **15.3% → 0.0%**, no regression.
- Added **`onRepair`** telemetry — and corrected a **contaminated metric** (I had
  inferred repairs as `plan_calls − steps`; measured at the source instead).
- Documented the model failure mode: **step-dropping on 3-hop chains** (search 2
  cities, skip the 3rd, hallucinate the value).

## Phase 3 — Validity + reliability evaluation (2026-06-11)

`docs/reports/2026-06-11-ondevice-capability-reliability.md`. Proper grounded judging
+ `pass^k` over stochastic trials (temp 0.7).

- **Plan-then-execute lifts grounded validity**: qwen3.5:2b **0→69%**,
  gemma3:4b **50→81%**. Externalizing the plan as a replayable `thought` stops
  step-dropping.
- **Per-step self-consistency FAILED** (qwen 40→20%): voting on each step
  reinforces the modal — often wrong — next step. Kept with a caveat.
- **Episode-level voting works**: `runSelfConsistent` (vote the final answer across
  independent runs) — gemma sum3 73%→100%.
- **Reliability ≠ validity**: gemma was 81% valid but only 50% `pass^4`. Reliability
  is the metric that matters for shipping.
- Methodology fix proven: a temp=0 run made plan-mode look like it *broke* qwen
  (0%); temp 0.7 multi-sample showed it greatly *helps*.

## Phase 4 — 42-minute stability soak (2026-06-12)

42.1 minutes, 183 episodes, plan-then-execute, both models.

- **183/183 completed, 0 repairs.**
- **No drift**: first-half vs second-half latency −3%, correctness +2pt → stable, no
  memory growth or slowdown over 40+ minutes of continuous operation.

## Phase 5 — Reaching 100% reliability (2026-06-12)

Full stack: **plan-then-execute + a process `verifyAnswer` + episode voting.**

- The **process verifier** (host rule via the shipped `verifyAnswer` seam) rejects
  an answer unless every entity was searched, the calculator was used, the answer
  equals the calculator result, the result is an integer, and the operands match the
  looked-up values. It eliminates **grounded-but-wrong** miscalculations (gemma
  sum3b `42.6`: 67%→100%).
- **Both models reach 100% on all 4 tasks** (gemma at vote N=5, qwen at N=9).
- The earlier "qwen 3-hop capability floor" was **wrong** — an under-sized ensemble.
  At N=9 qwen hits 100% on 3-hop. **Ensemble size N is the cost/reliability knob.**
- The one hard requirement: **the correct answer must be the model's modal output.**

## Phase 6 — Cost optimization (2026-06-12)

100% reliability shouldn't cost a full N runs every time. `runSelfConsistent` gained
(opt-in, back-compatible): `minSamples` (early-stop when the leader can't be caught),
`acceptFirst` (accept the first run that passes a sound check → N→1), `concurrency`
(parallel waves). Plus a composable `verifiers` module.

Measured (max N=9, vote correctness **100% throughout = 18/18 groups**):

| model | task | fixed N=9 | adaptive | accept-first |
| --- | --- | --- | --- | --- |
| gemma3:4b | diff | 9.0 | 5.0 | **1.0** |
| gemma3:4b | sum3 | 9.0 | 5.3 | **1.0** |
| gemma3:4b | sum3b | 9.0 | 5.3 | **1.7** |
| qwen3.5:2b | diff | 9.0 | 6.3 | **1.7** |
| qwen3.5:2b | sum3 | 9.0 | 7.7 | **2.0** |
| qwen3.5:2b | sum3b | 9.0 | 7.7 | **2.3** |

avg model runs per answer. Accept-first cuts cost **−74% to −89%** at unchanged
100%. The synergy: the process verifier runs *inside* each episode (retry-until-pass),
so a completed episode is already likely correct, and accept-first stops at the first
verified one — the recipe is effectively **"retry until provably correct, then stop"**,
not "run N then vote."

---

## The recipe (what to actually do)

1. `planMode: "plan-then-execute"` — externalize the plan; stops step-dropping.
2. A process `verifyAnswer` (compose from the `verifiers` module) that encodes the
   task's correctness invariants — turns "grounded" into "correct".
3. `runSelfConsistent` with `acceptFirst` (the sound verifier) + `minSamples: 1`,
   `samples: N` sized to the model — 100% at ~1–2 runs, not N.
4. Requirement: the correct answer must be the model's modal output. If not, use
   `agreement` to abstain/escalate, or pick a stronger model.

## What shipped (both SDKs, opt-in, core + remote lane untouched)

| Feature | Purpose |
| --- | --- |
| tolerant `type=<toolName>` normalization | remove a 15% repair tax |
| `onRepair` | repair telemetry |
| `toolPromptStyle: "compact"` | prefill savings for tool-heavy agents |
| `buildActionSchema()` | constrained/grammar decoding |
| render short-circuit + atomic action/obs blocks | O(kept) prompt cost |
| `planMode: "plan-then-execute"` | validity (anti step-drop) |
| `verifyAnswer` + `groundingVerifier()` + `verifiers` module | correctness gate |
| `examples` | few-shot adherence |
| `SelfConsistencyPlanner` (per-step) | kept; documented as the wrong tool for agents |
| `runSelfConsistent` (episode) + `minSamples`/`acceptFirst`/`concurrency` | reliability + cost |

## Honest boundaries

- The framework maximizes what the model can do; it does not replace capability. If
  a model's modal answer is wrong, no ensemble fixes it.
- 100% here is on a closed, verifiable task family (search→calculate). The recipe
  generalizes wherever a **cheap soundness check** exists (recompute, run tests,
  cross-check a source); open-ended tasks without one fall back to plain voting.
- The process verifier's domain rules are **host-supplied** (only the host knows
  "populations are integers") — correct layering; the SDK ships the seam + building
  blocks.

## Gate status

TS **530 tests** pass; Python **100% statement+branch**, ruff + mypy --strict clean.
All features opt-in / default-off; `core` and `model-openai` unchanged. Version held
at 0.4.0 (not published). Benchmark/eval scripts live under `/tmp` (not committed).
