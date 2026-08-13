# 10 — Web, API, and RPC: The Client/Server Split

There are **three** client/server surfaces, all on the same plugin spine:

1. **Web GUI** — browser ↔ Node host over HTTP + WebSocket (`packages/host/` + `packages/client/`).
2. **Typert Remotes** — a generated, typed RPC seam carried inside the web split's transport (`packages/typert/` + `packages/api/`).
3. **SDK / ACP** — JSON-RPC over stdio for external automation (`packages/sdk/`, `packages/acp/`, `python/`).

## The two compiler faces

```mermaid
flowchart TD
    subgraph HOST["Host face (tsconfig.host.json) — Node"]
        H1["core, llm, session, seams, boot, bundles"]
        H2["packages/host: webserver, apiproxy, frontend-static"]
        H3["packages/api/gateway + remotes host half"]
    end
    subgraph CLIENT["Client face (tsconfig.client.json) — browser"]
        C1["packages/client: connection, runtime, web-react, ui-*"]
        C2["packages/api/remotes client half (imports generated /remote values)"]
    end
    HOST -->|"generates /remote artifacts the client face compiles against"| CLIENT
```

Each package is registered in exactly one face; packages with browser halves expose a `/client` subpath. `packages/api/remotes` is the documented exception, splitting its own project into two faces.

## The web GUI split

```mermaid
flowchart TD
    subgraph HOST["Host (packages/host)"]
        WS["webserver — bare node:http, named routes<br/>(exact/prefix tables, one fallback seat, upgrade routes)"]
        AP["apiproxy — ApiProxyService: the transport-agnostic API gateway<br/>(fetch handler pair, registers no routes itself)"]
        FS["frontend-static — SPA dist, claims the fallback seat"]
        PI["plugin-inventory — Loader entries as Remote pluginInventory/list"]
    end
    subgraph CLI["Browser (packages/client)"]
        CON["connection — ConnectionController (reconnect/backoff),<br/>WebApiClient: HTTP fetch up + ONE WebSocket downlink per event stream"]
        RT["runtime — SessionManager, event windows, zustand stores"]
        WR["web-react — the only ctx↔React glue (slot renderer, SessionProvider)"]
        UI["ui-* — pure-props presentation components"]
    end
    CLI -->|"POST /api/<method> + /api/events.mux|host downlinks"| AP
    AP --> WS
```

The connection protocol is a four-quadrant envelope (`packages/host/apiproxy/src/api/rpc.ts`): `ClientRequest | ServerResponse | ServerRequest | ClientResponse` with branded `RpcId`s. HTTP status expresses only carrier failures; business errors ride as 200 + `result.ok:false`. A trust fence (`api-request-trust.ts`) rejects DNS-rebinding/cross-site requests; privileged methods pin to loopback. Server→browser pushes use `/api/events.mux` + `/api/events.host` WebSocket downlinks; `/api/respond` carries client responses to `ServerRequest`s (approvals, questions).

The legacy BFF exposes ~57 zod-validated methods (`packages/host/apiproxy/src/api/rpc-map.ts:24`): `session.*`, `subagent.*`, `host.*`, `workspace.*`, `skill.list`, `agentPreset.*`, `goal.*`, `settings.*`, `credentials.*`, `llm.*`.

## Typert: generated typed RPC

```mermaid
flowchart LR
    subgraph GEN["Build time — packages/typert/generator"]
        G1["seeds: tsconfig.host.json / tsconfig.client.json programs"]
        G2["discovery: public exports reachable from Context/Events augmentations<br/>+ @typert tags"]
        G3["model: FaceModel + TypeGraph (generics, JSDoc retained)"]
        G4["emit: per-package artifacts"]
        G1 --> G2 --> G3 --> G4
    end
    G4 --> ART1["lib/typert.host.{js,d.ts} → package/typert<br/>Zod schemas + TYPERT manifest (services/events/objects/invocations)"]
    G4 --> ART2["lib/typert.client.{js,d.ts} → package/client/typert"]
    G4 --> ART3["lib/typert.remote-client.{js,d.ts} → package/remote<br/>InvocationDescriptors + declaration merges into TypertRemoteMap"]
    ART3 -->|"runtime imports + ctx.remote.$mount"| REM["packages/api/remotes client face"]
    REM -->|"type-only imports for business packages"| BIZ["client business code: ctx.remote.<namespace>.<method>(...) — fully typed"]
```

The host tsdown phase runs the generator with `tsconfig.host.json` as the only seed and produces both the host reflection and the remote-client projection; the client tsdown never runs Typert.

### A Remote call, end-to-end

```mermaid
sequenceDiagram
    participant UI as Browser component
    participant CR as ctx.remote (ClientRemoteService)
    participant CON as Connection (HTTP /api)
    participant GW as TypertGatewayService (host)
    participant REG as ctx.typert (registry)
    participant SVC as Business service (e.g. MessageFeedbackService)

    UI->>CR: ctx.remote.messageFeedback.list({sessionId})
    CR->>CON: POST /api messageFeedback/list (named args)
    CON->>GW: claimed endpoint dispatch (<namespace>/<method>)
    GW->>REG: resolve strict InvocationDescriptor from ctx.typert.local
    GW->>SVC: invoke with validated args (agent/session lookups resolved)
    SVC-->>GW: result
    GW->>GW: codec-validate → ServerResponse
    GW-->>UI: typed result
```

Event fan-out in reverse: the host forwards allowlisted events (`API_REMOTE_FORWARDED_EVENTS`) as `host/remote-event` frames on the host WebSocket; the client runtime hands them to `ctx.remote.$dispatch`, and subscribers receive them with their owner package's own `Events` signatures.

## SDK JSON-RPC (stdio) — external automation

```mermaid
flowchart TD
    subgraph EXT["External process"]
        TS["TypeScript client: HarnessClient / DeepSeekHarness (packages/sdk/client)"]
        PY["Python client: deepseek_harness SDK (python/sdk)"]
    end
    subgraph SERVER["packages/sdk/server"]
        SRV["HarnessSdkJsonRpcServer (newline-delimited JSON-RPC 2.0)"]
        R1["initialize — cwd/provider/model/maxTokens"]
        R2["session/prompt — lazy agent+session, returns messageId"]
        R3["shutdown — disposes the WHOLE root runtime, exit 0"]
        N1["notifications: session.event, session.status,<br/>subagent.started, subagent.finished"]
    end
    EXT -->|"stdio"| SRV
    SRV --> R1 & R2 & R3 & N1
```

The high-level client API is `prompt → wait for whole-agent idle → collect events → finalResponse`. The Python SDK starts a matching bundled runtime (`python/sdk-runtime`).

## ACP — automation-only Agent Client Protocol

`packages/acp/acp` speaks the [Agent Client Protocol](https://agentclientprotocol.com) over JSON-RPC stdio (`packages/acp/acp/src/index.ts:105`) — an interoperability transport, not a UI:

```mermaid
sequenceDiagram
    participant C as ACP client (e.g. an IDE)
    participant S as dsh-acp server
    participant A as Agent

    C->>S: initialize → baseline prompts only
    C->>S: session/new → fresh agent (absolute cwd)
    C->>S: session/prompt (text blocks concatenated; one in-flight per session)
    S->>A: prompt; settles at whole-agent idle → end_turn
    A-->>S: committed assistant/message → one agent_message_chunk per text block
    S-->>C: result (stop-reason mapped in codec.ts)
    C->>S: session/request_permission → one-shot allow/reject
    Note over S: teardown: reject new work, settle pending,<br/>drain continuable descendants, dispose in parallel
```

No navigation, replay, modes, elicitation, or tool presentation — `subagent-acp` is the primary in-repo client.

## Adjacent services

- **session-query** (`ctx.sessionQuery`): combined session-history reads, traces, filters, FTS5 full-text search (SQLite provider) + a model-facing tool + ZIP log export.
- **attachment** (`ctx.attachments`): image validate/save/read with limits; local backend.
- **message-feedback**: the canonical `TypertRemoteService` example with `@Remote`-decorated methods.

Next: [11-session-plumbing.md](11-session-plumbing.md) — durability, projection, telemetry, compaction.
