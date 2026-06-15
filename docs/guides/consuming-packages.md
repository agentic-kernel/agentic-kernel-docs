# Installation

Agentic Kernel ships as two SDKs. Install the one for your stack — they are
feature-paired at the same version (**0.5.0**).

> **Access:** the packages are currently **private** (publishing paused). Installing
> them requires access to the `agentic-kernel` GitHub organization and a token. The
> code is **Apache-2.0** — once you have access you may use, modify, and redistribute it.

## Prerequisites

- **TypeScript:** Node.js ≥ 20, npm (or pnpm/yarn).
- **Python:** Python ≥ 3.11, `pip` (or `uv`).

---

## TypeScript (`@agentic-kernel/*`)

Published to **GitHub Packages** (npm registry).

### 1. Create a token

A maintainer adds you to the org/repo. Then create a **classic** Personal Access
Token at <https://github.com/settings/tokens> with the **`read:packages`** scope and
copy the `ghp_…` value.

!!! warning
    Fine-grained tokens (`github_pat_…`) currently return 403 against GitHub Packages'
    npm registry. Use a **classic** token.

### 2. Point npm at the registry

Add an `.npmrc` to your project (do **not** commit the token — keep it in an env var
or your global `~/.npmrc`):

```ini
@agentic-kernel:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

```bash
export NODE_AUTH_TOKEN=ghp_your_classic_token
```

### 3. Install

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

### 4. Verify

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
console.log(typeof createAgentEngine); // "function"
```

### CI

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    registry-url: https://npm.pkg.github.com
    scope: "@agentic-kernel"
- run: npm ci
  env:
    NODE_AUTH_TOKEN: ${{ secrets.AGENTIC_KERNEL_READ_TOKEN }}
```

---

## Python (`agentic-kernel`)

Single distribution; the kernel is `agentic_kernel.core`, adapters are sibling
subpackages. Not on PyPI while private — install from the Git repository.

### Install from Git

```bash
# core only (needs repo access; HTTPS will prompt for a token, or use SSH)
pip install "agentic-kernel @ git+https://github.com/agentic-kernel/agentic-kernel-python.git"

# with optional adapter clients (extras)
pip install "agentic-kernel[openai] @ git+https://github.com/agentic-kernel/agentic-kernel-python.git"
```

With a token in CI:

```bash
pip install "agentic-kernel @ git+https://${GITHUB_TOKEN}@github.com/agentic-kernel/agentic-kernel-python.git"
```

**Extras** install only an injectable client (the adapters import none of these at
runtime):

| Extra | Pulls in | For |
| --- | --- | --- |
| `openai` | `httpx` | remote models (build an `HttpFetch`) |
| `postgres` | `psycopg[binary]` | `memory_postgres` / `state_postgres` |
| `otel` | `opentelemetry-api`, `opentelemetry-sdk` | `observer_otel` |
| `all` | all of the above | everything |

### From source (development)

```bash
git clone https://github.com/agentic-kernel/agentic-kernel-python.git
cd agentic-kernel-python
uv sync                       # or: pip install -e ".[all]"
uv run pytest                 # run the test suite
```

### Verify

```python
import agentic_kernel as ak
print(ak.__version__)         # 0.5.0
from agentic_kernel.core import create_agent_engine
```

A console script `agentic-kernel-eval` is installed for the evaluator CLI.

---

Next: the [Getting started](getting-started.md) walkthrough wires a model and runs
your first agent.
