# 01 — The Big Picture

DeepSeek Harness (dsh) is a **plugin-based agent harness**: a runtime that composes an LLM agent from swappable pieces — model adapters, tools, filesystem, sandbox, persistence, UI — where every piece is a plugin on a vendored copy of the [Cordis](https://github.com/cordisjs/cordis) framework.

## Why "everything is a plugin"

There is **no privileged core**. The agent loop itself is a plugin that can be replaced from configuration. All behavior attaches to documented extension points (events and services); removing a plugin unwinds its contributions.

```mermaid
flowchart TD
    CTX["Cordis Context<br/>one shared runtime object"]
    P1["plugin: dsh-llm<br/>registers ctx.llm"]
    P2["plugin: dsh-session<br/>registers ctx.sessions + the event log"]
    P3["plugin: dsh-agent-loop<br/>the driver (also replaceable)"]
    P4["plugin: dsh-tool-bash<br/>a model-facing tool"]
    P5["plugin: dsh-sandbox<br/>confines subprocesses"]
    P6["plugin: dsh-session-persistence-jsonl<br/>writes the log to disk"]
    CTX --> P1 & P2 & P3 & P4 & P5 & P6
    P4 -->|"injects ctx.shell"| P1
    P3 -->|"injects ctx.llm, ctx.sessions, ctx.tools"| P1
    P3 --> P2
    P5 -->|"wraps spawns from ctx.subprocess"| P2
    P6 -->|"listens session/event"| P2
```

Implications that shape the whole codebase:

1. **Registrations are effects.** Every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns a disposer. Unload a plugin and its services, events, and tools disappear cleanly.
2. **Composition happens at boot** from YAML (`cordis.yml`/`cordis.patch.yml`), not in code — see [03-composition-and-boot.md](03-composition-and-boot.md).
3. **Model-visible ⟺ logged.** Anything the model can see is reconstructable from the session log, and a runtime invariant asserts it — see [05-session-log.md](05-session-log.md).

## The layered map

```mermaid
flowchart TD
    subgraph L1["Layer 1 — Surfaces (apps/)"]
        CLI["dsh CLI (apps/cli)<br/>--profile web | headless"]
        WEB["Web app (apps/web + packages/client)"]
        ACP["ACP server (packages/acp)<br/>automation stdio protocol"]
        SDK["SDK clients (packages/sdk, python/)<br/>JSON-RPC over stdio"]
    end
    subgraph L2["Layer 2 — Composition (boot + bundle)"]
        BOOT["app-boot: profile/bundle/patch loader"]
        B1["dsh-base: models, tools, fs, sandbox, persistence"]
        B2["dsh-web-app: server + browser app"]
        B3["dsh-headless: one-shot runner"]
    end
    subgraph L3["Layer 3 — The plugin spine (core/)"]
        LOOP["agent-loop: turn/step driver"]
        SES["session: durable event log"]
        SP["system-prompt + tools registry"]
        AG["agent: inbox, registry, lifecycle"]
    end
    subgraph L4["Layer 4 — Capability seams"]
        SH["shell / subprocess / fs / sandbox"]
        WB["web search / lsp / skill / workflow"]
        SA["subagent / terminal / jobs / compaction"]
    end
    subgraph L5["Layer 5 — Providers"]
        LOC["local providers"]
        SBX["sandboxed providers (bwrap / Landlock / Seatbelt / ACL)"]
        E2B["e2b remote sandbox (POC)"]
    end
    subgraph L6["Layer 6 — Models"]
        DS["DeepSeek adapter"]
        PI["pi-ai adapter"]
        RT["llm-retry policy"]
    end
    CLI --> BOOT
    WEB --> BOOT
    ACP --> BOOT
    SDK --> BOOT
    BOOT --> B1 & B2 & B3
    B1 & B2 & B3 --> LOOP
    LOOP --> SES & SP & AG
    LOOP --> SH & WB & SA
    SH --> LOC & SBX & E2B
    LOOP --> DS & PI & RT
```

The vertical arrow shows the key property: **the loop only talks to seams**, never to concrete providers. Swapping the provider behind `ctx.shell` or `ctx.fs` changes how bash, PTY, and LSP behave everywhere, with no changes to the loop.

## The three core invariants that everything else follows

```mermaid
flowchart LR
    subgraph I1["1 — Registrations are effects"]
        A1["ctx.effect(body)"]
        A2["body runs now, disposers unwind on plugin unload"]
        A1 --> A2
    end
    subgraph I2["2 — Model-visible ⟺ logged"]
        B1["prompt section, tool schema, message"]
        B2["...must be reconstructable from session/event"]
        B1 --> B2
    end
    subgraph I3["3 — A seam has three roles"]
        C1["Service Definition<br/>(ctx key + abstract class)"]
        C2["Service Provider<br/>(implements it)"]
        C3["Consumer<br/>(tool that injects it)"]
        C1 --> C2 --> C3
    end
```

## Where the code lives

| Concern | Location |
|---|---|
| The plugin runtime | `vendor/` (Cordis, rescopped to `@deepseek-ai/cordis`) |
| Product spine | `packages/core/` |
| Swappable capabilities | `packages/<seam>/` one group per seam |
| Durable session data | `packages/session/` |
| Remote/typed RPC | `packages/api/`, `packages/typert/`, `packages/sdk/` |
| Browser app | `packages/host/` + `packages/client/` |
| Shipped composition | `packages/bundle/`, `packages/examples/`, `examples/` |
| Python SDK | `python/` (SDK + bundled runtime, JSON-RPC stdio) |
| Native launcher | `native/landlock-run/` (Landlock self-restrict-then-exec) |

Next: [02-cordis-foundation.md](02-cordis-foundation.md) explains the framework mechanics everything rides on.
