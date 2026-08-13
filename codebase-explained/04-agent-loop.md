# 04 — The Agent Loop: Turns, Steps, and the Inbox

The default driver is `ReactLoopAgent` (`packages/core/agent-loop/src/agent.ts`). It turns admitted input into model requests, tool executions, and a durable transcript.

## The unit hierarchy

```mermaid
flowchart TD
    TURN["TURN — one drain of admitted input<br/>ends when the model and its tools stop,<br/>or a terminal policy intervenes"]
    STEP["STEP — one model request + the tools its response called"]
    ROUND["ROUND — an outer policy iteration containing a turn<br/>(goal round, Ralph round)"]
    ROUND --> TURN --> STEP
    TURN -->|"tools owe another request, or next-step input arrived"| STEP
```

## The phase machine

```mermaid
stateDiagram-v2
    [*] --> idle
    idle --> running: wakeDriver() — queued input wakes the driver
    running --> running: steps + continuation turns<br/>(fresh AbortController per turn)
    running --> idle: turn ends AND inbox empty
    idle --> maintenance: runMaintenance(task)<br/>(public status stays idle)
    maintenance --> idle: task done
    running --> running: wakeRequested latch while busy
    running --> idle: kick() finally — replay latched wake if inbox pending
```

## The inbox: two lanes, three intents

`Inbox` (`packages/core/agent/src/inbox.ts`) is a **durable projection** — every mutation appends an `agent/inbox/spliced` session event before mutating, so queue state replays from the log.

```mermaid
flowchart TD
    subgraph INBOX["agent.inbox"]
        NT["next-turn lane<br/>ordinary queued messages (followup)"]
        NS["next-step lane<br/>steering + injected context, consumed at step boundaries"]
    end
    F["agent.followup(msg) → next-turn, wakes"]
    S["agent.steer(msg) → next-step, wakes"]
    I["agent.inject(msg) → next-step, NEVER wakes<br/>(waits until another message does)"]
    F --> NT
    S --> NS
    I --> NS
    NS -->|"claim(target, turn):<br/>entire next-step batch + one next-turn message"| CL["pre-step claim"]
    CL --> NT
```

## The turn, as a sequence

Durable events are `turn/*`, `step/*`, `user/message`, `assistant/*`, `tool/*`; everything else is live coordination. (Condensed from `docs/agent-lifecycle.md`; the driver source is `packages/core/agent-loop/src/agent.ts:246-401`.)

```mermaid
sequenceDiagram
    participant IN as Inbox
    participant DR as Driver (ReactLoopAgent)
    participant PS as agent/pre-step waterfall
    participant REQ as agent/request waterfall
    participant LLM as ctx.llm (llm/stream waterfall)
    participant TL as ctx.tools
    participant LOG as Session log

    DR->>LOG: turn/start {turn}
    DR->>IN: claim(next-turn): one message + whole next-step batch
    DR->>PS: agent/pre-step {messages, turn, step} — authoritative decision
    alt reject / empty first claim
        DR->>LOG: turn/end {kind: blocked|completed} — no step spent
    else enter
        DR->>LOG: step/start {turn, step}
        DR->>LOG: user/message per entered message (surfaceOp: append)
        DR->>REQ: agent/request → config proposal (provider/model/tools)
        DR->>LLM: llm/stream(request)
        LLM-->>DR: StreamChunk*
        DR->>LOG: assistant/chunk {turn, step, chunk} — per chunk
        DR->>LOG: assistant/message {message, usage, sourceEventSeqs}
        alt response has tool calls
            loop classify by executionMode; barriers + rolling pool
                DR->>LOG: tool/call
                DR->>TL: pre-execute → execute → post-execute
                DR->>LOG: tool/result (cites its tool/call seq)
            end
            DR->>LOG: step/end
            Note over DR: tools owe another request → claim next-step → next step
        else no tool calls / max-tokens
            DR->>LOG: step/end
        end
        opt natural stop and inbox empty
            DR->>PS: agent/turn-stopping (serial, no next()) — terminal checkpoint
        end
    end
    DR->>LOG: turn/end {reason}
```

### What `agent/pre-step` can do

Listeners wrap `next()`; the returned decision is **authoritative**:

```mermaid
flowchart TD
    PS["agent/pre-step waterfall (default: enter claimed messages)"]
    PS --> R1["reject — drop the claim; turn closes with no step (log still records it)"]
    PS --> R2["enter(messages) — rewrite what the model sees (compaction, hooks, plan mode live here)"]
```

### What `agent/request` and `agent/request-error` can do

```mermaid
flowchart TD
    RQ["agent/request waterfall over a frozen seed config"]
    RQ --> OK["adapter resolves, request/header + request/context logged, request deep-frozen & dispatched"]
    ERR["agent/request-error waterfall (on error/aborted finish)"]
    ERR --> RETRY["listener returns {kind: 'retry'} → re-derive + re-stream<br/>WITHIN the same step (llm-retry lives here)"]
    ERR --> FAIL["otherwise LlmError propagates → turn ends kind:error"]
```

## The model-history invariant

Each step calls `session.deriveMessages()` — a pure projection of the log (see [05-session-log.md](05-session-log.md)) — and an invariant companion (`packages/core/agent-loop/src/invariant.ts:39`) asserts the dispatched request **byte-equals** the projection. The model can never see anything the log cannot reconstruct.

## Cancellation and error recovery

```mermaid
flowchart TD
    CANCEL["agent.cancel(cause) — first cause wins<br/>{user|parent|hook|disposed}"]
    CANCEL --> A["inbox.clear() → durable canceled splices<br/>(next-step first, keepInbox opts out)"]
    CANCEL --> B["abort the active turn/maintenance controller"]
    B --> C["turn() catch: abort → turn/end {kind: aborted}<br/>(every boundary calls signal.throwIfAborted())"]
    B --> D["disposal: cancel(disposed) → whenIdle() → scope.dispose() → registry detach"]
    TOOL["tool canceled before body → synthetic tool/call + tool/result pair<br/>keeps replay valid; started bodies drain to quiescence"]
    CRASH["crash mid-turn → persistence repair closes the orphaned turn<br/>with turn/end {kind: interrupted} (repair.ts)"]
```

## Agent lifecycle and goals

- The `Agent` interface (`packages/core/agent/src/runtime-types.ts:64`) exposes `session`, `inbox`, `status`, `ctx` (the agent's scoped context), `cancel`, `whenIdle`, `runMaintenance`, `send/followup/steer/inject`.
- `AgentHandle.dispose()` is a capability: only the holder can tear the agent down (`packages/core/agent/src/index.ts:172`).
- **Goal rounds**: `dsh-goal` keeps a durable objective on the session; `goal-round-driver` materializes each goal round as one goal-sourced turn. Activation must be re-armed by a human mutation after resume/fork (`docs/glossary.md` goal section).

Next: [05-session-log.md](05-session-log.md) — the durable source of truth everything derives from.
