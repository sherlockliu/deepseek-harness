# Agent Note: Diagram-first codebase explainer folder

Status: implemented

English | [中文](2026-08-13-codebase-explained-folder.zh.md)

## Problem

`docs/architecture.md` is a map for contributors who already know Cordis, and the documentation corpus gates budgets, tiers, one-home-per-fact, and bilingual pairing. A newcomer who wants a tutorial-style, diagram-heavy walkthrough of the whole codebase has no entry point that can hold that much detail without tripping those gates.

## Decision

`codebase-explained/` at the repository root holds fourteen diagram-first Markdown pages explaining the architecture and code: the plugin spine, composition and boot, the agent loop, the session log, scope and tools, capability seams, the event system, the LLM layer, web/API/RPC surfaces, session plumbing, the monorepo build, and one end-to-end request trace. Diagrams use Mermaid, validated with the same parser `scripts/verify-mermaid.ts` uses (all blocks parse); prose cites `file:line` references into source. The folder lives outside the documentation corpus: no gate scans it, no bilingual counterpart or budget applies — its index page is named `index.md`, not `README.md`, so the README-artifact bilingual rule does not capture it — and `docs/` remains the single authoritative home of facts. Pages link back into `docs/` for authority and state that source and `docs/` win over the explainer when they drift.

## Alternatives considered

**Put the pages under `docs/`.** The corpus would force tutorial/reference classification, word budgets, one-home-per-fact relocation, and bilingual pairing for every page; the explainer intentionally restates facts with diagrams, which the tier rules forbid. Rejected as ongoing maintenance cost for an onboarding aid.

**Put the pages in `.agents/notes/`.** Agent Notes are decision records with a fixed format and lifecycle, not teaching material. Rejected.

**Keep the explanation out of the repository.** The explainer was authored to be committed and reviewed; a PR lets maintainers correct it or delete it wholesale. Rejected.

## Consequences

- A new top-level folder that maintainers may keep honest or delete; because no gate covers it, its links and diagrams are only as correct as their last authoring and review.
- Onboarding is diagram-first: one folder reads the whole system without prior Cordis knowledge.
- Slight repository-root surface increase; the folder's index page states its non-authoritative stance.
