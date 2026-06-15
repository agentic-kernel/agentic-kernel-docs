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

## Scopes

Memory is tenant-namespaced: a scope is `tenant/[workspace/][agent]`. A seeded item's
`scope` must be **within** the agent's scope prefix to be retrievable — e.g. for tenant
`acme`, agent `researcher`, an item with scope `acme/researcher` matches.

## Production & governance

For durable storage use `memory-postgres` (or any `MemoryManager`). Long-term memory
has a governance lifecycle — `proposeWrite` / `supersede` / `tombstone` / `explain` /
`export` — and `SessionMemoryExtractor` / a memory governor can extract attributable
session summaries automatically on completion.
