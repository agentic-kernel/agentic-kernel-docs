# Consuming @agentic-kernel packages

The `@agentic-kernel/*` packages are published to **GitHub Packages**. While the
repository is private, the packages are private too — installing them requires a
token with access to the `agentic-kernel` organization.

The code is licensed under **Apache-2.0** (see `LICENSE`): once you have access,
you may use, modify, and redistribute it under those terms.

## 1. Get access

A maintainer adds you as a collaborator on the repository (or to the org). You
then create a **classic** Personal Access Token:

1. https://github.com/settings/tokens → *Generate new token (classic)*
2. Scope: **`read:packages`**
3. Copy the `ghp_...` token.

> Note: fine-grained tokens (`github_pat_...`) currently return 403 against
> GitHub Packages' npm registry. Use a classic token.

## 2. Point npm at GitHub Packages

In the consuming project, add an `.npmrc` (do not commit the token — use an env
var or a personal/global `~/.npmrc`):

```ini
@agentic-kernel:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

Then export the token in your shell or CI:

```bash
export NODE_AUTH_TOKEN=ghp_your_classic_token
```

## 3. Install

```bash
npm install @agentic-kernel/core
# adapters, as needed:
npm install @agentic-kernel/model-openai @agentic-kernel/state-postgres
```

## 4. Use

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
```

See the package READMEs and `docs/` for the full API.

## Published packages

`core`, `testing`, `conformance`, `evaluator`, `model-openai`,
`memory-postgres`, `observer-otel`, `multi-agent`, `distributed`, `state-file`,
`state-postgres` — all under the `@agentic-kernel/` scope.

## CI consumers

In GitHub Actions, authenticate with a token that has `read:packages` for this
org (a repo `secrets.GITHUB_TOKEN` only works within this repo; cross-repo needs
a PAT stored as a secret):

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
