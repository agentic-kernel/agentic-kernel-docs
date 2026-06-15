# 安装

Agentic Kernel 以两个 SDK 形式提供,功能对等、版本一致(**0.5.1**),许可为
**Apache-2.0**。

## 前置条件

- **TypeScript:** Node.js ≥ 20,npm(或 pnpm/yarn)。
- **Python:** Python ≥ 3.11,`pip`(或 `uv`)。

---

## TypeScript(`@agentic-kernel/*`)

发布在 **npmjs.org**——公开,无需认证。

```bash
# 内核
npm install @agentic-kernel/core

# 按需安装适配器
npm install @agentic-kernel/model-openai      # 远端模型(OpenAI Responses API)
npm install @agentic-kernel/model-ondevice    # 本地/小型模型
npm install @agentic-kernel/multi-agent @agentic-kernel/state-postgres
```

**可用的包**(均在 `@agentic-kernel/` 下):`core`、`model-openai`、`model-ondevice`、
`memory-postgres`、`state-file`、`state-postgres`、`observer-otel`、`multi-agent`、
`distributed`、`evaluator`、`testing`、`conformance`。

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
```

包为 ESM、自带类型,并附带 npm **provenance**(来源证明)。

---

## Python(`agentic-kernel-runtime`)

发布在 **PyPI**。分发名为 `agentic-kernel-runtime`;**导入包名是 `agentic_kernel`**。

```bash
pip install agentic-kernel-runtime

# 连同可选适配器客户端(extras)
pip install "agentic-kernel-runtime[openai]"   # httpx,用于远端模型
pip install "agentic-kernel-runtime[all]"      # httpx + psycopg + opentelemetry
```

| Extra | 引入 | 用于 |
| --- | --- | --- |
| `openai` | `httpx` | 远端模型(构建 `HttpFetch`) |
| `postgres` | `psycopg[binary]` | `memory_postgres` / `state_postgres` |
| `otel` | `opentelemetry-api`、`opentelemetry-sdk` | `observer_otel` |
| `all` | 以上全部 | 全部 |

```python
import agentic_kernel as ak
print(ak.__version__)                 # 0.5.1
from agentic_kernel.core import create_agent_engine
```

会安装一个控制台脚本 `agentic-kernel-eval`(评测 CLI)。

### 从源码(开发)

```bash
git clone https://github.com/agentic-kernel/agentic-kernel-python.git
cd agentic-kernel-python
uv sync                       # 或:pip install -e ".[all]"
uv run pytest
```

---

下一步:[快速上手](getting-started.md) 会接入模型并运行你的第一个 agent。
