# `model-ondevice` performance report

- **Date:** 2026-06-09
- **Package:** `@agentic-kernel/model-ondevice` (TS) / `agentic_kernel.model_ondevice` (Python)
- **Hardware:** Apple Silicon laptop, models served by local **Ollama 0.24**
- **Method:** the adapter drives real local models through an injected
  `TextGenerator` backed by Ollama `/api/chat` (`stream:false`, `temperature:0`,
  `think:false` unless noted). Per call we record wall latency, Ollama's token
  counts/durations, generator round-trips (= 1 + repair retries), and the parsed
  action kind. Benchmark scripts are not committed (kept under `/tmp`).

## TL;DR

1. **The adapter is not the bottleneck — by 3–4 orders of magnitude.** Its own
   CPU work (prompt render + token trim + tolerant parse) is **microseconds**;
   end-to-end latency is **seconds** and is ~100% model inference.
2. **The repair loop is free on capable models.** Across 5 models × 78 calls,
   **78/78 produced a valid action on the first try (0 repairs)** — including a
   reasoning model whose answer was wrapped in `<think>…</think>`. The restricted
   5-kind vocabulary + schema-shaped prompt is what makes small models reliable.
3. **Biggest end-to-end win is configuration, not code:** disabling "thinking"
   (or not using a reasoning model) on an interactive lane is ~**17×**.

## End-to-end latency (raw decoding, 6 iters × 3 scenarios)

Scenarios: `tool-use` (expects `tool_call`), `final-answer`, `after-observation`.

| model | ok/n | p50 | p90 | max | mean repairs | prompt tok | out tok | decode tok/s | adapter+net |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| qwen3.5:0.8b | 18/18 | 780ms | 921ms | 925ms | 0.00 | 207 | 23 | 48.3 | 1.6ms |
| qwen3.5:2b | 18/18 | 1460ms | 1581ms | 1613ms | 0.00 | 207 | 24 | 28.3 | 2.8ms |
| gemma3:4b | 18/18 | 1360ms | 1639ms | 1766ms | 0.00 | 208 | 27 | 30.6 | 6.4ms |
| qwen3:8b | 18/18 | 1911ms | 2262ms | 3228ms | 0.00 | 202 | 25 | 16.8 | 11.2ms |
| deepseek-r1:8b (think) | 12/12 | 32940ms | 44124ms | 44833ms | 0.00 | 183 | 466 | 17.5 | 6.1ms |

`adapter+net` = wall − Ollama `total_duration` ≈ localhost HTTP + JSON parse
(not adapter compute; see microbench). It tracks response size/scheduling, not
adapter work.

**Reading it:** latency scales with model size (0.8B→8B: ~0.78s→1.9s p50) and
decode throughput falls (48→17 tok/s) — both intrinsic to the model. The 0.8B
model is fast but under-powered (it chose `ask_user` where it should have called
`search`); **2–4B is the sweet spot** here (<1.7s to a correct action). The
reasoning model is accurate but its `<think>` block (avg 466 output tokens) makes
it 7–44s — unsuitable for an interactive lane unless thinking is disabled.

## Constrained decoding (Ollama `format` = `buildActionSchema()`)

Passing the action schema (now exported as `buildActionSchema`) to the runtime's
structured-output mode. 4 iters × 3 scenarios:

| model | ok/n | p50 | p90 | mean repairs |
| --- | --- | --- | --- | --- |
| qwen3.5:2b | 12/12 | 1480ms | 1555ms | 0.00 |
| gemma3:4b | 12/12 | 1231ms | 1528ms | 0.00 |

On these models (already 0-repair raw) constrained decoding is latency-neutral.
Its value is **insurance**: it makes invalid output structurally impossible, so
it matters most on weaker / more aggressively quantized / older models, and it
can trim leading filler tokens. The adapter already passes the schema as
`GenerateRequest.schema`; hosts opt in inside their `TextGenerator`.

## Adapter-only overhead (no model) — render + trim + parse, µs/call

Pure CPU cost as the transcript grows, before vs after the render short-circuit
(materialize newest-first, stop at the first block over budget → O(kept), not
O(history)):

| transcript blocks | before p50 | after p50 | before p99 | after p99 |
| --- | --- | --- | --- | --- |
| 0 | 8.0µs | 4.5µs | 39µs | 39µs |
| 40 | 22.0µs | 8.1µs | 105µs | 65µs |
| 100 | 38.9µs | 14.1µs | 185µs | 74µs |
| 600 | 178.4µs | 19.6µs | 440µs | 79µs |

At 600 blocks the optimization is ~**9× faster** and overhead is now roughly
flat. Either way the adapter costs **sub-millisecond** vs seconds of inference —
this change is correctness/scaling hygiene, not a latency lever.

## Tuning guide (by impact)

1. **Interactive lane: disable thinking / avoid reasoning models** — ~17×.
2. **Pick a 2–4B model** for the latency/quality sweet spot on-device.
3. **Reuse the runtime's KV prefix cache:** the adapter puts the stable
   `system` prompt (role/goal/instructions/tools/action shapes) first and the
   volatile transcript last, so a fixed tool set keeps a byte-stable prefix
   across steps. Note: once token-trim starts dropping the oldest blocks, the
   user-message prefix changes and the cache for that segment is invalidated.
4. **Tool-heavy agents: `toolPromptStyle: "compact"`** renders
   `name(p1, p2): description` instead of inlining each tool's full JSON schema,
   cutting prefill tokens (matters as tool count grows; negligible for 1–2 tools).
5. **Weak/quantized models: enable constrained decoding** via `buildActionSchema`
   wired to your runtime's grammar/format param — turns rare repairs into
   structurally-impossible ones.
6. `maxRepairAttempts` is a safety net, not a routine cost — it never fired on any
   tested model. Leave it at the default (2).
