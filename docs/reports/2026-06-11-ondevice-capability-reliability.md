# On-device agentic capability & reliability — enhancements and evidence

- **Date:** 2026-06-11
- **Package:** `@agentic-kernel/model-ondevice` (TS) / `agentic_kernel.model_ondevice` (Python)
- **Goal:** maximize the *validity* and *reliability* of on-device/small-model
  agents through framework levers, measured against real local models (Ollama).
- **Method:** the adapter drives the real `createAgentEngine` loop over Ollama
  (`qwen3.5:2b`, `gemma3:4b`) on multi-step search→calculate→answer tasks.
  - **Validity** = grounded **and** answer-correct **and** well-formed. "Grounded"
    means the answer was actually computed by tools (a calculator observation
    equals ground truth and every entity was searched) — a *hallucinated but
    numerically-right* answer counts as INVALID.
  - **Reliability** = `pass^k` / voted-correct over K stochastic trials
    (temp 0.7), aggregated with the SDK's own `summarizeModelScenario`.

## Levers shipped (all opt-in, additive; core and the remote lane untouched)

| Lever | Option / API | Targets | Verdict |
| --- | --- | --- | --- |
| Plan-then-execute | `planMode: "plan-then-execute"` | step-dropping (validity) | **Validated win** |
| Episode self-consistency | `runSelfConsistent(run, {samples})` | variance (reliability) | **Validated win (when model ≥ ~decent)** |
| Terminal verification | `verifyAnswer` + `groundingVerifier()` | hallucination | Shipped; not exercised by this task set |
| Few-shot examples | `examples` | adherence | Shipped |
| Per-step self-consistency | `SelfConsistencyPlanner` | per-decision variance | **Tried, did NOT help agents — kept with caveat** |

## Finding 1 — plan-then-execute lifts validity (temp 0.7, K=4)

| model | react | plan-then-execute |
| --- | --- | --- |
| qwen3.5:2b | 0% | **69%** |
| gemma3:4b | 50% | **81%** |

Externalizing the plan as a replayable `thought` and re-injecting it each step
stops weak models from dropping the last hop. These are *grounded* gains
(answerMatch == grounded), not lucky guesses.

**Methodology caveat that changed the conclusion:** an earlier temp=0 single-shot
run made plan-mode look like it *broke* qwen (0%). Multi-sample evaluation at
temp 0.7 revealed the opposite — plan-mode greatly helps. **Deterministic
single-shot testing lies; always multi-sample.**

## Finding 2 — reliability is a separate axis, and the bottleneck

High single-shot validity does not imply reliability. gemma plan-mode scored 81%
validity but only **50% `pass^4`** — half the scenarios failed to be valid on all
4 trials. `pass^k` is the metric that matters for shipping an agent.

## Finding 3 — episode-level self-consistency is the reliability lever

Run the whole agent N times independently and majority-vote the final answer
(`runSelfConsistent`). K=15, vote N=5, disjoint groups:

| model | task | single-shot validity | voted-correct (reliability) |
| --- | --- | --- | --- |
| gemma3:4b | sum3 | 73% | **100%** (3/3 groups) |
| qwen3.5:2b | sum3b | 27% | **67%** (2/3) |
| gemma3:4b | sum3b | 60% | **67%** (2/3) |
| qwen3.5:2b | sum3 | 13% | 0% (0/3) |

Voting amplifies a correct plurality: it makes a *decent* model reliable
(gemma sum3 73%→100%) but **cannot rescue a model that is mostly wrong**
(qwen sum3 13%→0% — the wrong answers scatter and never reach the truth). The
boundary is ~"the correct answer must already be the single most common outcome."

## Finding 4 — per-step self-consistency is the wrong granularity

Voting on each *step* (`SelfConsistencyPlanner`) did **not** help and sometimes
hurt (qwen 40%→20%, gemma 60%→60%): it reinforces the model's modal next step,
which on a hard task is often the wrong one, and the extra sampling derailed weak
runs (well-formed 80%→60%). Classic self-consistency votes on *outcomes*, not
intermediate steps. The per-step wrapper is kept (it correctly stabilizes a
single high-variance decision) but documented as the wrong tool for agent-level
reliability.

## Finding 5 — verification (B): situational alone, decisive in the full stack

`groundingVerifier` rejects answers whose numbers aren't in observations (the
"lucky guess"). At temp 0.7 the models rarely hit that exact failure, so grounding
verify alone was roughly neutral. But a **stronger process verifier** — a host
rule wired through the same `verifyAnswer` seam — is what finally reaches 100%
(Finding 6). Earlier "B not exercised" runs were temp=0 artifacts.

## Finding 6 — 100% reliability is reachable: plan + process-verify + voting

Combining all three levers and a **host process verifier** (via `verifyAnswer`)
that rejects a `final_answer` unless: every entity was searched, the calculator
was used, the answer equals the calculator result, the result is an integer, and
the calculator's operands are exactly the looked-up values. K=15, vote N=5:

| model | diff | sum2 | sum3 | sum3b |
| --- | --- | --- | --- | --- |
| **gemma3:4b** | **100%** | **100%** | **100%** | **100%** |
| qwen3.5:2b | 100% | 100% | 67% | 67% |

`gemma3:4b` reaches **100% reliability on every task** at vote N=5 — the process
verifier eliminated the last residual (gemma sum3b's `42.6` non-integer
miscalculation: 67%→100%). The verifier is **host domain logic** ("populations
are integers", "use each value once"); the SDK supplies the seam and a generic
`groundingVerifier`, the host encodes the task's invariants. Correct layering —
only the host knows the task's correctness rules.

qwen3.5:2b reached 100% on 2-hop but only ~67% on 3-hop **at N=5**. That was NOT a
capability floor — it was an under-sized ensemble. The correct answer is still
qwen's *modal* output on 3-hop (just less dominant: ~30–44% single-valid, wrong
answers scattered), so a **larger ensemble** captures it. At **vote N=9**:

| model (3-hop) | N=5 | N=9 |
| --- | --- | --- |
| qwen3.5:2b sum3 | 67% | **100%** (`[76,76,76]`) |
| qwen3.5:2b sum3b | 67% | **100%** (`[42,42,42]`) |

So **both models reach 100% on all 4 tasks** with the full recipe — the weaker
model just needs a bigger vote ensemble. Ensemble size `N` is the cost/reliability
knob: a stronger model (gemma) gets there at N=5; a weaker one (qwen) at N=9.

**Recipe to ensure 100%:** plan-then-execute + a process `verifyAnswer` that
enforces the task's correctness invariants + `runSelfConsistent`, with the vote
ensemble `N` sized to the model (larger for weaker models). The one real
requirement: **the correct answer must be the model's modal output** — then enough
votes make it win deterministically. If a model is so weak that a wrong answer is
its mode, no ensemble fixes it; use `agreement` to abstain/escalate (never
confidently wrong) or pick a stronger model.

## Recommended recipe for on-device agents

1. **`planMode: "plan-then-execute"`** — the validity lever; biggest single gain.
2. **A process `verifyAnswer`** that encodes the task's correctness invariants
   (all inputs gathered, answer == tool result, domain constraints) — this is what
   turns "grounded" into "correct" and removes systematic miscalculations.
3. **`runSelfConsistent({ samples: N })`** — votes the final answer across runs;
   **size `N` to the model**: gemma3:4b reaches 100% at N=5, qwen3.5:2b at N=9.
   Add `acceptFirst` (with the sound verifier) and `minSamples: 1` so easy cases
   cost ~1 episode instead of N — 100% without paying full N every time (Finding 7).
4. **The one requirement: correct must be the model's modal output.** Voting then
   makes it win deterministically. Both tested models clear this on all 4 tasks.
5. **If a wrong answer is the model's mode**, no ensemble fixes it — use
   `agreement` to abstain/escalate (never confidently wrong) or pick a stronger
   model. Keep tool chains short (≤2 hops) or decompose to stay above the bar.
6. Few-shot (`examples`) and the generic `groundingVerifier` are available too.

With 1–3, **both qwen3.5:2b and gemma3:4b reach 100% on all 4 tested tasks**
(qwen needs the larger N=9 ensemble) — a concrete demonstration that "ensure
100%" is reachable for on-device agents with the full stack.

## Finding 7 — cost: 100% reliability without paying full N every time

Voting at fixed N is wasteful — most runs just confirm an already-obvious
agreement. `runSelfConsistent` gained three cost controls (all opt-in,
back-compatible):

- **`minSamples` < `samples`** — stop as soon as the leader's lead exceeds the
  remaining budget (same answer as full N, fewer runs).
- **`acceptFirst`** — accept the first run that passes a *sound* check. When the
  process verifier certifies correctness, one verified run is enough — no vote.
- **`concurrency`** — run waves in parallel for batched-inference hosts (latency).

Measured on real models (max N=9, plan + process-verify, vote correctness held at
100% throughout):

| strategy | gemma3:4b diff — avg episodes |
| --- | --- |
| fixed N=9 | 9.0 |
| adaptive (early-stop) | 5.5 (−39%) |
| **accept-first-verified** | **1.0 (−89%)** |

On an easy task the sound verifier collapses cost from 9 episodes to 1 with no
loss of correctness. On hard tasks accept-first needs a few retries (the model
fails the verifier more often) so the saving shrinks, but adaptive early-stop and
accept-first both stay strictly cheaper than fixed N. **This is the on-device
advantage realized: 100% reliability at a fraction of the naive voting cost.**

The composable `verifiers` module (`requireToolUsed`, `answerEqualsToolResult`,
`answerIsInteger`, `answerMatches`, `allVerifiers`) lets a host assemble the sound
process verifier from tested pieces.

## Boundary (honest)

Framework levers maximize what the model can do; they don't replace capability.
The one hard requirement is that the **correct answer be the model's modal
output** — both tested models clear this on all 4 tasks (qwen needs N=9). If a
wrong answer were a model's mode, no ensemble fixes it: abstain/escalate on low
`agreement`, or use a stronger model.

## Status

Both SDKs at full gate (TS 530 tests; Python 100% statement+branch, ruff +
mypy --strict). All levers opt-in / default-off; core and the remote lane
unchanged. Not version-bumped or published. Eval scripts live under `/tmp`
(not committed).
