# Sessions & multi-turn

Since **0.8.0** a run can hold a conversation. When the planner emits
`ask_user`, the run parks at `waiting_for_user` — and the host answers it by
resuming with the reply:

=== "TypeScript"

    ```ts
    const turn1 = await engine.run({ agent, task, host });
    // turn1.status === "waiting_for_user"

    const turn2 = await engine.resume({
      runId: turn1.state.runId,
      agent, task, host,
      userMessage: "use the staging environment"
    });
    ```

=== "Python"

    ```python
    turn1 = await engine.run(agent=agent, task=task, host=host)
    # turn1.status == "waiting_for_user"

    turn2 = await engine.resume(
        agent=agent, task=task, host=host,
        run_id=turn1.state.run_id,
        user_message="use the staging environment",
    )
    ```

The kernel stamps the reply as a durable user `Message`, emits a
`user_message` event, and continues the loop. Replies are **replay-covered**:
`state.messages` reconstructs exactly from the event log (the same is true for
kernel-injected reflection messages). Context-driven memory recall picks the
latest user message up automatically, so recall retargets to the answer.

## The Session convenience

`createSession` / `create_session` wraps one task's lifecycle, including its
clarification turns:

=== "TypeScript"

    ```ts
    import { createSession } from "@agentic-kernel/core";

    const session = createSession(engine, { agent, host });
    const first = await session.send("deploy the service");   // may ask back
    const second = await session.send("staging, please");     // answers it
    ```

=== "Python"

    ```python
    from agentic_kernel.core import create_session

    session = create_session(engine, agent=agent, host=host)
    first = await session.send("deploy the service")
    second = await session.send("staging, please")
    ```

After a terminal status the session is sealed (`SESSION_COMPLETED`) — a new
task is a new session. Approval and schedule waits are host workflows, not user
turns, so `send` refuses them (`SESSION_NOT_AWAITING_USER`).

## Long-lived runs across upgrades

Runs persisted under an older schema version now migrate on load: the durable
stores (`state-file`, `state-postgres`) upgrade the snapshot along the known
version chain (option `migrate`, default on), so a run parked last month
resumes under this month's kernel. Unknown versions still fail hard — migration
is a designed path, not a guess.
