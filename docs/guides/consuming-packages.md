# Installation

Agentic Kernel ships as two SDKs, feature-paired at the same version (**0.7.0**),
licensed **Apache-2.0**.

## Prerequisites

- **TypeScript:** Node.js ≥ 20, npm (or pnpm/yarn).
- **Python:** Python ≥ 3.11, `pip` (or `uv`).

---

## TypeScript (`@agentic-kernel/*`)

Published to **npmjs.org** — public, no auth required.

```bash
# the kernel
npm install @agentic-kernel/core

# adapters, as needed
npm install @agentic-kernel/model-openai      # remote models (OpenAI Responses API)
npm install @agentic-kernel/model-ondevice    # local / small models
npm install @agentic-kernel/multi-agent @agentic-kernel/state-postgres
```

**Available packages** (all under `@agentic-kernel/`): `core`, `model-openai`,
`model-ondevice`, `memory-postgres`, `state-file`, `state-postgres`, `observer-otel`,
`multi-agent`, `distributed`, `evaluator`, `testing`, `conformance`.

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
```

Packages are ESM, ship their own types, and carry npm **provenance** attestations.

---

## Python (`agentic-kernel-runtime`)

Published to **PyPI**. The distribution is `agentic-kernel-runtime`; the **import
package is `agentic_kernel`**.

```bash
pip install agentic-kernel-runtime

# with optional adapter clients (extras)
pip install "agentic-kernel-runtime[openai]"   # httpx, for remote models
pip install "agentic-kernel-runtime[all]"      # httpx + psycopg + opentelemetry
```

| Extra | Pulls in | For |
| --- | --- | --- |
| `openai` | `httpx` | remote models (build an `HttpFetch`) |
| `postgres` | `psycopg[binary]` | `memory_postgres` / `state_postgres` |
| `otel` | `opentelemetry-api`, `opentelemetry-sdk` | `observer_otel` |
| `all` | all of the above | everything |

```python
import agentic_kernel as ak
print(ak.__version__)                 # 0.7.0
from agentic_kernel.core import create_agent_engine
```

A console script `agentic-kernel-eval` is installed for the evaluator CLI.

### From source (development)

```bash
git clone https://github.com/agentic-kernel/agentic-kernel-python.git
cd agentic-kernel-python
uv sync                       # or: pip install -e ".[all]"
uv run pytest
```

---

Next: the [Getting started](getting-started.md) walkthrough wires a model and runs
your first agent.
