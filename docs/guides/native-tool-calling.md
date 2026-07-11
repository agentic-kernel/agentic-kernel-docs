# Native tool calling

Since **0.8.0**, `OpenAIChatPlanner` is the recommended planner for complex and
long tasks. It speaks `/chat/completions` — so it works against OpenAI and every
OpenAI-compatible gateway (vLLM, Ollama, Groq, DeepSeek, …) — and uses
provider-native tool calling instead of one-action JSON output:

- **Per-tool schemas.** Your `ToolDefinition.inputSchema` reaches the provider
  as a native `tools` entry, so arguments are constrained per tool.
- **Parallel tool calls.** `tool_calls[]` map to the kernel's parallel
  `ToolCallAction[]` batch — fan-out costs one round instead of N.
- **Prompt-cache friendly.** The prompt is an append-only message transcript
  with a stable prefix, so provider caching stays effective as runs grow.
- **Real streaming.** Answers stream as content deltas; no JSON scraping.
- **Kernel-stamped identity.** The model never mints `id`/`createdAt` — the
  kernel stamps and de-duplicates action identity for the whole run.

=== "TypeScript"

    ```ts
    import { OpenAIChatPlanner } from "@agentic-kernel/model-openai";

    const planner = new OpenAIChatPlanner({
      apiKey: process.env.OPENAI_API_KEY!,
      model: "gpt-5.1",
      // baseUrl: "http://localhost:11434/v1",  // any OpenAI-compatible gateway
      parallelToolCalls: true,
      temperature: 0.2
    });
    ```

=== "Python"

    ```python
    import httpx
    from agentic_kernel.model_openai import OpenAIChatPlanner, OpenAIChatPlannerOptions
    from agentic_kernel.model_openai.httpx_fetch import httpx_fetch

    client = httpx.AsyncClient()
    planner = OpenAIChatPlanner(
        OpenAIChatPlannerOptions(
            api_key="...",
            model="gpt-5.1",
            fetch=httpx_fetch(client),   # reference HttpFetch — ships with [openai]
            parallel_tool_calls=True,
        )
    )
    ```

Control actions ship as a small synthetic-tool set the model can call:
`ask_user` (pauses at `waiting_for_user` — see [Sessions](sessions.md)) and
`stop` always; `push_goal`/`complete_goal` behind `enableGoalTools`. A control
call always wins over tool calls beside it, because kernel batches must be
tool-only.

The Responses-API planner (`OpenAIPlanner`) remains available for hosts that
want strict single-action structured output.
