# 05 — The Session Log: Event Sourcing and the Surface

`dsh-session` (`packages/core/session`) owns an **append-only event log** per session. Everything the model sees, everything the UI renders, and everything persisted derives from this stream. `ctx.sessions` is the store.

## Event sourcing

```mermaid
flowchart LR
    P["Plugin code"] -->|"session.append(type, data, surfaceOpts)"| S["Session: append-only SessionEvent[]<br/>seq = log.length"]
    S --> F["lossless JSON snapshot + deep-freeze<br/>surface transition validated"]
    F --> E["broadcast session/event (per-listener error containment)"]
    E --> LIST["listeners: persistence, projection, telemetry,<br/>titles, token-meter, UI bridges, acp, apiproxy..."]
```

## Durable events (SessionEventMap)

```mermaid
flowchart TD
    subgraph STRUCT["Turn/structure"]
        A1["turn/start, turn/end {reason}"]
        A2["step/start, step/end"]
    end
    subgraph MSG["Messages"]
        B1["user/message"]
        B2["assistant/chunk (raw stream fidelity)"]
        B3["assistant/message {usage, sourceEventSeqs}"]
    end
    subgraph TOOL["Tools"]
        C1["tool/call"]
        C2["tool/result (cites its call seq)"]
    end
    subgraph CTX["Context"]
        D1["request/header (provider/model/tools)"]
        D2["request/context (route/context-window change)"]
        D3["todo/write, session/end-seed, goal/*, llm/retry..."]
    end
```

- `SESSION_FORMAT_VERSION = 0` (`packages/core/session/src/types.ts:56`) — no compatibility promise pre-release; structural changes bump it.
- New event types ride the per-event `ignorable` envelope: builds that don't know a type refuse the log unless the event is marked `ignorable: true`.

## The surface: which events become model-visible messages

Only three event types produce messages: `user/message`, `assistant/message`, `tool/result` (`packages/core/session/src/surface.ts:15`). `SurfaceManager` folds the log into an ordered node list, so history is a **pure projection**:

```mermaid
flowchart TD
    LOG["append-only log<br/>chunks, boundaries, calls, results..."]
    SURF["SurfaceManager: nodes = [seqs in model-visible order]<br/>append ops add; replace ops splice a shadowed range"]
    DM["deriveMessages(): walk nodes, project each once:<br/>user/message → verbatim<br/>assistant/message → its message (empty content → null)<br/>tool/result → ToolResultMessage (toolCallId correlation)<br/>everything else → null"]
    CACHE["per-node projection cache<br/>a replace bumps replaceGeneration → rebuild"]
    LOG --> SURF --> DM --> CACHE
```

A `tool/result` **must** cite its `tool/call` via `sourceEventSeqs: [callSeq]` and the result block's `toolCallId` must equal the call id — validated at append/load (`packages/core/session/src/index.ts:301`). Compaction shrinks history by appending a `replace` surface op that shadows deleted nodes, keeping replay honest.

## Persistence write path

```mermaid
flowchart TD
    SE["session/event"] -->|"enqueue"| WB["SessionWriteBehind: per-session bounded batching<br/>structuredClone queue, 200 ms deadline, failure re-queued"]
    WB --> PC["PersistenceCoordinator.appendCore:<br/>event.seq === cursor + i (contiguous contract)"]
    PC --> BE["backend.appendBatch → materialized, cursor advances<br/>the durable write IS the transaction"]
    BE --> JSONL["JSONL backend: one append-only file per session<br/>temp write + fsync + link() publish"]
    BE --> SQLITE["SQLite backend: 1:1 events rows, one transaction<br/>SCHEMA_VERSION = 15"]
    FL["session/flush (parallel, awaited)"] --> WB
    CP["session-checkpoint-policy: flush before every<br/>loop-built model dispatch, tool exec, pre-step"] --> FL
```

Crash recovery (`packages/session/session-persistence/src/coordinator.ts:892`): on load, a torn tail is repaired and any orphaned open turn gets a synthetic `turn/end {kind: 'interrupted'}`.

## Projection: derived state over the same stream

```mermaid
flowchart LR
    DEF["ProjectionDefinition:<br/>init() / apply(state, event) / view(state)<br/>pure sync functions + stateVersion"]
    REG["SessionProjectionRegistry:<br/>one session/event subscription,<br/>eagerly folds every registered unit"]
    READ["snapshot(session) — synchronous consistent cut"]
    READ2["checkpoint(session) — write side of the persisted cache"]
    REG --> READ & READ2
```

Consumers: `token-meter` (usage/context-pressure), `session-stats` (whole-log stats). The projection cache (`dsh-session-projection-cache`) is a fold shortcut, never an authority — stale is allowed, wrong is not.

## What derives from the log

```mermaid
flowchart TD
    LOG["the session log"]
    LOG --> M["model history (deriveMessages)"]
    LOG --> R["replay (assistant/chunk*) and UI rendering"]
    LOG --> F["fork: copy up to a boundary seq"]
    LOG --> T["titles, stats, telemetry, consumed-work accounting"]
    LOG --> S["storage: JSONL files / SQLite rows"]
    LOG --> X["export (ZIP transcript download)"]
```

Next: [06-scope-and-tools.md](06-scope-and-tools.md) — how tools register, scope, and execute.
