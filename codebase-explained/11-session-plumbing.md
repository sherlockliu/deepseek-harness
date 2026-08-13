# 11 — Session Plumbing: Persistence, Projection, Telemetry, Compaction

The packages in `packages/session/` (plus `compaction/`, `plan/`, `todo/`) give the log durability and derived value.

## Persistence backends

```mermaid
flowchart TD
    DEF["Service Definition: SessionPersistence<br/>create / append / load (commit recovery) / inspect / readFrom / list / readRaw"]
    DEF --> JSONL
    DEF --> SQLITE
    subgraph JSONL["dsh-session-persistence-jsonl"]
        J1["one append-only file per session:<br/>root/<project>/<encodedSessionId>/session.jsonl[.zstd]"]
        J2["header record first; assistant/chunk runs packed into<br/>text-chunks/reasoning-chunks/tool-call-chunks rows (~60% smaller)"]
        J3["materialize = temp write + fsync + link() publish (never rename)"]
        J4["appends fsync with rollback-to-previous-size on failure;<br/>checksummed Zstandard frames, torn-frame recovery"]
    end
    subgraph SQLITE["dsh-session-persistence-sqlite"]
        S1["sessions row (existence = materialization) + 1:1 events rows"]
        S2["appendBatch = one transaction; commitRepair = one transaction"]
        S3["loadStoredFrom: seq >= fromSeq suffix read (seek-capable)"]
        S4["SCHEMA_VERSION = 15 (monotonic, strict check)"]
    end
```

Both are behind one contract; the coordinator (`packages/session/session-persistence/src/coordinator.ts`) is the single write-path owner: per-session promise chains (writes never interleave), contiguous-seq enforcement, lazy creation, HMR seed-prefix adoption, and torn-tail repair with synthetic `turn/end {kind:'interrupted'}` closers.

## Projection cache: the cold-start ladder

```mermaid
flowchart TD
    START["session opened"]
    START --> C["cached (sessionId,key,ver,seq,val) row? — must match log identity (createdAt+cwd)"]
    C -->|"hit"| T["persistence readFrom(floor) tail → registry restore → fold"]
    C -->|"miss"| F["cold fold from seq 0"]
    T --> W["write-behind throttling; mandatory writes at turn/end + disposal;<br/>every write fail-soft; rows may be stale, never wrong"]
    F --> W
```

## Titles and telemetry

```mermaid
flowchart LR
    T1["session-title: sole ctx.sessionTitle provider slot"]
    T2["providers: first-prompt-llm, all-prompts-llm, llm"]
    TE1["session-telemetry: session-telemetry/record waterfall"]
    TE2["session-telemetry-otel: OTel LoggerProvider +<br/>BatchLogRecordProcessor + OTLPLogExporter<br/>modes FULL / FEEDBACK_ONLY / DISABLED"]
    TE1 --> TE2
```

## Compaction

`dsh-compaction-basic` triggers on `agent/pre-step` (context pressure) and on `agent/request-error` (canonical context overflow), optionally prunes tool results first, then summarizes. Recovery works between the closed failed step and the failed turn close; it opens a fresh retry turn only when pruning or summarization **advances the surface replacement generation** — otherwise the original request error stays authoritative (`docs/agent-lifecycle.md:76`).

```mermaid
flowchart LR
    TRIG["pressure detected"] --> PRUNE["tool-result pruning (optional)"] --> SUM["summary selection"]
    SUM --> REPLACE["append replace surface op over the shadowed range<br/>— shrinks history WITHOUT rewriting the log"]
    REPLACE --> RETRY["fresh turn; deriveMessages() now folds the shorter surface"]
```

## Planning, todos, and background work

```mermaid
flowchart TD
    PLAN["plan mode (dsh-plan-mode): logged state, not a seam<br/>controller + exit tool + /plan command"]
    TODO["tool-todo: todo/write events on the session log"]
    JOB["ctx.jobs: background bash/terminal sends;<br/>job_output / job_list / job_kill tools collect or stop them"]
    TERM["ctx.terminals: persistent PTY sessions;<br/>terminal_open/send/read/signal/close/list"]
    WF["ctx.workflowEngine: worker-thread provider;<br/>workflow + ralph tools compose fresh-agent runs"]
    SPILL["spill: oversized outputs move to files,<br/>with inline byte budget policy"]
    PLAN --> TODO
    TODO --> JOB
    JOB --> TERM
    TERM --> WF
    WF --> SPILL
```

Next: [12-monorepo-and-build.md](12-monorepo-and-build.md) — how the many packages compile and get verified.
