# Adapter Contracts Flow

> **Status:** current as of 0.13.0. Covers the `BasemapAdapter` and
> `OperationalAdapter` interfaces in `src/adapters/types.ts`, their two v1
> implementations (`MapLibreBasemapAdapter`, `EsriOperationalAdapter`), and
> how `FuseSimple.create()` orchestrates both.

## Purpose

FuseSimple fuses two engines (MapLibre basemap, Esri operational view) into
a single map. The code that does the fusing — the sync loop, the click and
hover plumbing, the LOD math — is engine-agnostic. Swapping either engine
must not require touching `src/core/`. That requirement is what the two
adapter interfaces exist to enforce.

The split is intentional and asymmetric. The **basemap** adapter owns an
engine instance FuseSimple creates and controls end-to-end (MapLibre map,
PMTiles archive, tile style). The **operational** adapter wraps an engine
instance the *consumer* already created (their `MapView`) and exposes
exactly the hooks the core needs: view state, click/hover passthrough,
attribution, and a DOM anchor for the basemap. Neither adapter knows the
other exists.

If you're adding a second basemap engine (hypothetical Leaflet + raster,
spec §7 v2), a second operational engine (hypothetical Leaflet operational
view, spec §11), or debugging a leak/race across the engine boundary, this
is the doc to start with.

## Entry Points

- `src/adapters/types.ts:86` — `BasemapAdapter` interface. The contract a
  basemap engine must satisfy for the sync core to drive it.
- `src/adapters/types.ts:168` — `OperationalAdapter` interface. The
  contract an operational engine must satisfy to feed view state into the
  sync core and route passthrough events back out.
- `src/adapters/types.ts:14-84` — structural cross-engine types
  (`LngLat`, `ScreenPoint`, `GeoScreenPoint`, `ViewState`,
  `PMTilesHeader`, `BasemapFeature`). Everything that crosses an adapter
  boundary is one of these shapes.
- `src/adapters/maplibre.ts:125` — `MapLibreBasemapAdapter` —
  `BasemapAdapter` implementation over `maplibre-gl` + `pmtiles`.
- `src/adapters/esri.ts:133` — `EsriOperationalAdapter` —
  `OperationalAdapter` implementation over a duck-typed
  `@arcgis/core/views/MapView`.
- `src/core/FuseSimple.ts:218` — `FuseSimple.create()` — the only place
  both adapters are held together and wired into the sync loop.

## The Interfaces

### `BasemapAdapter` (types.ts:86)

The basemap side is the *driven* end of the sync loop. It receives view
state, never produces it. But because FuseSimple creates and owns the
basemap engine, this interface is wider than the operational one — it
covers the whole lifecycle (mount, resize, destroy) plus a header read
that the core needs for LOD math.

| Method | File:line | Contract |
|---|---|---|
| `mount(container)` | `types.ts:94` | Async. Must be called exactly once per adapter instance. A second call is a programming error and should throw rather than silently re-mount. Resolves once the engine has loaded its style and registered any protocols it needs. Safe to be racing against `destroy()` — see "Destroy during mount" below. |
| `setViewState(state)` | `types.ts:101` | Sync. Called at most once per animation frame by the sync core (`FuseSimple_v1_Decisions.md` §"Sync Architecture — rAF Batching"). `ViewState.bearing` is already sign-flipped from Esri rotation — adapters never see raw Esri conventions. |
| `resize(width, height)` | `types.ts:110` | Sync. CSS pixels, already `Math.round`ed by the caller. Adapter must NOT round again (Decisions §"CSS translate rounding — blurry tiles fix"). |
| `queryFeaturesAt(point)` | `types.ts:118` | Sync. Returns an empty array (never `null` / `undefined`) so callers can `for (const f of …)` without guards. Results are the engine-agnostic `BasemapFeature` shape — MapLibre's `MapGeoJSONFeature` does not leak out. |
| `readHeader()` | `types.ts:126` | Async. Must be cached after first call (the core reads it multiple times during setup). Can be called before `mount()`. |
| `setLayerVisibility(id, visible)` | `types.ts:136` | Sync. No-ops silently if the layer ID doesn't exist. This mirrors `setLayoutProperty(id, 'visibility', …)` behind the adapter wall. |
| `destroy()` | `types.ts:143` | Sync. Tears down the engine, drops listeners. Safe to call even if `mount` hasn't resolved (the consumer may yank us mid-load). |
| `_getRenderedCenter?()` | `types.ts:149` | Optional, debug-only. Leading underscore + `?` signals "not part of the public consumer API" but the orchestrator may still call it via `?.`. |
| `_getRenderedZoom?()` | `types.ts:155` | Same. |
| `_getCanvasInfo?()` | `types.ts:160` | Same. |

### `OperationalAdapter` (types.ts:168)

The operational side is the *driver*. It emits view state and passthrough
events; it never applies them. FuseSimple doesn't create the operational
engine — the consumer did that already — so this interface is deliberately
thin. No `mount`, no `destroy` on the engine itself. The adapter only owns
the DOM bits and subscriptions it creates on behalf of FuseSimple.

| Method | File:line | Contract |
|---|---|---|
| `onViewStateChange(h)` | `types.ts:170` | Subscribe to center/zoom/rotation changes. Must return an unsubscribe fn. The handler receives a post-conversion `ViewState` (bearing, not rotation). |
| `onPassthroughClick(h)` | `types.ts:173` | Subscribe to clicks that did NOT hit an operational feature. The adapter runs the hit-test; the core only sees passthroughs. Returns an unsubscribe fn. |
| `onPassthroughHover(h)` | `types.ts:176` | Same shape as click. Adapter is responsible for throttling (rAF) so the core doesn't get flooded. |
| `addAttribution(text)` | `types.ts:183` | Idempotent. Safe to call multiple times with the same text — implementations should de-dupe rather than stack duplicates. |
| `getBasemapContainer()` | `types.ts:190` | Return a DOM element the basemap will mount into. Must be attached to the document and will shortly have non-zero size. This is the one DOM affordance the operational adapter exposes to the basemap adapter, and it's mediated through the core. |
| `destroy()` | `types.ts:198` | Tear down adapter-owned DOM, observers, and listeners. Safe to call multiple times and safe to call mid-mount. Mirrors `BasemapAdapter.destroy()` so the core can walk both adapters symmetrically during teardown. |

## State Ownership

Which adapter owns what. "Owns" means: creates, mutates, and is responsible
for tearing down. A table because this is the question new contributors ask
most often.

| Resource | Owner | Notes |
|---|---|---|
| Sibling `<div>` for the basemap | Operational (`esri.ts:467-478`) | Created lazily on first `getBasemapContainer()`, inserted as first child of `view.container`. |
| MapLibre `Map` instance | Basemap (`maplibre.ts:220`) | Constructed in `mount()`, `remove()`d in `destroy()`. |
| `PMTiles` archive | Basemap (`maplibre.ts:195`) | One per adapter. Shared `Protocol` is module-global (see below). |
| `maplibregl.addProtocol("pmtiles", …)` | Module (`maplibre.ts:58-65`) | Process-global singleton `sharedProtocol`. Not torn down in `destroy()` — other adapters may still be using it. |
| Esri view-state subscription | Operational (`esri.ts:214-216`) | Three `reactiveUtils.watch()` handles for center/zoom/rotation. Microtask-batched into one handler call per synchronous burst. |
| `view.background`, `view.constraints.lods`, `view.map.basemap` | Core (`FuseSimple.ts:297-329`) | Mutated by `FuseSimple.create()`, not by either adapter. These touch policy (spatial reference, transparency, LOD ceiling) that sits above a single engine. |
| `view.map.layers` stash/restore | Core (`FuseSimple.ts:253-260, 425-431`) | Prevents Esri's extent auto-fit from overriding the consumer's initial center/zoom. Not the adapter's problem. |
| Attribution DOM injection + `MutationObserver` | Operational (`esri.ts:393-436`) | Pinned to `__powered-by` via a sibling `<span>`. Observer stored on the adapter and disconnected in `destroy()`. |
| Click / hover `hitTest` + rAF throttle | Operational (`esri.ts:243-355`) | The basemap adapter never sees operational-layer features; the operational adapter never sees basemap features. Only the `GeoScreenPoint` crosses. |
| Event emitter (`basemap-click`, `basemap-hover`) | Core (`FuseSimple.ts:529-543`) | Adapters produce half-events; the core joins them and emits. |
| LOD computation | Core (`FuseSimple.ts:314`, `src/core/lod.ts`) | Pure function, not an adapter method. The basemap adapter supplies `PMTilesHeader`; the core does the math. |
| Zoom correction (Esri linear vs MapLibre exponential) | Core (`FuseSimple.ts:377-390`) | `zoomFromResolution(view.resolution)` is computed in the core and written into the `ViewState` before it crosses into the basemap adapter. Adapters never see raw Esri zoom values for fractional levels. |

The pattern: policy lives in the core, engine mechanics live in the
adapters. Anything that requires knowing about *both* engines at once is
wrong on an adapter and belongs in the core.

## Initialization Contract

Order matters. `FuseSimple.create()` calls adapter methods in a specific
sequence, and each method assumes earlier ones have run. A new adapter
must satisfy the same preconditions.

1. **Construct operational adapter first** (`FuseSimple.ts:283`). It
   duck-type-validates the consumer's view (`esri.ts:524-545`) and throws
   early with a clear message if the view doesn't quack. No DOM work yet.
2. **Construct basemap adapter** (`FuseSimple.ts:284-288`). Still no DOM
   work — just option stashing and a default logger if none was passed.
3. **Read header before mount** (`FuseSimple.ts:313`). The core calls
   `basemapAdapter.readHeader()` to get `bounds` / `center` / `minZoom` /
   `maxZoom` for LOD computation. The basemap adapter must support this
   pre-mount: `MapLibreBasemapAdapter.readHeader()` (`maplibre.ts:372-399`)
   lazily creates the PMTiles archive if `mount()` hasn't run yet.
4. **Apply LODs and sync `maxZoom` to the adapter** (`FuseSimple.ts:324-354`).
   The core mutates `view.constraints.lods` directly (not via the adapter)
   and calls `basemapAdapter.setMaxZoom()` so both engines share a zoom
   ceiling. `setMaxZoom` must be called before `mount` — the adapter reads
   the value during Map construction (`maplibre.ts:219`).
5. **Get basemap container from operational adapter** (`FuseSimple.ts:357`).
   `operationalAdapter.getBasemapContainer()` must return an attached,
   sized DOM element. The Esri implementation lazily creates and caches it.
6. **Mount basemap** (`FuseSimple.ts:358`). `await basemapAdapter.mount(container)`.
   The adapter is responsible for everything between "empty div" and "load
   event fired" — protocol registration, style resolution, `new Map(...)`,
   waiting for `"load"`.
7. **Inject attributions** (`FuseSimple.ts:363-364`). Two calls:
   consumer-configured OSM credit and the FuseSimple version badge. The
   adapter's idempotency contract means re-calling with the same text is
   a no-op.
8. **Subscribe to view state** (`FuseSimple.ts:377`).
   `operationalAdapter.onViewStateChange(...)` returns an unsubscribe fn
   that the core pushes onto its `teardowns` array.
9. **Initial sync push** (`FuseSimple.ts:400-419`). The core reads the
   current Esri view state and calls `basemapAdapter.setViewState(...)`
   once so MapLibre starts at the right place. Watchers only fire on
   *changes*.
10. **Subscribe to passthrough events** (`FuseSimple.ts:456-485`). Click
    and hover each return unsubscribe fns, both pushed onto `teardowns`.

A concrete invariant: any adapter method the core calls between step 1 and
step 6 must work without `mount` having completed. That's `readHeader()`
and `setMaxZoom()` on the basemap side, and `getBasemapContainer()` +
`addAttribution()` on the operational side. The rest assume mount is done.

## Encapsulation Rules

The adapter layer is the firewall between consumer code and engine SDKs.
Two rules, enforced by code organization rather than tooling:

**Rule 1:** `src/adapters/maplibre.ts` is the *only* file in the tree that
imports from `maplibre-gl` or `pmtiles`. Grep proves it. Adding a second
basemap adapter means a second file with those imports and nothing else.

**Rule 2:** `src/adapters/esri.ts` does not import from `@arcgis/core` at
all — not even as `import type`. Every Esri type it uses is expressed as a
local `Duck…` interface (`esri.ts:28-81`) and the view is validated at
runtime by `narrowView()` (`esri.ts:524-545`). This is stricter than rule 1
because `@arcgis/core` is a peer dep — it's not guaranteed to be resolvable
from FuseSimple's own `node_modules` at build time, and requiring it would
force every consumer to satisfy TypeScript's module resolution for the
full SDK.

Supporting rules:

- `src/adapters/types.ts` is forbidden from importing from any engine
  package (`types.ts:8-11`). Every cross-engine primitive here is
  structural.
- `BasemapFeature` (`types.ts:71-84`) is a minimal normalized shape.
  MapLibre's `MapGeoJSONFeature` is narrowed down at the boundary in
  `queryFeaturesAt` (`maplibre.ts:343-364`) so consumer code never sees it.
- `ViewState.bearing` is always MapLibre-convention (CW-positive, opposite
  of Esri). The sign flip lives at exactly one site in
  `onViewStateChange` (`esri.ts:196`) so the basemap adapter never has to
  know Esri's rotation convention exists.
- `reactiveUtils` is resolved dynamically in the core
  (`FuseSimple.ts:636-652`) and handed to the operational adapter as
  `DuckReactiveUtils`. The adapter never `import`s it, static or dynamic.

If a new adapter violates either rule, "go to definition" from consumer
code will reach into a private engine type, the bundle budget will balloon,
or both. That's the concrete observable failure; the rule itself is just
the shortest path to preventing it.

## Destroy Symmetry and Teardown Ordering

Teardown reverses setup. `FuseSimple#destroy()` (`FuseSimple.ts:603-624`)
runs three phases:

1. **Unsubscribe in reverse registration order** (`FuseSimple.ts:610-613`).
   The `teardowns` array was appended in setup order; the loop walks it
   backward. In practice: passthrough hover, passthrough click, resize
   relay, sync watcher. Reverse order matters because earlier subscriptions
   may have created resources later ones depend on — e.g. the resize relay
   implicitly relies on the sync loop to have applied the initial view
   state before the first resize.
2. **Destroy the basemap adapter** (`FuseSimple.ts:616`). Removes the
   MapLibre map, drops the archive reference, nulls the container pointer.
   Does NOT call `maplibregl.removeProtocol("pmtiles")` — the protocol is
   process-global and another adapter may still be using it
   (`maplibre.ts:54-57, 437-446`).
3. **Destroy the operational adapter** (`FuseSimple.ts:617`). Disconnects
   attribution observers, removes the sibling div. Does NOT tear down the
   consumer's `MapView` — that's not ours.

The destroy is idempotent at the FuseSimple level (`destroyed` flag on
line 604). Both concrete adapters are also idempotent: the basemap adapter
sets `isDestroyed = true` and guards state on subsequent calls; the
operational adapter just no-ops if `basemapContainer` is already null.

### Destroy during mount

The interesting case is `destroy()` called while `mount()` is still in
flight. `MapLibreBasemapAdapter` handles this with an `isDestroyed` flag
checked at every `await` point:

- **Before the Map is constructed** (`maplibre.ts:207-213`): bail out
  without creating a Map. Nothing to tear down.
- **Between `new maplibregl.Map(...)` and the `"load"` event**
  (`maplibre.ts:250-256`): `destroy()` already called `map.remove()`, so
  the `mount()` Promise resolves as a no-op once `"load"` arrives (or
  doesn't). Callers never see an unhandled rejection.

The `AbortSignal` alternative was considered and deferred: the flag is
simpler and the header fetch is fast enough in practice that an abort path
hasn't paid for itself. See `docs/flows/basemap-lifecycle.md` §"Destroy
during mount" for the longer history.

## Debug Tags

Which adapter logs under which tag. Same scheme as the rest of FuseSimple
(see `CLAUDE.md` §"FuseSimple Debug Tags (v1)").

| Tag | Emitted by | Examples |
|---|---|---|
| `LIFECYCLE` | Both adapters + core | Mount start/complete, destroy start/complete, listener start/stop, basemap container create/remove, layer stash/restore. |
| `HEADER` | Basemap (`maplibre.ts:391-397`) | PMTiles header parsed. LOD-out-of-bounds warnings fire from the core, not the adapter. |
| `STYLE` | Basemap (`maplibre.ts:201-205, 410-421`) | Style shape (URL vs object vs bundled), layer visibility changes. |
| `SYNC` | Operational (`esri.ts:199-204`) + core (`FuseSimple.ts:382-387`) | Batched view state emit from the operational adapter; zoom correction from resolution in the core. |
| `PASSTHROUGH` | Operational (`esri.ts:258-261, 321-325`) | Click and hover passthroughs. |
| `EVENTS` | Core (`FuseSimple.ts:465-469`) | `basemap-click` / `basemap-hover` emission to user listeners. Adapters never emit this directly. |
| `ATTRIBUTION` | Operational (`esri.ts:381-432`) | Widget-not-found, injection, re-injection after MutationObserver fires. |
| `BUG` | Core (`FuseSimple.ts:535-540`) | Always-on (until `isProduction: true`). Currently one entry point: uncaught errors in user event handlers. |

Suggested activation when debugging an adapter-boundary issue:
`?debug=LIFECYCLE,SYNC,PASSTHROUGH`. That's every log line that fires
because state crossed from one adapter to the other.

## Failure Modes

Adapter-level failures, separate from the broader lifecycle failures
catalogued in `docs/flows/basemap-lifecycle.md`.

- **Throws** at `EsriOperationalAdapter` construction (`esri.ts:524-545`)
  if the passed `view` isn't a MapView-shaped object. Message lists the
  missing member.
- **Throws** at `MapLibreBasemapAdapter.mount()` (`maplibre.ts:171-176`)
  on double-mount or post-destroy mount. Programming errors, not
  recoverable.
- **Throws** at `EsriOperationalAdapter.getBasemapContainer()`
  (`esri.ts:454-460`) if `view.container` is null. This means the consumer
  constructed a MapView without a container, which should have failed
  earlier — we surface it with a clearer message.
- **Silently no-ops** on `BasemapAdapter.setLayerVisibility()` if the
  layer ID doesn't exist (`maplibre.ts:407-421`). Logged under `STYLE`
  but not thrown — the contract calls this out explicitly in
  `types.ts:132-136`.
- **Silently no-ops** on `OperationalAdapter.addAttribution()` if the
  Esri attribution widget isn't in the DOM (`esri.ts:380-384`). The
  MutationObserver will re-fire once the widget renders, but this path
  is not currently retried.
- **Swallows** `hitTest` rejections during click/hover passthrough
  (`esri.ts:264-267, 327-329`). If the view is destroyed mid-event, the
  rejection is ignored rather than surfaced — there's nobody left to
  receive the event.
- **Error in user event handler** — the core emits via a try/catch
  (`FuseSimple.ts:532-541`) and logs under `BUG`. A badly-behaved consumer
  listener won't take out the whole event pipeline.

## Extending the Model

### Adding a second basemap adapter

Scenario: someone wants a raster-tile basemap via a different engine (not
scheduled for v1 — spec §7 lists Leaflet and OpenLayers adapters as "v2+").
Checklist:

1. New file `src/adapters/<engine>.ts` implementing `BasemapAdapter`.
   Only file in the tree that imports the engine SDK.
2. `readHeader()` returns a `PMTilesHeader`-shaped object. If the engine
   doesn't read PMTiles, synthesize the shape from whatever metadata it
   has (the structural type was chosen deliberately — see `types.ts:55-63`).
3. `setViewState()` accepts MapLibre-convention bearing. If the engine
   uses a different convention, flip it inside the adapter — the sync
   core and the other adapter never learn about it.
4. `queryFeaturesAt()` returns the engine-agnostic `BasemapFeature` shape,
   not the engine's native feature type.
5. `destroy()` is safe to call mid-mount. Use the `isDestroyed` flag
   pattern from `maplibre.ts:142` unless the engine has a cleaner cancel
   story.
6. Engine-global state (e.g. `maplibregl.addProtocol` equivalent) lives at
   module scope and is NOT torn down in `destroy()` — other instances may
   still need it.
7. Extend `BasemapOptions.engine` (`FuseSimple.ts:39`) to a union and
   branch in `FuseSimple.create()` on which adapter to construct. The
   current single-engine union is a simplification, not a fundamental
   constraint.

### Adding a second operational adapter

Harder, because the operational adapter owns the DOM container for the
basemap and the core mutates engine-specific properties
(`view.background`, `view.constraints.lods`, `view.map.basemap`) directly.
Checklist:

1. New file `src/adapters/<engine>-operational.ts` implementing
   `OperationalAdapter`. Only file that imports the SDK.
2. `getBasemapContainer()` must return an element whose size tracks the
   operational view. The Esri implementation relies on a sibling div plus
   a `reactiveUtils.watch(view.size)` relay in the core — a new adapter
   would need an equivalent, probably via `ResizeObserver`.
3. `onViewStateChange()` must produce `ViewState` with MapLibre-convention
   bearing, regardless of the engine's native rotation convention.
4. Click and hover passthrough need a hit-test primitive. If the engine
   has nothing equivalent, synthesize it by letting the event reach the
   basemap and deciding there — but this inverts the current "operational
   is the driver" model and would warrant a design doc.
5. `addAttribution()` must de-dupe — the core calls it twice (OSM credit
   + version badge).
6. The core currently reaches into `view.background`,
   `view.constraints.lods`, `view.map.basemap`, and `view.map.layers`
   directly in `FuseSimple.create()` (`FuseSimple.ts:253-329`). These
   would need to be lifted onto the `OperationalAdapter` interface
   (`setBackground`, `setLODs`, `stripBasemap`, `stashLayers` /
   `restoreLayers`) before a non-Esri operational adapter could work.
   That's the biggest structural change in this checklist.

### Explicitly out of scope for v1

Per `docs/FuseSimple_v1_Spec.md` §7 and §11:

- 3D SceneView support (v2) — the `ViewState` struct has no pitch or
  tilt field. Adding one is an interface change.
- Multiple PMTiles sources (v2) — the basemap adapter accepts exactly one
  `source` string.
- Runtime style editing (v2) — `setLayerVisibility()` is the only
  runtime-style mutation exposed.
- Leaflet / OpenLayers adapters (v2+) — see checklists above.
- Reverse-direction fusion (Esri as basemap, MapLibre as operational) —
  technically expressible in the adapter model, but the initial extent
  / LOD / transparency policy all assume the current direction. Call it
  a v2+ redesign.

## Prior Art References

- Spec §11 Open Design — Engine Adapters —
  `docs/FuseSimple_v1_Spec.md` lines 280-297
- Decision — PMTiles protocol registration — ordering —
  `docs/FuseSimple_v1_Decisions.md#pmtiles-protocol-registration--ordering`
- Decision §1 DOM Strategy — sibling-div pattern —
  `docs/FuseSimple_v1_Decisions.md#1-dom-strategy--dual-canvas-stack`
- Basemap lifecycle flow —
  `docs/flows/basemap-lifecycle.md`

## Open Questions

- Q: Does the `BasemapAdapter` need an explicit `isMounted()` predicate,
  or is the current "every method no-ops pre-mount" contract sufficient?
  Leaning no — adding it would invite `if (!adapter.isMounted()) return`
  guards in core code that currently works fine without them.
- Q: `BasemapFeature.properties` is typed `Record<string, unknown>`. A
  generic `BasemapFeature<P>` would let consumers narrow per-layer, but
  every call site would have to pick a type. Deferred until a consumer
  actually asks.
