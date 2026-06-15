# AgentEngine Master Developer Guide

The definitive guide to building robust, real-time, and scalable agentic applications using the Agentic Kernel SDK.

---

## 1. Architecture Philosophy: The Microkernel Approach

AgentEngine is designed as a **deterministic state machine** and **intelligent router**. It does not own your models, databases, or tools; instead, it provides a high-fidelity "Kernel" that coordinates these components through strict contracts.

### The Dual-Track Execution Model

To solve the conflict between "atomic reliability" and "real-time feedback," AgentEngine employs a dual-track architecture:

1.  **The Reliable Track (Atomic)**: 
    *   **Components**: `StateStore`, `PolicyManager`, `State`.
    *   **Goal**: Ensures every action and observation is durably recorded and governed. If a run crashes, it can be resumed with 100% fidelity.
    *   **Behavior**: Events are written only when a step is fully validated and committed.

2.  **The Observability Track (Real-time)**:
    *   **Components**: `Observer`, `runStream`, `PlannerChunk`.
    *   **Goal**: Provides zero-latency feedback to the UI.
    *   **Behavior**: Emits "thinking" tokens and "live status" updates instantly, bypassing the durable write-lock to ensure the UI never feels "stuck."

---

## 2. Core Components & Setup

### Package Selection

| Package | Purpose | Mandatory? |
| :--- | :--- | :--- |
| `@agentic-kernel/core` | Core runtime, contracts, and base adapters. | **Yes** |
| `@agentic-kernel/model-openai` | Planner implementation for OpenAI-compatible APIs (Responses API). | No |
| `@agentic-kernel/multi-agent` | Coordination logic for sub-agents and task graphs. | No |
| `@agentic-kernel/state-postgres` | Durable Postgres persistence for production. | Recommended |

### Minimal Real-time Example

The modern way to run AgentEngine is via `runStream`, which provides a first-class `AsyncIterable` event stream.

```typescript
import { AgentEngine, BasicPolicy, StreamingObserver } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

const engine = new AgentEngine({
  planner: new OpenAIPlanner({ apiKey: "..." }),
  policy: new BasicPolicy({ maxSteps: 10 }),
  // stateStore: new PostgresStateStore(...), // For production
});

// Create the run
const runInput = {
  agent: { id: "expert-1", role: "Researcher", ... },
  task: { id: "task-101", input: "Research current AI trends" },
  host: { tenantId: "org-a", workspaceId: "team-1", ... }
};

// Consume the event stream
for await (const event of engine.runStream(runInput)) {
  switch (event.type) {
    case "reasoning_chunk":
      process.stdout.write(event.payload.content); // Stream thinking tokens
      break;
    case "action_selected":
      console.log(`\nDecision: ${event.payload.actionType}`);
      break;
    case "task_completed":
      console.log(`\nFinal Answer: ${event.payload.output}`);
      break;
  }
}
```

---

## 3. Advanced Streaming & UI Routing

### UI Routing Alignment (The Root-ID Pattern)

In multi-agent or concurrent scenarios, the Host app needs to know exactly where to route a token (e.g., which chat bubble?). AgentEngine solves this by **injecting routing IDs at the root level** of every event:

```json
{
  "id": "ev-123",
  "type": "reasoning_chunk",
  "runId": "run-456",
  "agentId": "specialist-a",  // Root-level for fast routing
  "taskId": "subtask-789",    // Root-level for UI mapping
  "step": 2,                  // Current execution step
  "payload": { ... }
}
```

**Developer Rule**: Always use the root-level `agentId` and `taskId` in your UI dispatcher/Electron IPC bridge to ensure "thinking" tokens appear in the correct component.

### Intelligent Batching

To prevent "IPC flooding" (which can lag Electron or Web browsers), AgentEngine automatically batches high-frequency tokens:
*   **Async Planners**: Flushed every 10 tokens (approx. 500ms latency).
*   **Sync Planners**: Aggregated via Microtasks (aggregated before the next tick).

---

## 4. Multi-Agent Orchestration

### Fractal Delegation

Sub-agents in AgentEngine are "Fractal": a sub-agent is just another `AgentEngine` run. 

**Key Features**:
1.  **Observer Inheritance**: When an Orchestrator spawns a sub-agent, the parent's `runStream` automatically captures the sub-agent's events. The user sees a single, unified stream of "The Orchestrator is thinking... The Specialist is thinking... Specialist finished... Orchestrator is answering."
2.  **Seed Memory**: You can "warm up" a sub-agent by injecting `seedMemory` into its task spec, giving it context without requiring a new retrieval phase.

```typescript
// Subagent execution automatically propagates events back to the parent stream
const result = await engine.run({
  ...parentContext,
  observer: parentObserver // Inheritance happens here
});
```

---

## 5. Proactive Memory Governance

AgentEngine distinguishes between **Ambient Context** (passive retrieval) and **Proactive Knowledge** (active writing).

### The Memory Tool Pattern

Never allow an Agent to write directly to a database. Instead, expose memory operations as **Tools**. This ensures they are governed by `Policy` and can require `Human Approval`.

```typescript
// Recommended pattern: Capture MemoryManager in a tool closure
export function createMemoryWriteTool(memory: LongTermMemoryManager) {
  return {
    name: "propose_fact",
    execute: async (args, context) => {
      // Logic to propose a new fact to the user's permanent memory
      return await memory.proposeWrite({ ... }, [context.state.runId]);
    }
  };
}
```

---

## 6. Production Hardening Checklist

1.  **State Isolation**: Ensure your `StateStore` implementation uses `runId` as the primary partition key.
2.  **Concurrency Safety**: Use **Run-level Observers** (passing `observer` to `.run()`) instead of instance-level observers to avoid event cross-talk in multi-tenant environments.
3.  **Human-in-the-Loop**: Implement an `ApprovalProvider` for any tools with `sideEffectLevel: "destructive"`.
4.  **Contract Validation**: Use `@agentic-kernel/conformance` to verify your custom Postgres or VectorDB adapters before deploying.

---

## 7. Troubleshooting "The Blank Screen"

If your UI shows "Thinking..." but never updates:
1.  **Check IDs**: Ensure your UI component is listening for the specific `taskId` provided in the `AgentEvent`.
2.  **Verify Flush**: Ensure your Planner yields `PlannerChunk` events. If using a custom Planner, call the `onChunk` callback frequently.
3.  **Inheritance**: If the missing tokens are from a sub-agent, verify that the `DelegationManager` is receiving the parent's `Observer`.

---

*Generated by AgentEngine Architecture Team, May 2026.*
