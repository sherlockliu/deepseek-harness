# 08 — The Event System: Domains and Dispatch

Events are the extension points. Picking the right domain is the first decision in most changes. (The authoritative matrix is the generated [event map](../docs/event-producer-consumer.md).)

## Two planes: durable vs live

```mermaid
flowchart TD
    subgraph DUR["Durable — session events<br/>(SessionEventMap, appended to the log, broadcast via session/event)"]
        D1["turn/* step/* user/message assistant/chunk assistant/message tool/call tool/result request/header todo/write goal/* ..."]
    end
    subgraph LIVE["Live — coordination events<br/>(agent/* tools/* fs/* llm/* ... never persisted)"]
        L1["agent/pre-step, agent/request, agent/request-error,<br/>agent/turn-stopping, agent/status, inbox events"]
        L2["tools/pre-execute, tools/execute, tools/post-execute, tools/result"]
        L3["llm/stream, llm/adapters-updated"]
        L4["fs/write-intent, fs/edit-intent, fs/observed"]
    end
    DUR -->|"use when the fact must survive a reload"| RULE
    LIVE -->|"use to observe or intercept work in flight"| RULE["Model-visible ⟺ logged:<br/>anything reaching a model request must be reconstructable<br/>from the log — new model-visible input needs a session event"]
```

## Dispatch-mode vocabulary

| Mode | Behavior | dsh examples |
|---|---|---|
| `emit` | sync fire-and-forget | `session/event`, `agent/status`, `tools/result`, `fs/observed` |
| `parallel` | await all listeners | `session/flush` |
| `serial` | in order, can bail | `agent/turn-stopping` |
| `bail` | sync until truthy | (registry checks) |
| `waterfall` | listeners wrap `next()`; not calling it short-circuits | `agent/pre-step`, `agent/request`, `agent/request-error`, `llm/stream`, `tools/*-execute`, `fs/*-intent`, `system-prompt/assemble`, `approval/request` |

**Waterfall semantics are the critical rule:** a listener must call `next()` to delegate; returning without it owns the decision. `agent/pre-step` decides what the model sees; `fs/write-intent` decides the freshness policy; `agent/request-error` decides retry.

## Domain taxonomy

```mermaid
flowchart TD
    EV["events"]
    EV --> AG["agent/* — inbox, status, pre-step, request,<br/>request-error, turn-stopping, error, session-start<br/>(declared in packages/core/agent/src/runtime-types.ts)"]
    EV --> SES["session/* — created, disposed, event, flush<br/>(the fan-out hub)"]
    EV --> LLM["llm/* — stream, adapters-updated"]
    EV --> TL["tools/* — pre/execute/post-execute, result, change, code-dispatch-log"]
    EV --> FS["fs/* — write-intent, edit-intent (waterfalls), observed (emit)"]
    EV --> SP["system-prompt/* — assemble (waterfall), change"]
    EV --> SA["subagent/* — start, end, provider-added/removed"]
    EV --> AP["approval/request (waterfall)"]
    EV --> WK["workflow/* — phase/log emit chain"]
    EV --> CR["credentials/updated, settings/updated, domain/changed,<br/>skills/change, goal/changed"]
```

## Typed events via declaration merging

```mermaid
flowchart LR
    subgraph MOD["Each declaring package"]
        M1["declare module '<pkg>' {<br/>  interface Events {<br/>    'fs/write-intent'(target, actor, next): Promise<FsWriteIntent | undefined><br/>  }<br/>}"]
    end
    subgraph MERGE["Merge-extensible maps"]
        M2["Events union — dispatch is typed across all packages"]
        M3["SessionEventMap union — payloads must carry the envelope's ignorable flag if optional"]
    end
    MOD --> MERGE
```

Session events additionally require JSDoc `@mode` + payload `@param`s; scoped keys absent from payloads need `@dshScopeScan unsupported`. `SessionEventMap` members are required-on-read by default.

## Scoped dispatch

```mermaid
flowchart TD
    D["event about one agent's activity"]
    D -->|"dispatched with that agent's scopeTarget carrier"| F["filter admits untagged listeners +<br/>listeners tagged with the key or an ancestor"]
    D -->|"events about a registry itself (a tool was added)"| U["registry-subject — stays unfiltered"]
    F --> V["scope invariant: a scope-filtered event without a carrier<br/>(or with a wrong carrier key) fails the invariant"]
```

## Producer → consumer highlights

```mermaid
flowchart LR
    P1["agent-loop"] -->|"produces"| E1["turn/start, step/start, assistant/*, agent/*"]
    E1 -->|"consumed by"| C1["persistence, projection, telemetry, UI, checkpoint-policy, compaction, hooks, plan-mode"]
    P2["tool-bash/fs/web..."] -->|"produce"| E2["tool/call, tool/result"]
    E2 -->|"consumed by"| C2["log, token-meter, titles, session-stats, telemetry"]
    P3["llm providers"] -->|"produce"| E3["llm/stream"]
    E3 -->|"consumed by"| C3["agent-loop, llm-replay, checkpoint-policy, session-title"]
```

Next: [09-llm-layer.md](09-llm-layer.md) — the model adapter seam.
