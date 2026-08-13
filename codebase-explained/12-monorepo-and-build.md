# 12 — Monorepo Layout and the Build Pipeline

## Workspace layout

pnpm workspaces: `vendor/*`, `packages/*/*`, `native/landlock-run` (+ its packages), `apps/*`, `website`. Every npm package is `@deepseek-ai/dsh-<name>`; vendored packages are rescoped and `private: true`; `@deepseek-ai/cordis` is a peerDependency (+ dev) of every harness package.

```mermaid
flowchart TD
    subgraph WS["pnpm workspace"]
        V["vendor/* — 9 pinned Cordis-family packages"]
        P["packages/<group>/<pkg> — ~150 packages,<br/>groups: core api typert llm e2b shell subprocess terminal fs lsp skill web<br/>compaction context subagent bundle workflow todo plan preset guard self-modification<br/>hooks session identity settings credentials acp interaction boot sdk examples support util"]
        A["apps/cli (dsh bin) + apps/web (vite entry)"]
        N["native/landlock-run workspace (entry + platform packages)"]
        W["website (VitePress)"]
        PY["python/ (separate Python packaging, driven by scripts/)"]
    end
```

## ESM and faces

- ESM everywhere (`"type": "module"`); cross-package imports use package names; local relative imports use `.ts`.
- **Source plane vs artifact plane, never mixed**: static gates and tests resolve workspace imports through tsconfig `paths` to `src`; gates consuming built `lib/` declare that dependency.
- Two aggregate programs — `tsconfig.host.json` (Node) and `tsconfig.client.json` (browser) — keep compiler faces explicit; each package uses one aggregate except `api/remotes`.

## Build pipeline

```mermaid
flowchart TD
    A["tsc -b tsconfig.host.json"] --> B["tsdown --env.DSH_BUILD_FACE host"]
    B -->|"typertPlugin: generates typert.host/typert.client/typert.remote-client artifacts in writeBundle"| G["per-package lib/ with /typert, /client, /remote exports"]
    A2["tsc -b tsconfig.client.json"] --> B2["tsdown --env.DSH_BUILD_FACE client"]
    G --> A2
    B --> B2
```

`dsh` source-launch runs through tsx's ESM-only hook (`node --import tsx/esm`), so modules it reaches must stay ESM.

## Verification layers

| Layer | Command | What it proves |
|---|---|---|
| Unit | `pnpm run test` (vitest) | package behavior, focused tests |
| Coverage gate | `pnpm run test:coverage` | per-file 100% on `packages/*/*/src` (the CI gate) |
| Real API | `pnpm run test:e2e` | provider behavior (self-skips without `DEEPSEEK_API_KEY`) |
| Keyless snapshots | `pnpm run test:snapshot` | assembled-app transcripts vs recorded outputs |
| Web | `pnpm run test:web[:perf][:stress]` | built browser app replay |
| Typecheck | `pnpm run typecheck` | both faces compile |
| Lint | `pnpm run lint` | oxlint after contracts build |
| Duplication | `pnpm run duplication` | jscpd clone detection |
| Hygiene | `pnpm run hygiene` | knip + publint + workspace constraints + NodeNext consumer check |
| Docs | `pnpm run doc-sync` | doc budgets, md-links, type-equiv, mermaid, cordis configs... |
| Windows | `pnpm run check:windows-wine` | only when diagnosing a known Windows failure |

`scripts/run-gates.ts` is the gate runner; `check:all` / `check:ci` orchestrate. The repo's own standards live in the root [AGENTS.md](../AGENTS.md) and the skills under `.agents/skills/`.

## The verification philosophy in one picture

```mermaid
flowchart LR
    subgraph SRC["What changes"]
        S1["plugin code"]
        S2["cordis.yml composition"]
        S3["docs / prose"]
    end
    subgraph GATE["Which check proves it"]
        G1["focused unit tests"]
        G2["keyless snapshot via a runnable example"]
        G3["doc-sync gates"]
    end
    SRC --> GATE
    Note1["Match evidence to the surface; CI owns exhaustive coverage + platform matrix"]
```

## python/ and native/

```mermaid
flowchart LR
    subgraph PY["python/"]
        P1["sdk — high-level turns API + JSON-RPC client"]
        P2["sdk-runtime — bundled runtime binaries + default agent config"]
    end
    subgraph NAT["native/landlock-run"]
        N1["entry package + platform packages (npm optionalDependencies)"]
        N2["self-restrict-then-exec launcher for Linux Landlock"]
    end
    P1 -->|"stdio JSON-RPC"| P2
    NAT -->|"consumed by dsh-sandbox-local"| SB["sandbox confine"]
```

Next: [13-end-to-end.md](13-end-to-end.md) — one request through everything.
