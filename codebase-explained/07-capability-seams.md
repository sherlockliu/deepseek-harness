# 07 — Capability Seams: the Three-Role Pattern

A **seam** is a swappable capability with three roles. One role alone is not a seam; adding a capability means designing all three. `packages/shell` is the canonical example.

## The pattern

```mermaid
flowchart TD
    subgraph DEF["Service Definition package (dsh-shell)"]
        D1["ctx.shell — ShellExecutor (abstract class,<br/>registered via super(ctx, 'shell'))"]
        D2["vocabulary types: ShellExecRequest / Spec / Result / Process"]
    end
    subgraph PROV["Service Providers"]
        P1["dsh-bash-local — LocalBashExecutor"]
        P2["dsh-bash-sandbox — SandboxBashExecutor<br/>(extends the local one, confines argv)"]
        P3["dsh-pwsh-local / dsh-pwsh-sandbox"]
    end
    subgraph CONS["Consumers"]
        C1["dsh-tool-bash — the bash tool (injects 'shell')"]
        C2["dsh-tool-bash-persistent"]
    end
    PROV -->|"provide"| DEF
    CONS -->|"inject + call"| DEF
```

Roles normally occupy separate packages when they evolve independently, but a package may own several roles when they are one concern (`dsh-llm` owns its Service Definition and Consumer).

## The resolve/spec split: explicit defaults at package boundaries

Defaults are an explicit `resolve(request): Spec` step in the owning implementation — never a hidden `?? default` inside `run()`:

```mermaid
flowchart LR
    T["tool: build ShellExecRequest<br/>(command, cwd?, timeoutMs?, policy?)"]
    T --> R["executor.resolve(request) → ShellExecSpec<br/>implementation-owned defaults: cwd, clamped timeout, stdout caps, env layering"]
    R --> U["executor.run(spec) → ShellRunResult<br/>rejects only for infrastructure failures"]
```

## The shell seam, end-to-end (foreground bash)

```mermaid
sequenceDiagram
    participant M as Model
    participant TL as tool-bash
    participant SE as ShellExecutor (provider)
    participant SUB as ctx.subprocess
    participant OS as OS process tree
    participant SBX as ctx.sandbox (if sandboxed)

    M->>TL: tool/call bash {command}
    TL->>TL: validate args + escalation pairing
    TL->>SE: resolve policy: sandboxPolicy.resolve({session})
    alt escalation (wider mode)
        TL->>TL: approveEscalation BEFORE anything runs (strict-widening check)
    end
    TL->>SE: shell.resolve(request) → spec
    SE->>SE: LocalBashExecutor.resolve: cwd, clampTimeout, stdout caps
    alt sandboxed provider
        SE->>SBX: confine(['bash','-c',cmd], policy) → wrapped argv (exact argv crosses the seam)
        SBX-->>SE: {argv: [bwrap/landlock/seatbelt/ACL runner..., '--', ...]}
    end
    SE->>SUB: subprocess.spawn(spec: stdio collect + spill, graceMs, signal)
    SUB->>OS: Node spawn, detached tree, bounded OutputCollector
    OS-->>SUB: exit
    SUB-->>SE: handle.done (timedOut vs aborted first-cause)
    SE-->>TL: ShellRunResult {exitCode, stdout/stderr, sandbox?}
    TL-->>M: tool/result — terminal card "[exit code: N]"
```

### The sandbox seam

`SandboxProvider.confine(argv, policy)` takes the **exact argv about to be spawned** and returns `{argv, enforcement, denialSignatures, runnerFailureRules}` — fail-closed, never a silent unconfined pass (`packages/sandbox/sandbox/src/index.ts:175`):

```mermaid
flowchart TD
    IN["confine(['bash','-c',command], policy)"]
    IN --> CHAIN["platform chain selection:<br/>linux: bwrap → landlock<br/>darwin: seatbelt<br/>win32: windows-acl"]
    CHAIN --> B1["bwrap: ro-bind /, tmpfs /tmp, re-bind workspace rw"]
    CHAIN --> B2["landlock: native launcher, allow-list grants"]
    CHAIN --> B3["seatbelt: sandbox-exec deny file-write* + allow-listed subpaths"]
    CHAIN --> B4["ACL: runner with per-workspace write SID + per-session temp SID"]
    B1 & B2 & B3 & B4 --> OUT["denial text classified vs denialSignatures;<br/>runner fatal-lines vs runnerFailureRules;<br/>unusable runner → SANDBOX_UNAVAILABLE"]
```

`dsh-bash-sandbox` stays mechanically identical to the local executor — the only additions are policy resolution, denial classification, and `result.sandbox` stamping. The tool advertises `sandbox_permissions`/`justification` args only when `ctx.shell.sandboxMode` is defined.

## The seam inventory

| Seam | Service Definition (ctx key) | Providers | Consumers |
|---|---|---|---|
| Bash/PowerShell | `ctx.shell` — ShellExecutor | bash-local, bash-sandbox, pwsh-* | tool-bash, tool-pwsh |
| Subprocess | `ctx.subprocess` — SubprocessRuntime | local, e2b | bash executors, terminal, LSP, subagents |
| Filesystem | `ctx.fs` — FileSystem | local, sandbox, e2b | tool-fs, str-replace-editor, LSP, skills |
| Sandbox | `ctx.sandbox` — SandboxProvider | local (bwrap/landlock/seatbelt/ACL) | bash-sandbox |
| Sandbox policy | `ctx.sandboxPolicy` | single impl | bash-sandbox, fs-sandbox, terminal, tools |
| LSP | `ctx.lsp` — Lsp registry | stdio (multi-server over fs+subprocess) | tool-lsp |
| Web | `ctx.web` — WebRuntime registry | search-exa/perplexity/deepseek, fetch-http | tool-web |
| Skill | `ctx.skills` — SkillRegistry | filesystem, badge | tool-skill |
| Subagent | `ctx.subagents` — SubagentRuntime (named-provider registry) | spawn/fork in-process, acp, codex, claude-code, dsh-sdk | tool-subagent(-control/-report), workflow |
| Workflow | `ctx.workflowEngine` | worker-thread | tool-workflow, tool-ralph |
| Terminal | `ctx.terminals` — TerminalSessionService | terminal-bash (over subprocess) | tool-terminal |
| Jobs | `ctx.jobs` — JobRegistry | jobs-local | tool-jobs, bash background |
| Compaction | `ctx.compaction` | compaction-basic | /compact command |
| Approval | `ctx.approval` | single impl | tool-bash/fs escalation, ask-user |
| Goal | `ctx.goals` | single impl | tool-goal, command-goal, goal-round-driver |
| E2B (POC) | `ctx.e2b` | single impl | fs-e2b, subprocess-e2b |
| LLM | `ctx.llm` — LlmRuntime | deepseek, pi-ai | agent-loop |

## The point of seams: provider swaps

```mermaid
flowchart LR
    subgraph ONE["Composition 1 — local"]
        S1["subprocess-local + fs-local + bash-local"]
    end
    subgraph TWO["Composition 2 — sandboxed"]
        S2["subprocess-local + fs-sandbox + bash-sandbox + sandbox-local"]
    end
    subgraph THREE["Composition 3 — remote (POC)"]
        S3["subprocess-e2b + fs-e2b + e2b runtime"]
    end
    ONE -->|"same tool-bash, tool-fs, LSP consumers"| X["One swap moves Bash, PTY, and LSP together<br/>— no provider forks, no loop changes"]
    TWO --> X
    THREE --> X
```

### FS policy events (no service, event-only policy)

```mermaid
sequenceDiagram
    participant TL as tool-fs write/edit
    participant POL as fs-observation-policy
    participant FS as FileSystem provider

    TL->>POL: ctx.waterfall('fs/write-intent', target, actor, next)
    Note over POL: unseen → createIfAbsent<br/>present → replaceIfVersion(version) — occupies the single decision slot
    POL-->>TL: FsWriteIntent
    TL->>FS: fs.writeText(target, content, intent) — provider enforces the CAS check
    FS-->>TL: outcome (stale → FS_STALE_VERSION → model-facing remedy)
    TL->>POL: ctx.emit('fs/observed', target, {kind:'present', version}, actor)
    Note over POL: edit-intent throws FS_NOT_OBSERVED unless the tool read the file first
```

Dropping the policy plugin degrades gracefully: the tools' default `() => undefined` yields the bare provider's unconditional atomic write.

### Subagent providers: a named-provider registry

Unlike `ctx.shell` (second load throws), `ctx.subagents` keeps **multiple providers coexisting**; callers pick by name (`packages/subagent/subagent/src/index.ts:171`). Providers advertise capabilities (`outputSchema`, `depthLimit`, `toolFilter`, `persona`); the runtime rejects a request needing a capability the provider lacks **before** the provider starts.

```mermaid
flowchart TD
    R["tool-subagent request → SubagentRuntime"]
    R -->|"pre-flight capability check"| P1["spawn-in-process: fresh child agent, inherits nothing"]
    R --> P2["fork-in-process: seeded with parent's completed-turn prefix<br/>(events up to last turn/end)"]
    R --> P3["acp / codex / claude-code: real out-of-process children via ctx.subprocess"]
    R --> P4["dsh-sdk: SDK client spawns the child runtime itself"]
    P2 -->|"prepareContinuable"| CONT["continuable children never become a SubagentRun —<br/>the manager drives the child AgentHandle's inbox directly"]
```

Next: [08-event-system.md](08-event-system.md) — the full event taxonomy.
