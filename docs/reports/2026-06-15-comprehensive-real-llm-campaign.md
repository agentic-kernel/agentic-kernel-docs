# Comprehensive Real-LLM Test Campaign — Agent Kernel (TS + Python)

**Date:** 2026-06-15
**Scope:** Build + execute + report a comprehensive real-LLM test campaign covering
every kernel surface, in both SDKs, against real endpoints, with stress testing.
**Endpoints (verified live):** OpenAI-compatible gateway `http://127.0.0.1:17680/v1`
(deepseek-v4-flash/pro, claude-opus-4-8, gpt-5.5, gemini-2.5-pro); on-device Ollama
(`qwen3.5:2b`, `qwen3-embedding:4b`).
**Methodology:** multi-sample pass^k (never single-shot); grounded correctness (judge
the process, not substring); per-episode replay-integrity; crash/throw/event-loss
counters; cost from real token usage. Built on `@agentic-kernel/evaluator`.

---

## 1. Outcome

**No kernel defect surfaced.** Every functional scenario reached pass^k = 1.0 on the
remote models; all 9 flow harnesses pass in **both** SDKs with identical results; the
stress lane ran clean (0 errors / 0 throws / 0 replay-mismatch / 0 event-loss). The
one prior defect from the 06-13 campaign (replay over-counting policy-denied actions)
is now permanently regression-guarded by flow **S12** in both SDKs.

The on-device lane is mechanically sound but quality-bound by the 2B model (documented
below), consistent with the 06-11/06-12 findings.

---

## 2. Coverage — 17 areas

| # | Area | How tested | Lane |
|---|---|---|---|
| S1 | Core loop / grounded multi-hop | EvalSuite, grounded judge | real-LLM |
| S2 | Tool mechanics (timeout/output-schema/scope/input-schema) | flow, scripted | mechanism |
| S3 | Capabilities | covered by conformance (`capability` contract) | — |
| S4 | Policy refusal of destructive request | EvalSuite, grounded judge | real-LLM |
| S5 | Approval lifecycle & resume (approve→complete, reject→stop) | flow | mechanism |
| S6 | Schedule + ask_user wait/resume | flow | mechanism |
| S7 | Bounded autonomy (budget stops runaway planner) | flow | mechanism |
| S8 | Answer grounded in relevance-ranked memory | EvalSuite | real-LLM |
| S9 | EmbeddingMemory semantic recall | flow, **real embeddings** | real-LLM |
| S10 | Long-term memory lifecycle (supersede/tombstone) | covered by conformance + unit | — |
| S11 | Context compaction on long tasks | long-run (1.5k blocks) + unit | mechanism |
| S12 | State vs replay integrity incl. denied path | flow (regression guard) | mechanism |
| S13 | Delegation | covered by `multi-agent` unit + S14 orchestration | — |
| S14 | Orchestration patterns (vote / mapReduce / planner-worker-critic) | flow | **real-LLM** |
| S15 | Distributed scheduler (lease/complete/dead-letter) | flow | mechanism |
| S16 | Streaming (`runStream` ordered events + reasoning chunks) | flow | mechanism |
| S17 | Prompt-injection resistance | EvalSuite, grounded judge | real-LLM |

Mechanism flows use scripted planners deliberately: kernel behavior is
model-independent, so a stochastic model would add only flakiness, not signal. The
model-driven flows (S1/S4/S8/S9/S14/S17) use real LLMs where the model's behavior is
the thing under test.

---

## 3. Functional suite — pass^k (trials = 3, temp 0.3)

| Scenario | deepseek-v4-flash | deepseek-v4-pro |
|---|---|---|
| S1 grounded multi-hop | pass^k 1.0 | pass^k 1.0 |
| S4 policy refusal | pass^k 1.0 | pass^k 1.0 |
| S8 memory-grounded | pass^k 1.0 | pass^k 1.0 |
| S17 prompt injection | pass^k 1.0 | pass^k 1.0 |

`pass^k = 1.0` means **every** trial passed — the strict reliability metric.
S1 is graded grounded: `search_city` + `calculator` both invoked successfully **and**
the answer number equals the calculator's observed result (2,620,000).

---

## 4. Flow harnesses — 9/9 in BOTH SDKs (identical results)

| Flow | TS | Python | Detail |
|---|---|---|---|
| S2 tool mechanics | ✅ | ✅ | 4 tool calls → 4 failed observations, codes `handler_error, output_schema_failed, missing_scopes, schema_validation_failed` |
| S5 approval resume | ✅ | ✅ | approve→completed, reject→stopped |
| S6 schedule/ask resume | ✅ | ✅ | waiting_for_schedule→completed; waiting_for_user→completed |
| S7 bounded autonomy | ✅ | ✅ | runaway thought-planner → stopped + `budget_exhausted` |
| S9 EmbeddingMemory | ✅ | ✅ | paraphrase with **no shared words** recalled e3 (mitochondrion/powerhouse) |
| S12 replay integrity | ✅ | ✅ | live == replay on happy + policy-denied paths |
| S14 orchestration | ✅ | ✅ | vote=germany@1.00; mapReduce sum=2,620,000 (exact) |
| S15 distributed | ✅ | ✅ | lease/complete + non-retryable → dead-letter |
| S16 streaming | ✅ | ✅ | ordered `task_started … reasoning_chunk … task_completed` |

S9 and S14 exercise code added earlier this session (`EmbeddingMemory`, the multi-agent
orchestration patterns) under real models for the first time — both correct.

---

## 5. Stress lane (deepseek-v4-flash)

40 concurrent episodes (concurrency 10) on a **shared** state store + observer:

| Metric | Value |
|---|---|
| completed | 40 / 40 |
| accurate (grounded) | 40 / 40 |
| API errors | 0 |
| engine throws | 0 |
| replay mismatches | 0 |
| event loss | 0 |
| throughput | ~91 episodes/min |
| cost | $0.017 (143 calls, 80k+13k tok) |

**Very-long run:** 1,500 tool/observation blocks → 1,500 steps, replay == live (lifts
the 1,000-iteration backstop to prove transcript growth + replay beyond the default
ceiling). The default 1,000-iteration backstop itself was also observed firing.

**Cross-model (hard population-sum, trials 3):**

| Model | pass^k | accuracy | p95 latency | cost |
|---|---|---|---|---|
| DeepSeek V4 Flash | 1.0 | 1.0 | 5.9 s | $0.0013 |
| DeepSeek V4 Pro | 1.0 | 1.0 | 11.1 s | $0.0060 |

---

## 6. On-device lane

- **Embeddings (S9):** `qwen3-embedding:4b` via Ollama — semantic recall correct (the
  paraphrased query with no lexical overlap retrieved the right fact). On-device
  `EmbeddingMemory` works end-to-end.
- **Planning:** `qwen3.5:2b` in plan-then-execute runs mechanically (valid actions,
  ~8 s/step) but quality is model-bound — on the refusal task it emitted `ask_user`
  rather than a clean refusal. This is the previously-measured small-model capability
  ceiling (06-11/06-12), not a kernel issue. The recommended on-device recipe
  (plan-then-execute + `groundingVerifier` + `runSelfConsistent`) remains the path to
  100% reliability for the 2–4B class.

---

## 7. Dual-SDK parity

TS (`packages/sdk-validation/src/campaign/`) and Python
(`agentic-kernel-python/campaign/`) run the same suites/flows against the same
endpoints and produce identical pass/fail and key numbers (S9 top=e3; S14
vote=germany@1.00, sum=2,620,000; 7/7 mechanism flows with identical failure codes).
Per [[dual-sdk-parity]].

---

## 8. Honest not-covered / deferred

- **Official benchmarks (BFCL / tau2 / GAIA):** infra is present under `eval/official/`
  and prior coverage stands from the 2026-05-30 run (BFCL `simple_python` 5/5; tau-bench
  & tau2 single-task reward 1.0). A **full** category/domain sweep was not re-run this
  pass. **GAIA L1–L3 remains gated** — `HF_TOKEN` is not set in this environment, so the
  dataset cannot be loaded; this lane is explicitly **not covered** (no silent skip).
- **Postgres / OTel live adapters:** validated by the `conformance` contract suites and
  the existing `*.live.test.ts`; not re-exercised under load this pass (no DB/collector
  provisioned here).
- **S3 capabilities / S10 LT-memory lifecycle / S13 delegation** are covered by the
  conformance + unit suites rather than a bespoke real-LLM flow; S14 exercises the
  delegation-of-work path under a real model.

---

## 9. Reproduce

```bash
# TS — functional suite + flows (lanes: ondevice|remote|both; scale: smoke|moderate|heavy)
cd packages/sdk-validation && npm run build
CAMPAIGN_LANES=remote CAMPAIGN_SCALE=moderate CAMPAIGN_MODELS=deepseek-v4-flash,deepseek-v4-pro \
  node dist/campaign/run-campaign.js
# TS — stress + cross-model + long-run
STRESS_EPISODES=40 STRESS_CONCURRENCY=10 CROSS_MODELS=deepseek-v4-flash,deepseek-v4-pro \
  node dist/campaign/run-stress.js

# Python — parity flows
cd agentic-kernel-python && .venv/bin/python -m campaign.flows
```

Set `GATEWAY_BASE_URL` / `GATEWAY_API_KEY` / `OLLAMA_BASE_URL` to override endpoints.

---

## 10. Verdict

The kernel passed a comprehensive real-LLM campaign across every surface and both
SDKs: 100% functional pass^k on remote models, all lifecycle/integration flows green,
a clean concurrent stress lane with intact replay, and validated semantic memory +
multi-agent orchestration under real models. No new defect; the one historical defect
is now regression-guarded. The honest gaps are the gated GAIA dataset and a full
external-benchmark re-sweep, both clearly flagged above.
