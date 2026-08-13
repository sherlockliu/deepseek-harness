# 03 — Composition and Boot: Profiles, Bundles, Patches

A running `dsh` is a **plugin tree composed at boot from ordered layers**. No code writes the tree; YAML does.

## Profiles and bundles

```mermaid
flowchart TD
    subgraph P["Profile — named composition in $DSH_HOME/profiles/<name>/"]
        PKG["package.json<br/>dsh.profile.bundles = [dsh-base, dsh-web-app]"]
        PATCH["cordis.patch.yml — user's own rows/overrides"]
        ROOT["cordis.yml — an EMPTY entry list, the include anchor"]
    end
    subgraph B1["Bundle @deepseek-ai/dsh-base"]
        B1P["package.json<br/>dsh.bundle.patch = ./cordis.patch.yml"]
        B1R["cordis.patch.yml — ~100 rows:<br/>llm, session, tools, agent-loop, bash, fs, sandbox, skills, subagents..."]
    end
    subgraph B2["Bundle @deepseek-ai/dsh-web-app"]
        B2R["cordis.patch.yml — server + browser client roster,<br/>agent-presets, disables base tool rows (moves behind presets)"]
    end
    P -->|"stacks, in listed order"| B1
    P --> B2
```

A **bundle** is a distribution format for Cordis config rows plus the code they mount. Whatever a bundle inserts stays **patchable** by the layers above it.

Two shipped templates exist (`apps/cli/src/profile-boot.ts`, `packages/boot/app-boot/src/profile.ts:114`):

| Template | Bundle stack | Result |
|---|---|---|
| `web` | `[dsh-base, dsh-web-app]` | Full browser app over HTTP/WebSocket |
| `headless` | `[dsh-base, dsh-headless]` | One-shot task runner, no server |

## Patch layering order

Layers apply to the empty entry list, lowest first (`apps/cli/src/profile-boot.ts:122-171`):

```mermaid
flowchart TD
    L1["1 — bundle rows<br/>each bundle's cordis.patch.yml, in dsh.profile.bundles order"]
    L2["2 — profile patch<br/><profile>/cordis.patch.yml"]
    L3["3 — home patch<br/>$DSH_HOME/cordis.patch.yml (machine-local, all profiles)"]
    L4["4 — --patch overlays<br/>in argv order"]
    L5["5 — launcher-generated overlays<br/>shipped agent-preset root, telemetry opt-out"]
    L1 --> L2 --> L3 --> L4 --> L5
```

Semantics of `applyEntryPatches` (`vendor/include/src/index.ts:58-128`):

```mermaid
flowchart TD
    subgraph INS["insert patches"]
        I1["{ insert: 'dsh-tool-bash', id: 'tool-bash' }<br/>appends a row (into a group if named)"]
        I2["inserted rows are indexed immediately —<br/>a LATER layer can patch a row an EARLIER layer inserted"]
        I1 --> I2
    end
    subgraph TGT["id-targeted patches"]
        T1["{ id: 'system-prompt', config: {...} }<br/>REPLACES the row's whole config — not a deep merge"]
        T2["{ id: 'hmr', disabled: true } disables it"]
        T3["unknown id or name mismatch: warn + SKIP<br/>(one shared overlay may apply to many surfaces)"]
        T1 --> T2 --> T3
    end
```

This is why `dsh-web-app` "disables" base tool rows (`disabled: true`) instead of deleting them — the base layer keeps them patchable, and per-session **agent presets** take over that role.

## The boot sequence

Product CLI path — `apps/cli/src/bin.ts` → `apps/cli/src/profile-boot.ts` → `packages/boot/app-boot/src/index.ts`:

```mermaid
sequenceDiagram
    participant U as User
    participant CLI as apps/cli (parseDshArgs)
    participant PB as profile-boot (runProfile)
    participant BOOT as app-boot (boot)
    participant LOADER as vendored Loader
    participant INCL as cordis:include
    participant PLUG as Plugins
    U->>CLI: dsh --profile web "task"
    CLI->>PB: loadLayeredEnv → process.env snapshot (rejects bootstrap-only names)
    PB->>PB: composeProfile(name, patchFiles):<br/>flatten all 5 layers into one patch list
    PB->>BOOT: boot(NAME, configPath, patches, prepare)
    BOOT->>BOOT: ctx = new Context(); ctx.baseUrl = dir of cordis.yml
    BOOT->>BOOT: ctx.provide('dshHomePath', ...); await ctx.plugin(Loader)
    BOOT->>PB: prepare(ctx) — launch env + cmdline args
    BOOT->>LOADER: mountRootInclude: register cordis:include + cordis:group builtins
    LOADER->>INCL: create pinned entry { id: 'include', config: { path, patches } }
    INCL->>INCL: read YAML → applyEntryPatches(rows, patches) → root.update(rows)
    LOADER->>PLUG: per row: import module, interpolate !!js, start fiber
    Note over LOADER,PLUG: activation is inject-driven, not row-order-driven
    BOOT->>BOOT: await ctx.get('loader').await() — settle whole tree
    BOOT->>BOOT: assertEntriesActivated — fail loud with per-row reasons
    BOOT-->>PB: running context
    PB->>PB: watchUserPatches — HMR on profile/home patch files
    PB->>PLUG: mount cordis-plugin-timer + hmr when no hmr row exists
```

Failure anywhere disposes the partial tree (`ctx.fiber.dispose()`), and `installFailLoud` turns a late unhandled rejection into one labelled stderr line + exit 1, restoring the terminal first (`packages/boot/app-boot/src/index.ts:609`).

## Live user edits: HMR

```mermaid
sequenceDiagram
    participant F as User edits cordis.patch.yml
    participant W as watchUserPatches
    participant I as Include entry
    participant E as Entry tree
    F->>W: file change
    W->>W: composeLive: re-stack bundle layers below + overlays above
    W->>I: entry.update({ path, patches: recomposed })
    I->>E: applyEntryPatches again
    E->>E: per row diff: in-place patch / restart / dispose — rollback on failure
```

## Inspect what your machine boots

```sh
dsh --profile web --dump-config
```

`renderConfigDump` (`packages/boot/app-boot/src/index.ts:379`) runs the **same** `applyEntryPatches` call the boot uses, with per-layer provenance comments — so dumps, flags, and boots cannot drift. Any printed row can be replaced by a patch of your own.

## Two boot shapes

```mermaid
flowchart LR
    subgraph A["Product CLI"]
        A1["profiles/bundles/patches<br/>--profile web|headless, --patch"]
        A2["HMR, fail-loud, layered env"]
    end
    subgraph B["Demo bins (packages/examples/*-demo)"]
        B1["boot(NAME, ./cordis.yml)<br/>direct leaf composition"]
        B2["e.g. dsh-acp-demo, dsh-sdk-jsonrpc-demo, agent-spine-demo"]
    end
```

Both call the same `boot()`. The demos skip profile machinery entirely.

Next: [04-agent-loop.md](04-agent-loop.md) — what the composed tree actually *does*.
