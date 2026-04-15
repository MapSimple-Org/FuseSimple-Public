# Loading Splash Flow

## Purpose

Mask the MapLibre canvas during `FuseSimple.create()` so the consumer never
sees an unsynced or partially-tiled basemap. MapLibre's default viewport
(world view) and its tile-load sequence produce a visible flash between
`new Map(...)` and the first post-sync render. The splash is a neutral
`<div>` that covers the basemap container from before mount until MapLibre
reports `idle`, then fades out and removes itself. A timeout safeguard
dismisses the splash if `idle` never fires so a silent MapLibre failure
can't leave the map permanently masked.

## Entry Points

- `src/core/FuseSimple.ts` — `FuseSimple.create()` — creates the splash
  right after `getBasemapContainer()` returns (between Step 5a and the
  `await basemapAdapter.mount(...)` of Step 5).
- `src/core/splash.ts` — `createSplash(container, options, logger)` —
  builds the DOM, starts the timeout timer, returns a `{ dismiss, destroy }`
  handle.
- `src/adapters/maplibre.ts` — `MapLibreBasemapAdapter.onceIdle(cb)` — wraps
  `map.once('idle', ...)`. Called by `FuseSimple.create()` at Step 7c.
- `FuseSimple.destroy()` — invokes `splash.destroy()` via the teardown
  stack, covering the case where the consumer tears down mid-init.

## Sequence

1. `FuseSimple.create()` reaches Step 5 and calls
   `operationalAdapter.getBasemapContainer()`.
2. `createSplash(container, options.splash, logger)` is called:
   - Defaults merged with consumer-supplied overrides.
   - If `enabled === false`, returns a noop handle; nothing mounts.
   - Otherwise creates an absolutely-positioned `<div>` with
     `inset: 0`, `z-index: 1`, `pointer-events: none`, the configured
     `background`, and a centered `<div>` label containing `text` at
     `textOpacity`. Appends to the basemap container.
   - Starts `window.setTimeout` for the safeguard (default 10s).
3. `basemapAdapter.mount(container)` runs. MapLibre constructs behind the
   splash; all of its tile-load, style-apply, and initial-render work
   happens hidden from the user.
4. Steps 6 and 7 of the create flow run (attribution, sync subscription,
   initial sync push). The initial sync push triggers MapLibre to jump
   to the consumer's center/zoom and re-render.
5. Step 7c: `basemapAdapter.onceIdle(() => splash.dismiss())` registers a
   one-shot listener on MapLibre's `idle` event. Both the unsubscribe and
   `splash.destroy` go on the `instance.teardowns` stack.
6. MapLibre finishes rendering at the correct viewport, emits `idle`.
7. `splash.dismiss()`:
   - Sets `dismissed = true`, clears the timeout.
   - If `fadeDuration <= 0`, removes the element immediately.
   - Otherwise sets `opacity: 0`, registers a one-shot `transitionend`
     listener that removes the element, plus a `fadeDuration + 50ms`
     fallback timer in case `transitionend` never fires (tab backgrounded,
     style rewrite, etc.).
8. Element is removed from the DOM. `teardowns` still holds
   `splash.destroy`, which is a no-op at this point since the element
   is already gone.

### Alternate path: timeout fires before idle

1. 10 seconds pass without `idle`.
2. Timeout handler logs a `BUG`-level entry (bugId `splash-timeout`) and
   calls `dismiss()`.
3. Same fade sequence as normal dismissal.

### Alternate path: `destroy()` mid-init

1. Consumer calls `fuse.destroy()` before `idle` has fired.
2. Teardown stack runs. The `unsubIdle` teardown cancels the pending
   MapLibre listener. The `splash.destroy` teardown:
   - Sets `dismissed = true`, clears the timeout.
   - Removes the element immediately (no fade).
3. No DOM leakage, no dangling timers, no orphaned MapLibre listeners.

## State Transitions

- `splash` local in `create()`: written once at Step 2, read by the
  teardown callbacks.
- DOM: the splash `<div>` is appended to the basemap container at Step 2,
  removed at Step 7 (or earlier on timeout / destroy).
- `window.setTimeout` handles: one (the safeguard) created at Step 2,
  cleared by `dismiss()` or `destroy()`.
- MapLibre: no state mutation — `onceIdle` is a listener subscription,
  not a state change.

## Failure Modes

- **Silent timeout.** If `idle` never fires within `timeout` ms, the
  splash removes itself and logs a `BUG` entry with the configured
  timeout value. The consumer sees the map appear (possibly still
  loading tiles).
- **Splash called before mount.** `BasemapAdapter.onceIdle` is a no-op
  if the map hasn't mounted yet — logs a `LIFECYCLE` line and returns a
  noop unsubscribe. Not reachable in current code (the subscription
  happens after `await mount()`).
- **`transitionend` never fires.** Fade timer falls back to
  `setTimeout(removeNow, fadeDuration + 50)`, guaranteeing removal.
- **Splash disabled.** `createSplash` returns a noop handle; `onceIdle`
  still fires (and dismisses a splash that was never mounted), teardown
  still runs — no observable effect either way.

## Debug Tags

| Tag | Covers | When it fires |
|---|---|---|
| `LIFECYCLE` | splash mount, dismiss, destroy, onceIdle subscription, `basemap idle` | Once each during a normal `create()` + `destroy()` cycle |
| `BUG` | timeout safeguard tripped | Only if MapLibre doesn't reach `idle` within the configured timeout |

Suggested activation: `?debug=LIFECYCLE`

## Prior Art References

- Spec: `docs/FuseSimple-LoadingCurtain-Spec.md` — original requirements
  (the component was renamed "splash" from "curtain" during
  implementation).
- Flow reference: `docs/flows/basemap-lifecycle.md` — the full `create()`
  sequence this splash slots into.

## Open Questions

- Q: Should a future `logoUrl` option replace the text, or render
  alongside it (text as an accessible fallback)? Deferred until the
  MapSimple logo lands.
