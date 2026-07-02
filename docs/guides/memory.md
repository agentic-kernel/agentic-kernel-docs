# Memory

Memory is injected behind the `MemoryManager` seam. The `ContextBuilder` retrieves
in-scope items and injects them into the planner context (emitting `memory_retrieved`
/ `memory_injected`). Core ships three reference managers.

## InMemoryMemory — term-overlap relevance

Retrieval ranks in-scope items by **how many query terms appear in the content** (a
question surfaces a fact that shares words), *not* substring containment. `preference`
items are always floored in so standing instructions surface. Ties break by item score,
then recency. For tests and local examples.

## EmbeddingMemory — semantic recall

Ranks in-scope items by cosine similarity between the query embedding and each item's
content embedding. The embedder is **injected** — no model is bundled, so it runs in
any runtime. A text-less query falls back to score/recency.

=== "TypeScript"

    ```ts
    import { EmbeddingMemory, type Embedder } from "@agentic-kernel/core";

    const embed: Embedder = async (text) => myEmbeddingModel(text); // number[]
    const memory = new EmbeddingMemory({ embed });
    // new AgentEngine({ planner, policy, memory, ... })
    ```

=== "Python"

    ```python
    from agentic_kernel.core import EmbeddingMemory

    async def embed(text: str) -> list[float]:
        return await my_embedding_model(text)

    memory = EmbeddingMemory(embed=embed)
    # engine = create_agent_engine(planner=..., policy=..., memory=memory)
    ```

## Context-driven recall

Since **0.6.1**, the recall query *follows the run* instead of repeating the opening
task string. Each round, `ContextBuilder` composes the retrieval query from:

1. the task input (opening intent),
2. the latest user message (if it differs from the task input), and
3. snippets of recent **successful** observation results (where the run actually went),

capped at 2000 characters (observation snippets at 200 each). A run that starts
"fix the homepage layout" and drifts to "wire the staging DB" now recalls
staging-DB memory mid-run instead of only homepage memory.

On the first round (no messages or observations yet) the query is exactly
`task.input`, so turn-1 behavior is unchanged. This is internal query construction —
no API or schema change; hosts get it automatically.

## Scopes

Memory is tenant-namespaced: a scope is `tenant/[workspace/][agent]`. A seeded item's
`scope` must be **within** the agent's scope prefix to be retrievable — e.g. for tenant
`acme`, agent `researcher`, an item with scope `acme/researcher` matches.

## Production & governance

For durable storage use `memory-postgres` (or any `MemoryManager`). Long-term memory
has a governance lifecycle — `proposeWrite` / `supersede` / `tombstone` / `explain` /
`export` — and `SessionMemoryExtractor` / a memory governor can extract attributable
session summaries automatically on completion.
