# 安装

Agentic Kernel 以两个 SDK 形式提供。安装你所用技术栈对应的那个——它们功能对等、版本
一致(**0.5.0**)。

> **访问权:** 包当前为**私有**(发布暂停)。安装需要 `agentic-kernel` GitHub 组织的
> 访问权与一个 token。代码为 **Apache-2.0**——获得访问权后即可使用、修改、再分发。

## 前置条件

- **TypeScript:** Node.js ≥ 20,npm(或 pnpm/yarn)。
- **Python:** Python ≥ 3.11,`pip`(或 `uv`)。

---

## TypeScript(`@agentic-kernel/*`)

发布在 **GitHub Packages**(npm registry)。

### 1. 创建 token

由维护者把你加入组织/仓库。然后在 <https://github.com/settings/tokens> 创建一个
**经典(classic)** 个人访问令牌,勾选 **`read:packages`** 范围,复制 `ghp_…`。

!!! warning
    细粒度令牌(`github_pat_…`)目前对 GitHub Packages 的 npm registry 会返回 403。
    请使用**经典**令牌。

### 2. 让 npm 指向该 registry

在项目里添加 `.npmrc`(**不要**提交令牌——放进环境变量或全局 `~/.npmrc`):

```ini
@agentic-kernel:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

```bash
export NODE_AUTH_TOKEN=ghp_your_classic_token
```

### 3. 安装

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

### 4. 验证

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

## Python(`agentic-kernel`)

单一分发包;内核是 `agentic_kernel.core`,适配器是同级子包。私有期间不在 PyPI——
从 Git 仓库安装。

### 从 Git 安装

```bash
# 仅内核(需要仓库访问权;HTTPS 会提示输入 token,或用 SSH)
pip install "agentic-kernel @ git+https://github.com/agentic-kernel/agentic-kernel-python.git"

# 连同可选适配器客户端(extras)
pip install "agentic-kernel[openai] @ git+https://github.com/agentic-kernel/agentic-kernel-python.git"
```

CI 中带 token:

```bash
pip install "agentic-kernel @ git+https://${GITHUB_TOKEN}@github.com/agentic-kernel/agentic-kernel-python.git"
```

**Extras** 只安装一个可注入的客户端(适配器运行时并不 import 它们):

| Extra | 引入 | 用于 |
| --- | --- | --- |
| `openai` | `httpx` | 远端模型(构建 `HttpFetch`) |
| `postgres` | `psycopg[binary]` | `memory_postgres` / `state_postgres` |
| `otel` | `opentelemetry-api`、`opentelemetry-sdk` | `observer_otel` |
| `all` | 以上全部 | 全部 |

### 从源码(开发)

```bash
git clone https://github.com/agentic-kernel/agentic-kernel-python.git
cd agentic-kernel-python
uv sync                       # 或:pip install -e ".[all]"
uv run pytest                 # 运行测试套件
```

### 验证

```python
import agentic_kernel as ak
print(ak.__version__)         # 0.5.0
from agentic_kernel.core import create_agent_engine
```

会安装一个控制台脚本 `agentic-kernel-eval`(评测 CLI)。

---

下一步:[快速上手](getting-started.md) 会接入模型并运行你的第一个 agent。
