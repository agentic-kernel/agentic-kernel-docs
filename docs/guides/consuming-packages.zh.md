# 安装与引入 @agentic-kernel 包

`@agentic-kernel/*` 包发布在 **GitHub Packages**。仓库私有期间,包也为私有——安装需要
一个可访问 `agentic-kernel` 组织的 token。

代码以 **Apache-2.0** 授权(见 `LICENSE`):获得访问权后,你可在该许可下使用、修改、
再分发。

## 1. 获取访问权

由维护者把你加为仓库协作者(或加入组织)。然后创建一个**经典(classic)** 个人访问
令牌:

1. https://github.com/settings/tokens → *Generate new token (classic)*
2. 勾选范围:**`read:packages`**
3. 复制 `ghp_...` 令牌。

> 注意:细粒度令牌(`github_pat_...`)目前对 GitHub Packages 的 npm registry 会返回
> 403,请使用经典令牌。

## 2. 让 npm 指向 GitHub Packages

在消费项目里添加 `.npmrc`(不要提交令牌——用环境变量或个人/全局 `~/.npmrc`):

```ini
@agentic-kernel:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

然后在 shell 或 CI 中导出令牌:

```bash
export NODE_AUTH_TOKEN=ghp_your_classic_token
```

## 3. 安装

```bash
npm install @agentic-kernel/core
# 按需安装适配器:
npm install @agentic-kernel/model-openai @agentic-kernel/state-postgres
```

## 4. 使用

```ts
import { createAgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
```

完整 API 见各包 README 与本文档站。

## 已发布的包

`core`、`testing`、`conformance`、`evaluator`、`model-openai`、`model-ondevice`、
`memory-postgres`、`observer-otel`、`multi-agent`、`distributed`、`state-file`、
`state-postgres` —— 均在 `@agentic-kernel/` 作用域下。

## CI 消费方

在 GitHub Actions 中,用对本组织有 `read:packages` 的令牌认证(仓库的
`secrets.GITHUB_TOKEN` 仅在本仓库内有效;跨仓库需把 PAT 存为 secret):

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
