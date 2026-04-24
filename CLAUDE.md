# Claude Code Instructions for FuseSimple

## Project Overview

FuseSimple is a JavaScript library (TypeScript source) that fuses two mapping engines into a single seamless map: a hidden MapLibre GL JS + PMTiles basemap underneath a transparent ArcGIS Maps SDK for JavaScript MapView on top. The developer never touches MapLibre directly. One import, one `FuseSimple.create()` call, working map.

FuseSimple is part of the broader **MapSimple** family (sibling to the QuerySimple / HelperSimple / FeedSimple Experience Builder widgets). It shares MapSimple's conventions on tone, git workflow, debugging discipline, and language precision, but the stack is completely different — this is a standalone library, not an ExB widget.

**Source of truth:** `docs/FuseSimple_v1_Spec.md`. If implementation diverges from the spec, flag it and ask before proceeding.

## Critical Rules

1. **No Unauthorized Changes:** NEVER make changes the user did not explicitly ask for. Do not proceed with any code modification, fix, or refactor without explicit user approval. This is the #1 rule.
2. **Ask First When Unsure:** Do not take liberties. If you aren't sure about a logic change, an architectural decision, the right file to touch, or whether a request implies a broader change, ask before acting. A 30-second clarification beats a 30-minute revert. This rule is the safety net under #1 — when in doubt, stop and ask.
3. **The Scalpel Rule:** Be precise in everything — code, debugging, language, documentation. Surgical edits, not sweeping changes. Evidence, not assumptions. Say exactly what you mean.
4. **Git Commits:** NEVER commit without explicit user approval. NEVER include `Co-Authored-By` or any co-author attribution in commit messages. All commits are authored solely by the developer.
5. **Branches:** NEVER develop on `main` or `develop`. All work happens on `feature/*` branches.
6. **Debugging Discipline:** Never guess at root causes. Add instrumentation, build, test, read the logs, then fix. A fix must trace back to evidence. If you only have a hypothesis, say so — don't present reasoning as proof. When in doubt, add more instrumentation, not more code changes.
7. **Log-First Rule:** Investigating a bug? Add `debugLogger.log(...)` calls to capture state and timing **before** guessing at a fix. The logger is URL-activated, so you can ship instrumentation without changing default behavior. Read the logs, then fix.
8. **Precision in Language:** Distinguish data from inference. Use "the logs show" for evidence, "I believe" for interpretation. Never say "this fixes the bug" until tested — say "this should address the cause identified in the logs." State exactly what a change does and why, not what you hope it does.
9. **Document Failed Attempts:** When investigating a bug or design problem, document what has been tried and why it didn't work before moving on to the next approach. This prevents going down the same path twice across sessions. Use a dedicated `docs/<topic>-investigation.md` file for multi-attempt investigations. Each entry should include: the approach, the result, and the status (reverted, current, untried).

## Audit Practice

These rules exist to prevent code drift during updates, refactors, and upgrades.

### Pattern Preservation
- This project uses established patterns for logging, state management, sync logic, and adapter structure. Do not refactor, rename, restructure, or "improve" existing patterns unless explicitly asked.
- If you believe a pattern should change, flag it and wait for approval. Do not act on the belief.
- When updating code, match the existing style. New code should look like it was written by the same developer who wrote the rest of the project.

### Scope Containment
- Only modify files directly related to the current task.
- Do not "improve" adjacent code, clean up unrelated formatting, or optimize things you were not asked to touch.
- If you notice something that should be fixed but is outside the current task, note it. Do not fix it.

### Diff Awareness
- After any change, summarize what was modified and why.
- Flag any changes that touch code outside the scope of the current task.
- If a change alters behavior (not just syntax or formatting), call it out explicitly.

### Dependency Discipline
- Do not add new packages or dependencies without explicit approval.
- If a task can be accomplished with existing dependencies or native code, use those.
- If you believe a new dependency is warranted, state why and what alternatives you considered.
- Watch the bundle budget (250 KB gzipped). Every new dependency costs.

### Test Gate
- Do not consider a task complete until all tests pass.
- If a test fails, determine whether the test is wrong or the code is wrong. Do not silently "fix" a test to make it pass without understanding why it failed.
- Never generate tests that validate what the code does. Tests must validate what the code should do. If the AI writes both the code and the tests, the tests become a mirror, not a validator.

### Baseline Comparison
- Before starting a refactor, the current passing test suite is the behavioral baseline.
- Any test that passed before the change and fails after the change is a regression until proven otherwise.
- Do not assume a broken test means the test was outdated. Investigate first.

## Git Workflow

```
feature/*  <-- All development happens here
    |
 develop   <-- Integration/testing (merge feature here when done)
    |
  main     <-- Production stable (merge develop here when stable)
```

**Before ANY code changes:**
1. Check current branch: `git branch --show-current`
2. If on `main` or `develop`, STOP and ask the user which feature branch to use
3. Switch to feature branch: `git checkout feature/branch-name`
4. Then proceed with development

**Exception:** Documentation-only changes (`*.md` files) MAY be done on `develop`, but prefer feature branches.

**The Save Game Milestone:** Always commit and push a stable, tested baseline before starting high-risk refactors (e.g., adapter rewrites, sync logic changes, build system changes).

## Tone & Style

- Do not use excessive dashes in prose or comments.
- Avoid "AI-isms" or overly formal "robotic" language.
- Maintain a direct, peer-to-peer tone with a sense of levity.

(The "do not take liberties / ask first" rule lives in Critical Rules #2 — it's about how to proceed, not tone.)

## Technical Standards

### Stack
- **Language:** TypeScript, compiled to ES modules + type declarations for distribution
- **Build tool:** Vite (library mode for the lib, dev server for the demo)
- **Runtime deps (internal, hidden from consumer):**
  - `maplibre-gl` 5.x
  - `pmtiles` 4.x
- **Peer dep:** `@arcgis/core` 5.0+ (consumer brings their own)
- **Target browsers:** Chrome, Firefox, Edge, Safari (current). WebGL required.

### Bundle Budget
Acceptance criterion #8: no more than ~250 KB gzipped added to the consuming app. Watch this every time a dependency is added or imports are widened.

### Versioning
- **Pre-1.0 until release.** FuseSimple stays in `0.x` until there's a well-tested version that meets all spec acceptance criteria. `1.0.0` is reserved for that release. No exceptions, no early `1.0.0` for marketing reasons.
- **Bump the minor version on each meaningful change.** `0.1.0 → 0.2.0 → 0.3.0 → ...` Edit `version` in `package.json` as part of the same change. "Meaningful" means real code, behavior, or API changes. Typo fixes, comment edits, and pure docs touches don't bump.
- **Patch (`0.x.Y`) is reserved for tiny in-between fixes** that don't warrant a minor bump (e.g., a one-line bug fix between two larger changes). Most edits should bump minor.
- Pre-1.0 semver doesn't promise API stability anyway, so we don't follow strict major/minor/patch semantics during this phase. The MapSimple ExB widgets use a similar "bump on every change" pattern via a `MINOR_VERSION` constant; FuseSimple uses `package.json` `version` for the same purpose since it's a published npm library.
- Once we ship `1.0.0`, normal semver applies: breaking changes bump major, additive features bump minor, fixes bump patch.

### Logging
- **No `console.log` in shipped library code.** Ever. Use the FuseSimple `debugLogger`.
- The logger is a port of the MapSimple ExB `debugLogger` (originally from `shared-code/mapsimple-common/debug-logger.ts`), customized for a standalone library context. It lives at `src/debug-logger.ts` and is **exported** from `src/index.ts` so consumer apps can register their own tags and use the same `?debug=` URL flag.
- Constructor option `name` (renamed from `widgetName` since FuseSimple isn't a widget). Constructor option `features: string[]` lists registered tags. Constructor option `isProduction?: boolean` (default `false`) — when `true`, the URL param is ignored entirely and **all** logging is suppressed, including `BUG` level. Use this in production builds where end-users could otherwise hit `?debug=all` and inspect internals.
- `FuseSimple.create({ ..., isProduction: true })` plumbs the flag down to FuseSimple's internal logger. Consumer apps that build their own `DebugLogger` instance set `isProduction` on their own constructor.
- Activation (when `isProduction` is `false`):
  - `?debug=all` — enable every registered tag
  - `?debug=SYNC,PASSTHROUGH` — enable specific tags (comma-separated, case-insensitive)
  - `?debug=false` — explicitly disable all logging
  - (no param) — disabled, zero overhead
- The logger is iframe-aware: it checks `window.parent` if `?debug=` isn't on the current window URL. This matters because consumers may embed FuseSimple inside an ExB widget or other iframed host.

#### FuseSimple Debug Tags (v1)

| Tag | Covers |
|---|---|
| `LIFECYCLE` | `create()`, mount, destroy, teardown ordering, listener cleanup |
| `HEADER` | PMTiles archive header parse, bbox/center/min/max zoom, computing Esri `TileInfo.create().lods` |
| `STYLE` | Basemap style resolution (bundled Protomaps Light vs URL vs inline object), load, apply to MapLibre |
| `SYNC` | Esri `view.watch` → MapLibre `setCenter`/`setZoom`/`setBearing`, including `bearing = -rotation` |
| `PASSTHROUGH` | hitTest filter, `queryRenderedFeatures` call, coordinate translation, hover throttling |
| `EVENTS` | Public event emitter — `basemap-click` / `basemap-hover` `on/off/emit` activity |
| `ATTRIBUTION` | OSM attribution injection into the Esri Attribution widget |
| `BUG` | Always-on (until `isProduction: true`). For known issues with `bugId` / `category` / `description` fields |

Add a new tag only when an existing one would answer two genuinely different debugging questions. Don't fragment for granularity's sake — that's how QuerySimple ended up with 30+ tags.

#### `BUG` Convention

Use `debugLogger.log("BUG", { bugId, category, description, ... })` for known issues that should always surface (until production lockdown). `BUG` writes via `console.warn` with a distinctive prefix so it stands out in mixed console output. It's the lightweight in-code bug tracker.

### URL Parameters

FuseSimple owns **no** URL state of its own beyond `?debug=` (provided by the debug logger). The library does not read, write, or consume any other query string or hash fragment. URL routing is the consumer's concern.

### Testing
- "When I say test, I mean internal and manual testing by me." Pause for manual testing at significant milestones rather than assuming automated tests cover it. The King County demo is the primary manual test surface.
- Unit tests are welcome for deterministic logic (header parsing, rotation/bearing math, URL handling). Don't write speculative test scaffolding before there's logic to test.

#### Intended Test Shape (when we get there)

Mirroring the MapSimple ExB dual-layer pattern, scaled down for a small library:

- **Vitest unit tests** for deterministic logic: PMTiles header parsing, `rotation → bearing` conversion, LOD computation from header bounds, debug logger tag registration / `isProduction` lockdown behavior.
- **Playwright smoke test** of the King County demo: page loads, basemap visible, Esri layer visible, click on a road fires `basemap-click`, OSM attribution present in the attribution widget.

This is the **intended** shape, not a v1 prerequisite. Don't scaffold test infra speculatively. When we add the first deterministic-logic file that warrants a unit test, we add Vitest then.

### Code Discipline
- **Zero Duplication:** if a helper exists, use it. Don't recreate utilities.
- **Adapter encapsulation:** consumers must never import or see MapLibre or PMTiles types. Anything cross-engine lives behind the adapter interface in `src/adapters/types.ts`.
- **No speculative abstractions:** the spec defines v1. Don't build for v2 features (3D, multi-PMTiles, Leaflet adapter) until they're scheduled.

## Project Structure

```
FuseSimple/
  CLAUDE.md                      -- this file
  README.md                      -- public-facing
  LICENSE                        -- MIT
  LICENSES.md                    -- third-party attributions (MapLibre, PMTiles)
  package.json
  tsconfig.json
  vite.config.ts                 -- library build config
  docs/
    FuseSimple_v1_Spec.md               -- source of truth (WHAT)
    FuseSimple_v1_Decisions.md          -- implementation decisions (HOW)
    FuseSimple_v1_Prior_Art_Reference.md -- patterns from maplibre-gl-leaflet, sync-move, etc.
  src/
    index.ts                     -- public entry, exports FuseSimple + DebugLogger
    debug-logger.ts              -- URL-activated debug logger (ported from MapSimple ExB)
    core/
      FuseSimple.ts              -- main class, create() factory
      sync.ts                    -- view state sync (Esri -> MapLibre)
    adapters/
      types.ts                   -- adapter interface
      maplibre.ts                -- MapLibre basemap adapter
      esri.ts                    -- Esri operational adapter
    styles/
      protomaps-light.json       -- bundled default basemap style (TBD)
  demo/
    index.html
    main.ts
    README.md                    -- how to run, where to get the King County PMTiles
```

## Architecture Reminders (from the spec)

- **Esri is the interaction driver.** All user input goes through the Esri MapView. FuseSimple listens to Esri view state and pushes it down to MapLibre.
- **Sync is one-way (Esri -> MapLibre)** for v1: center, zoom (1:1, no offset, both Web Mercator 256px), rotation (`bearing = -rotation`).
- **Extent comes from the PMTiles header** (bbox, center, default zoom). No dummy layers.
- **Click/hover passthrough:** if Esri click misses a feature, query MapLibre `queryRenderedFeatures()` and surface as `basemap-click` / `basemap-hover` events.
- **Adapter pattern:** the sync core is engine-agnostic. `basemap` and `operational` each declare an `engine`. v1 ships `maplibre` (basemap) and `esri` (operational).
- **OSM attribution** must be injected into the Esri attribution widget automatically.

## Out of Scope for v1

3D SceneView, pitch/tilt, terrain, multiple PMTiles sources, runtime style editing, Leaflet/OpenLayers adapters. See spec section 7.
