# `<arcgis-map>` Readiness Flow

## Purpose

Let web-component consumers pass the `<arcgis-map>` element directly as
`operational.view` and have `FuseSimple.create()` await the element's
internal readiness itself. This exists because the "wait for the event"
pattern (`addEventListener("arcgisViewReadyChange", handler)`) has a real
race: on cache-warm loads, Esri's SDK module registers the custom element
and its `connectedCallback` kicks off async view init before the
consumer's module reaches the `addEventListener` line. The event fires to
zero listeners and never fires again; the handler — which is the only
place `FuseSimple.create()` would have been called — is silently dead.

The element-passing path is race-free because it uses `viewOnReady()`, a
Stencil-style promise that captures *state*, not a *moment*. If the
component is already ready when we await it, the promise resolves
immediately. If not, it resolves when ready. Either way the race window
vanishes.

## Entry Points

- `src/core/FuseSimple.ts` — `isArcgisMapElement(v)` — duck-type guard:
  `v instanceof HTMLElement && typeof v.viewOnReady === "function"`.
  `HTMLElement` distinguishes element from MapView (a plain class);
  `viewOnReady` distinguishes `<arcgis-map>` from arbitrary DOM elements.
- `src/core/FuseSimple.ts` — `awaitArcgisMapReady(mapEl, logger)` —
  races `mapEl.viewOnReady()` against a 15 s timeout and an early-fail
  listener on `arcgisViewReadyError`. Cleans up listener + timer in a
  `finally` block.
- `src/core/FuseSimple.ts` — `FuseSimple.create()` — runs the detection
  as Step -1 (before the `view` cast), and, if matched, swaps
  `operational.view` to `mapEl.view` after readiness resolves. The
  remainder of `create()` proceeds unchanged because it only ever sees
  a resolved `MapView`.

## Sequence

1. Consumer writes `FuseSimple.create({ operational: { view: mapEl } })`
   where `mapEl` is the result of `document.querySelector("arcgis-map")`.
2. `create()` spreads `options.operational` into a local `operational`
   (never mutates the caller's object — could be frozen or reused).
3. Logger is constructed. First `LIFECYCLE` line logs.
4. `isArcgisMapElement(operational.view)` runs. If false (a MapView was
   passed), the whole Step -1 block is skipped and the classic path runs
   unchanged.
5. If true, log `LIFECYCLE: resolving <arcgis-map> element via
   viewOnReady()`.
6. `awaitArcgisMapReady(mapEl, logger)` is called. Internally:
   - A `Promise.race` is built between three promises:
     - `mapEl.viewOnReady()` — the happy path.
     - An `arcgisViewReadyError` listener — early-fail with Esri's real
       error.
     - A 15 s timeout.
   - Whichever settles first wins.
7. On success: the `finally` block clears the timeout and removes the
   error listener. `create()` reassigns `operational = { ...operational,
   view: mapEl.view }` so downstream code sees a resolved MapView. Logs
   `LIFECYCLE: <arcgis-map> element resolved to MapView`.
8. On failure: the `finally` block still cleans up. The rejection
   propagates out of `create()` unchanged — consumer sees a diagnostic
   `Error`.
9. `create()` proceeds with Step 0 (layer stashing) onward as if a
   MapView had been passed directly.

## State Transitions

```
  operational.view = HTMLElement           (input, web-component path)
         │
         │  isArcgisMapElement() → true
         ▼
  await awaitArcgisMapReady(mapEl):
    race(viewOnReady, viewReadyError, timeout)
         │
         ├─ viewOnReady resolves ─────────► operational.view = mapEl.view
         │                                         (now a MapView)
         ├─ viewReadyError fires ─────────► reject with Esri's message
         └─ 15 s elapses ─────────────────► reject with timeout message
```

## Failure Modes

| Failure | Outcome | Consumer sees |
|---|---|---|
| Missing `basemap` attribute on `<arcgis-map>` | Element never fires `arcgisViewReadyChange`, never becomes ready | 15 s wait, then `"<arcgis-map> did not become ready within 15000ms"` with a hint about attributes |
| Malformed `item-id` (portal item not found) | Esri emits `arcgisViewReadyError` with a 404/403 | Immediate rejection: `"<arcgis-map> failed to become ready — <Esri error>"` |
| Blocked portal request (CORS, offline) | Same as above | Immediate rejection with the network error surfaced |
| Consumer passes a plain `<div>` by mistake | `isArcgisMapElement` returns false (no `viewOnReady` method) | Classic path runs; `view.spatialReference?.wkid` is undefined; SR check passes (optional), but later duck-type calls into the view throw `TypeError` — not a great error but explicit enough to notice |
| Consumer passes an already-constructed MapView (classic path) | `isArcgisMapElement` returns false | Step -1 skipped, no change in behavior from previous versions |

## Debug Tags

| Tag | What it shows |
|---|---|
| `LIFECYCLE` | `"resolving <arcgis-map> element via viewOnReady()"`, `"<arcgis-map> element resolved to MapView"` |
| `BUG` | `bugId: "arcgis-view-ready-error"` when Esri emits `arcgisViewReadyError`; `bugId: "arcgis-map-ready-timeout"` when the 15 s safety net fires |

## Prior Art

- **Esri 5.0 `viewOnReady()`** — the vendor-documented Stencil hook.
  Source: `docs/arcgis_js_v50_sdk/javascript/v5-0/storybook/map-components/chunks/map.js:131-133`.
  Chains `componentOnReady()` (Stencil hydration) and `view.whenReady()`
  (Esri MapView ready) in order. Both are resolve-once promises that
  resolve immediately if their state is already true — hence race-free.
- **Stencil `componentOnReady()`** — standard pattern across Ionic,
  Calcite Components, and other Stencil-based custom-element libraries.
  Recommended by Esri's Calcite docs for any code that touches component
  properties before hydration completes.

## Open Questions

- **Should the timeout be configurable?** Currently hardcoded to 15 s
  inside `awaitArcgisMapReady`. If a consumer has a slow portal or a
  large item, they might legitimately need longer. Add a `readyTimeout`
  option on `OperationalOptions` if this comes up.
- **Should we surface the pending error-listener as a public event?**
  Today we log `arcgis-view-ready-error` to `BUG` and then reject.
  Consumer apps might want to instrument this themselves. Not adding
  until someone asks.
