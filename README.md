# agentic-kernel-docs

Documentation site for **Agentic Kernel** — the microkernel agent runtime
(TypeScript + Python, feature-paired). Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
and published to GitHub Pages.

**Live site:** https://agentic-kernel.github.io/agentic-kernel-docs/

## Contents

- **Architecture** — the run loop, data model, and injection seams.
- **Guides** — getting started, consuming packages, memory, on-device models, multi-agent orchestration.
- **Manuals** — master developer guide, exhaustive manual, 用户手册（中文）.
- **Reports** — real-LLM evaluation, capability, and stress campaigns.

Source docs are aggregated here; package READMEs live in their code repos
([agentic-kernel](https://github.com/agentic-kernel/agentic-kernel),
[agentic-kernel-python](https://github.com/agentic-kernel/agentic-kernel-python)).

## Develop locally

```bash
pip install "mkdocs-material>=9.5"
mkdocs serve   # http://127.0.0.1:8000
```

Pushes to `main` build and deploy automatically via `.github/workflows/deploy.yml`.

## License

Apache-2.0.
