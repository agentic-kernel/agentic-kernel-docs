# Official Agent Benchmark Run

Date: 2026-05-30

This report records direct runs against official benchmark repositories and datasets for AgentEngine model capability validation. These are not synthetic AgentEngine scenarios.

## Environment

- Repository: `/Users/chengqing/Projects/github/agent-kernel`
- Official benchmark workspace: `/Users/chengqing/Projects/github/agent-kernel/eval/official`
- Model endpoint: OpenAI-compatible endpoint from `/tmp/agent-engine-openai.env`
- Available model family used in raw runs: DeepSeek-compatible OpenAI endpoint

## BFCL

Official source:

- Repository: `https://github.com/ShishirPatil/gorilla`
- Local path: `/Users/chengqing/Projects/github/agent-kernel/eval/official/gorilla/berkeley-function-call-leaderboard`

Run scope:

- Official BFCL runner
- Official raw BFCL data under `bfcl_eval/data`
- Category: `simple_python`
- Subset: `simple_python_0` through `simple_python_4`
- Model: `DeepSeek-V3.2-Exp-FC`

Commands:

```bash
cd /Users/chengqing/Projects/github/agent-kernel/eval/official/gorilla/berkeley-function-call-leaderboard
set -a; source /tmp/agent-engine-openai.env; set +a
export DEEPSEEK_API_KEY="$OPENAI_API_KEY"
export BFCL_PROJECT_ROOT="$PWD"
. .venv/bin/activate
bfcl generate --model DeepSeek-V3.2-Exp-FC --run-ids --num-threads 1 --result-dir result-official
bfcl evaluate --model DeepSeek-V3.2-Exp-FC --test-category simple_python --partial-eval --result-dir result-official --score-dir score-official
```

Result:

- Accuracy: `1.0`
- Correct count: `5`
- Total count: `5`
- Score file: `/Users/chengqing/Projects/github/agent-kernel/eval/official/gorilla/berkeley-function-call-leaderboard/score-official/DeepSeek-V3.2-Exp-FC/non_live/BFCL_v4_simple_python_score.json`
- Raw result file: `/Users/chengqing/Projects/github/agent-kernel/eval/official/gorilla/berkeley-function-call-leaderboard/result-official/DeepSeek-V3.2-Exp-FC/non_live/BFCL_v4_simple_python_result.json`

Interpretation:

This is a valid official BFCL partial run over raw BFCL cases. It is not a full BFCL leaderboard run. The `data_overall.csv` overall value is not meaningful for this subset because unrun categories remain empty.

## GAIA

Official source:

- Dataset: `https://huggingface.co/datasets/gaia-benchmark/GAIA`
- Local attempt environment: `/Users/chengqing/Projects/github/agent-kernel/eval/official/gaia-venv`

Attempted loads:

```python
load_dataset("gaia-benchmark/GAIA", "2023_all", split="validation[:1]")
load_dataset("gaia-benchmark/GAIA", "2023_level1", split="validation[:1]")
load_dataset("gaia-benchmark/GAIA", split="validation[:1]")
```

Result:

- Status: blocked
- Reason: Hugging Face reports the dataset is gated and requires authentication/access approval.
- Error: `Dataset 'gaia-benchmark/GAIA' is a gated dataset on the Hub. You must be authenticated to access it.`

Next requirement:

Provide a Hugging Face token with accepted GAIA dataset access:

```bash
export HF_TOKEN=...
```

Then rerun the GAIA loader and evaluator.

## tau-bench

Official source:

- Deprecated original repository: `https://github.com/sierra-research/tau-bench`
- Current successor repository: `https://github.com/sierra-research/tau2-bench`
- Local original path: `/Users/chengqing/Projects/github/agent-kernel/eval/official/tau-bench`
- Local successor path: `/Users/chengqing/Projects/github/agent-kernel/eval/official/tau2-bench`

Important note:

The original `tau-bench` repository states that its tasks are outdated and recommends the `tau2-bench` repository, now updated as tau3-bench, for fixed tasks and new domains.

### Original tau-bench Deprecated Raw Run

Run scope:

- Official old `tau-bench` runner
- Environment: `retail`
- Task ID: `0`
- Agent strategy: `tool-calling`
- User strategy: `llm`
- Model: `deepseek-v4-flash`

Command:

```bash
cd /Users/chengqing/Projects/github/agent-kernel/eval/official/tau-bench
set -a; source /tmp/agent-engine-openai.env; set +a
. .venv/bin/activate
python run.py \
  --agent-strategy tool-calling \
  --env retail \
  --model deepseek-v4-flash \
  --model-provider openai \
  --user-model deepseek-v4-flash \
  --user-model-provider openai \
  --user-strategy llm \
  --max-concurrency 1 \
  --task-ids 0
```

Result:

- Average reward: `1.0`
- Pass^1: `1.0`
- Result file: `/Users/chengqing/Projects/github/agent-kernel/eval/official/tau-bench/results/tool-calling-deepseek-v4-flash-0.0_range_0--1_user-deepseek-v4-flash-llm_0530122121.json`

### tau2/tau3 Successor Raw Run

Run scope:

- Official `tau2-bench` runner
- Domain: `mock`
- Tasks: 1
- Trials: 1
- Agent: `llm_agent`
- User: `user_simulator`
- Model: `deepseek/deepseek-chat`

Command:

```bash
cd /Users/chengqing/Projects/github/agent-kernel/eval/official/tau2-bench
set -a; source /tmp/agent-engine-openai.env; set +a
export DEEPSEEK_API_KEY="$OPENAI_API_KEY"
uv run tau2 check-data
uv run tau2 run \
  --domain mock \
  --agent-llm deepseek/deepseek-chat \
  --user-llm deepseek/deepseek-chat \
  --num-trials 1 \
  --num-tasks 1 \
  --max-concurrency 1 \
  --timeout 180 \
  --save-to agent-engine-deepseek-mock
```

Result:

- Reward: `1.0`
- Pass^1: `1.0`
- DB match: `1 / 1`
- Write actions: `1 / 1`
- Agent errors: `0`
- User errors: `0`
- Result file: `/Users/chengqing/Projects/github/agent-kernel/eval/official/tau2-bench/data/simulations/agent-engine-deepseek-mock/results.json`

## Coverage Status

| Benchmark | Official Raw Data | Runner Executed | Scope | Result |
| --- | --- | --- | --- | --- |
| BFCL | Yes | Yes | 5 `simple_python` cases | 5/5, accuracy 1.0 |
| GAIA | Gated | No | Blocked by Hugging Face auth | Requires `HF_TOKEN` and access approval |
| tau-bench original | Yes | Yes | 1 deprecated retail task | reward 1.0, Pass^1 1.0 |
| tau2/tau3 successor | Yes | Yes | 1 mock task | reward 1.0, Pass^1 1.0 |

## What This Proves

- The local environment can run official benchmark repositories, not only AgentEngine synthetic tests.
- The OpenAI-compatible model endpoint can execute BFCL function-calling evaluation and tau-style tool-agent-user interaction evaluation.
- The SDK evaluation path should use official raw datasets as external conformance gates, while keeping AgentEngine-specific conformance tests separate.

## What Is Not Yet Proven

- Full BFCL leaderboard coverage across all categories.
- Full tau2/tau3 airline, retail, telecom, banking, and voice domains.
- GAIA validation, because dataset access is gated.
- Statistically stable pass@k/pass^k over multiple seeds and trials.
- Cross-model comparison beyond the currently configured endpoint.

## Recommended Product Gate

For small-scope release validation:

1. BFCL: run at least 50 cases across simple, multiple, parallel, and relevance categories.
2. tau2/tau3: run at least 10 tasks each for retail, airline, and telecom with 3 trials.
3. GAIA: run validation level 1 once Hugging Face access is available.
4. Store every run under `eval/official/runs/<date>/<benchmark>/<model>/`.
5. Require reproducible command logs, raw result files, and summarized metrics before marking the SDK as release-ready.

