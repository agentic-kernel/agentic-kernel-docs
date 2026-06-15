# AgentEngine Exhaustive Developer Manual

**Version**: 1.0 (May 2026)  
**Status**: Production Ready  
**Scope**: Core Runtime, Streaming, Multi-Agent, and UI Integration.

---

## Table of Contents

1.  [Architectural Foundations](#1-architectural-foundations)
2.  [Getting Started](#2-getting-started)
3.  [The Core Loop: Sense-Plan-Act](#3-the-core-loop-sense-plan-act)
4.  [Streaming Deep Dive](#4-streaming-deep-dive)
5.  [UI Integration & Event Routing](#5-ui-integration--event-routing)
6.  [Tool Development Guide](#6-tool-development-guide)
7.  [Memory & Context Management](#7-memory--context-management)
8.  [Multi-Agent & Delegation Systems](#8-multi-agent--delegation-systems)
9.  [Safety, Policy & Approvals](#9-safety-policy--approvals)
10. [Persistence & State Management](#10-persistence--state-management)
11. [Testing & Validation](#11-testing--validation)

---

## 1. Architectural Foundations

### The Microkernel Philosophy
AgentEngine follows a **microkernel architecture**. The SDK itself is a pure state machine and event router. It does not contain pre-defined models or databases.

*   **Host-Driven**: The host provides the "Body" (Tools, Models, DBs).
*   **SDK-Governed**: The SDK provides the "Brain" (Loop, Logic, Safety).

### The Dual-Track Execution Model
The SDK operates on two parallel tracks to ensure both reliability and responsiveness:

| Track | Target | Data Type | Requirement |
| :--- | :--- | :--- | :--- |
| **Reliable Track** | Database / State | `AgentState`, `Log` | Atomic, Durable, Verified. |
| **Observability Track**| UI / WebSocket | `AgentEvent`, `Chunk`| Real-time, Ephemeral, Low-latency. |

---

## 2. Getting Started

### Installation
```bash
npm install @agentic-kernel/core @agentic-kernel/model-openai
```

### The "Hello World" Run
The most basic setup using a streaming OpenAI planner and in-memory storage.

```typescript
import { AgentEngine, BasicPolicy, ToolRegistry } from "@agentic-kernel/core";
import { OpenAIPlanner } from "@agentic-kernel/model-openai";

// 1. Setup Tools
const tools = new ToolRegistry();

// 2. Setup Engine
const engine = new AgentEngine({
  planner: new OpenAIPlanner({ apiKey: "sk-..." }),
  policy: new BasicPolicy({ maxSteps: 5 }),
  toolRegistry: tools
});

// 3. Execution (The Streaming Entry Point)
const stream = engine.runStream({
  agent: { id: "a1", role: "assistant", goal: "help user", instructions: "be brief", tools: [] },
  task: { id: "t1", input: "Explain Quantum Physics in one sentence." },
  host: { tenantId: "demo", principal: { id: "u1", type: "user", tenantId: "demo" }, effectiveScopes: [] }
});

for await (const event of stream) {
  if (event.type === "reasoning_chunk") {
    process.stdout.write(event.payload.content);
  }
}
```

---

## 3. The Core Loop: Sense-Plan-Act

AgentEngine executes in discrete **Steps**. Each step follows this lifecycle:

1.  **Context Construction**: SDK gathers messages, tools, and memory into an `AgentContext`.
2.  **Planning**: `Planner` receives the context and decides on an `Action` (e.g., call a tool or give a final answer).
3.  **Policy Validation**: `PolicyManager` checks if the action is allowed.
4.  **Execution**: `ToolScheduler` or `DelegationManager` executes the action.
5.  **Observation**: The result is recorded as an `Observation`, and the loop continues.

---

## 4. Streaming Deep Dive

### `runStream(input)`
This is the recommended entry point. It returns an `AsyncIterable<AgentEvent>`.

*   **Concurrency Safe**: Uses local state for chunk buffering.
*   **Self-Completing**: Automatically closes when the agent finishes.
*   **Composite**: Merges thinking tokens, tool calls, and lifecycle events into one stream.

### Event Types to Watch
| Event Type | Use Case |
| :--- | :--- |
| `reasoning_chunk` | Raw tokens from the model's thinking/reasoning phase. |
| `text_chunk` | Incremental tokens of the final answer (available in supported planners). |
| `tool_started` | Show a loading state for a specific tool. |
| `action_selected` | Indicate the agent has decided on its next move. |

---

## 5. UI Integration & Event Routing

### Precision Routing with Root-Level IDs
Every `AgentEvent` contains root-level metadata to help the UI route the data correctly:

```typescript
{
  id: string;        // Unique event ID
  type: string;      // Event type (e.g., reasoning_chunk)
  runId: string;     // The unique execution run
  agentId: string;   // Which agent produced this? (Crucial for multi-agent)
  taskId: string;    // Which task does this belong to? (Matches UI component)
  step: number;      // Current execution step
  payload: object;   // Data specific to the event type
}
```

### UI Handler Example (Pseudo-code)
```typescript
engine.runStream(input).on('event', (event) => {
  // Find the right chat bubble using taskId and agentId
  const bubble = UI.getBubble(event.taskId, event.agentId);
  
  if (event.type === "reasoning_chunk") {
    bubble.appendThinking(event.payload.content);
  } else if (event.type === "task_completed") {
    bubble.setFinalAnswer(event.payload.output);
  }
});
```

---

## 6. Tool Development Guide

### Writing a High-Fidelity Tool
Tools should be pure handlers that rely on the provided `context`.

```typescript
tools.register({
  name: "search_docs",
  description: "Search internal documentation",
  inputSchema: { ... },
  sideEffectLevel: "read",
  execute: async (args, context) => {
    // ALWAYS use context.host to derive permissions
    const results = await db.query(args.query, { tenant: context.host.tenantId });
    
    // Optional: Report progress for long-running tools
    context.reportProgress?.({ status: "searching_index" });
    
    return results;
  }
});
```

---

## 7. Memory & Context Management

### Passive Injection (Ambient Memory)
The SDK automatically calls `MemoryManager.retrieve` before each planning step.
*   **Best for**: "Read-only" context like user preferences or past relevant facts.

### Proactive Writing (The Tool Pattern)
Agents cannot write to memory directly. You must provide a "Memory Tool."

```typescript
// Define a tool that closes over your memory manager
const writeMemoryTool = {
  name: "store_fact",
  execute: async (args, context) => {
    await myMemoryManager.proposeWrite({ content: args.fact, ... });
    return { status: "success" };
  }
};
```

---

## 8. Multi-Agent & Delegation Systems

### How Delegation Works
1.  **Orchestrator** decides to delegate a sub-task.
2.  SDK calls `DelegationManager`.
3.  A **Sub-agent** is instantiated (it's just another AgentEngine run).
4.  The **Sub-agent** inherits the parent's `Observer`, allowing its thinking tokens to flow back to the parent's stream.

### Seed Memory for Specialists
When spawning a sub-agent, you can pass `seedMemory` to give it an immediate knowledge boost without a cold retrieval.

---

## 9. Safety, Policy & Approvals

### The Policy Pipeline
Every action is passed through the `PolicyManager`.
*   **Decision: Deny**: Fails the run immediately.
*   **Decision: Require Approval**: Suspends the run and returns a `waiting_for_approval` status.

### Human-in-the-loop (HITL)
```typescript
const result = await engine.run(input);

if (result.status === "waiting_for_approval") {
  // 1. Show UI to user
  // 2. User clicks "Approve"
  // 3. Resume the run
  await engine.resolveApproval({
    runId: result.state.runId,
    decision: "approved",
    ...
  });
}
```

---

## 10. Persistence & State Management

### Durable StateStore
In production, use `@agentic-kernel/state-postgres`.
*   **Runs are Resumeable**: If the process restarts, use `engine.resume({ runId: "..." })` to pick up exactly where it left off.
*   **Audit Trails**: Every event in the `Reliable Track` is logged, providing a perfect audit trail.

---

## 11. Testing & Validation

### Conformance Testing
If you implement your own `StateStore` or `Planner`, use the conformance suite:

```typescript
import { runStateStoreConformance } from "@agentic-kernel/conformance";

describe("MyCustomStore", () => {
  runStateStoreConformance(() => new MyCustomStore());
});
```

### Trace Assertions
Verify that your Agent actually used the right tools in the right order:

```typescript
const result = await engine.run(input);
expect(result.state.actions.map(a => a.toolName)).toContain("search_docs");
```

---

*End of Manual.*
