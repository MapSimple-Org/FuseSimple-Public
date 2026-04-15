# Changelog

All notable changes to FuseSimple are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
FuseSimple is pre-1.0: we do not promise API stability until `1.0.0` ships.
Minor version bumps on meaningful changes, patch reserved for small in-between fixes.

## [Unreleased]

## [0.14.1] — 2026-04-15

### Added
- **`operational.view` now accepts an `<arcgis-map>` element directly**,
  not just a constructed `MapView`. Web-component consumers can pass the
  element reference and FuseSimple awaits its `viewOnReady()` hook
  internally — no `addEventListener("arcgisViewReadyChange")` plumbing
  required. Closes a latent listener race that only surfaced in the
  web-component pattern: on cache-warm loads, the custom element could
  reach `ready` between Esri's SDK registering it and the consumer's
  module attaching a listener, so the event fired to zero listeners and
  `FuseSimple.create()` never ran. Symptom was a silent failure with the
  throwaway `basemap="gray-vector"` visible and nothing else.
- **15 s readiness timeout with a diagnostic error.** If an `<arcgis-map>`
  element never becomes ready (malformed attributes, missing `basemap`,
  blocked portal), `create()` rejects with a message pointing at the
  likely causes. Also listens for Esri's `arcgisViewReadyError` event
  and fails fast with the underlying error if it fires first.
- New flow doc: `docs/flows/web-component-ready.md`.

### Changed
- **Showcase:** `demo/showcase/king-county.html` migrated to the element-
  passing pattern. Pinned to `?v=0.14.1`.
- **README:** Option A example and the "Initialization order / Web
  component" section rewritten for the new pattern; old listener pattern
  shown as a diff for upgraders.
- **API doc:** `OperationalOptions.view` clarified to accept both shapes;
  new errors table entry for the readiness timeout.

### Compatibility
- Non-breaking: existing `operational.view: mapView` usage unchanged.
- Recommended: upgrade web-component demos to pass the element.

## [0.14.0] — 2026-04-15

### Added
- **Loading splash.** A neutral overlay covers the MapLibre canvas during
  `FuseSimple.create()` so the consumer never sees an unsynced or
  partially-tiled basemap. Dismisses itself (with a ~350ms fade) when
  MapLibre reports `idle`. A configurable timeout (default 10s) force-
  removes the splash if `idle` never fires and logs a `BUG`-level warning.
  Enabled by default — set `splash: { enabled: false }` to opt out.
  New module `src/core/splash.ts`; wired into the `create()` sequence
  between `getBasemapContainer()` and `basemapAdapter.mount()`.
- **`BasemapAdapter.onceIdle(callback)`** interface method, implemented
  in `MapLibreBasemapAdapter` via `map.once('idle', …)`. Returns an
  unsubscribe so mid-init `destroy()` cancels the pending listener
  cleanly.
- **`SplashOptions`** exported from the public entry so consumers can
  type their options: `enabled`, `background`, `text`, `textOpacity`,
  `fadeDuration`, `timeout`. `logoUrl` reserved for a future release.

### Changed
- `package.json` bumped to `0.14.0`.

## [0.13.1] — 2026-04-14

### Fixed
- **`OperationalAdapter` interface contract.** The interface in
  `src/adapters/types.ts` did not declare `destroy()`, but
  `FuseSimple.ts` called it on the concrete `EsriOperationalAdapter`.
  Added `destroy(): void` to the interface and narrowed the core's
  field type from `EsriOperationalAdapter` to `OperationalAdapter` so
  a future non-Esri implementation satisfies the same contract.

### Added
- **`docs/flows/adapter-contracts.md`** — new flow doc covering both
  adapter interfaces, the two v1 implementations, state ownership
  between core and adapters, destroy symmetry, and the checklist for
  adding a third engine.

### Changed (demos + docs)
- King County (web component demo) swapped from `dark-gray-vector`
  → `gray-vector` as its throwaway basemap attribute. FuseSimple
  strips it on `create()` and mounts the PMTiles basemap in its
  place. `<arcgis-map>` refuses to initialize without a `basemap`
  attribute, so this demo keeps one.
- LA Buses and Clustering (classic-API demos) now follow the
  recommended pattern: empty Map at construction
  (`new EsriMap({ layers: [] })`, no basemap), `FuseSimple.create()`
  drives view init, operational layers added only *after* `create()`
  resolves. Previously LA Buses added layers after its own
  `await view.when()`; Clustering attached its FeatureLayer at map
  construction.
- `README.md` quickstart (Option B) rewritten to demonstrate the
  recommended classic-API pattern: empty Map, `FuseSimple.create()`
  owns init, `map.add(...)` afterward. Removes the dangling
  `await view.when()` and the throwaway-basemap line that mixed two
  patterns.
- `README.md` now has a consolidated **"Initialization order"**
  section split into two subsections for the classic API
  (`new MapView(...)`, used by LA Buses + Clustering demos) and the
  web component (`<arcgis-map>`, used by King County). Each names
  the rules, shows the recommended pattern, and explains *why*
  — classic API lets FuseSimple own init so layer-stashing protects
  the consumer's `center`/`zoom`; web component has Esri own init
  before `arcgisViewReadyChange` fires, so stashing doesn't apply
  and the basemap attribute is mandatory. Closes with a
  failure-mode → cause table.
- `docs/standalone-guide.md` `<arcgis-map>` example bumped from
  `dark-gray-vector` → `gray-vector` for consistency with King
  County. "Basemap attribute is required" note extended with a
  paragraph explaining that `<arcgis-map>` view init has already
  happened by the time `arcgisViewReadyChange` fires, so
  FuseSimple's layer-stashing doesn't apply to the web component path.
- King County tweaks: initial zoom 12→13, parcels `minScale: 18000`,
  popup spacer.
- CDN bundle cache-buster bumped to `?v=0.13.1` in all three demos.

## [0.13.0] — 2026-04-13

### Changed
- **Dist layout renamed and split.** The CDN bundle's verbose
  `fusesimple.standalone.js` is gone. The drop-in CDN build now ships
  as `dist/fusesimple.js` (unminified, debuggable) and
  `dist/fusesimple.min.js` (minified, production) — matching the
  convention used by MapLibre, Leaflet, and D3. The npm-targeted
  library build moves to `dist/lib/` (`fusesimple.js`,
  `fusesimple.cjs`, `index.d.ts`) so the short CDN filenames don't
  collide with it.
- `package.json` `main` / `module` / `types` / `exports` now point at
  `./dist/lib/*`. `size-limit` budgets both the npm ES module
  (`dist/lib/fusesimple.js`, 250 KB gzip) and the minified CDN bundle
  (`dist/fusesimple.min.js`, 400 KB gzip).
- `npm run build:cdn` now builds both the unminified and minified
  variants in sequence. `npm run build:all` runs the full library +
  CDN build.

### Migration
- Hosted CDN: update `<script src>` / `import` paths from
  `fusesimple.standalone.js` to `fusesimple.min.js` (or
  `fusesimple.js` for debugging).
- npm consumers: no changes required — the package `exports` map
  resolves `@mapsimple/fusesimple` to the new `dist/lib/` paths
  automatically.

## [0.12.1] — 2026-04-13

### Fixed
- **Attribution pinned to right** — OSM credit and FuseSimple version are
  now injected as pinned `<span>` elements next to "Powered by Esri"
  instead of appended to the scrolling `__sources` text. Long layer
  attributions (e.g. the power plant dataset's multi-line copyright) no
  longer bury the OSM and FuseSimple credits. The injected elements use
  `flex-shrink: 0` to stay visible while `__sources` truncates with
  ellipsis on the left.

## [0.12.0] — 2026-04-13

### Fixed
- **Initial extent override** — layers added to `view.map` before
  `FuseSimple.create()` no longer hijack the consumer's `center`/`zoom`.
  Esri's MapView auto-fits to layer extents during initialization:
  layers with large extents zoom out to fit, while layers with no extent
  (empty `GraphicsLayer`, `StreamLayer` before data arrives) default to
  world extent (zoom 0). FuseSimple now stashes all layers at the top of
  `create()` (Step 0) and restores them after the initial sync push
  (Step 12a), so the view initializes with the consumer's intended
  viewport. This also ensures `Home` and other widgets capture the
  correct "home" viewpoint regardless of creation order. See
  `docs/initial-extent-investigation.md` for the root cause analysis.

### Changed
- **Lifecycle steps renumbered** — the `create()` sequence now starts at
  Step 0 (layer stashing) and includes Step 12a (layer restore). All
  docs updated to reflect the new numbering.

## [0.11.2] — 2026-04-13

### Fixed
- **Microtask-batched sync** — Three separate `reactiveUtils.watch()` calls
  (center, zoom, rotation) now coalesce into a single MapLibre `jumpTo()` +
  `redraw()` per frame via `queueMicrotask()`. Previously, a pinch-zoom
  triggered 2-3 redundant full WebGL render passes per frame.

## [0.11.1] — 2026-04-13

### Fixed
- **MutationObserver leak** — attribution observers are now stored and
  disconnected on `destroy()`. Previously they kept firing after teardown.
- **Duplicate PMTiles archive** — `mount()` reuses the archive created by
  `readHeader()` instead of creating a second instance and HTTP connection.
- **maxZoom via setter** — `setMaxZoom()` method replaces fragile
  post-construction mutation of the shared options object.
- **Orphaned container div** — `EsriOperationalAdapter.destroy()` now
  removes the basemap container div from the DOM.
- **emit/off iteration safety** — listener array is snapshot before
  iteration so `off()` mid-callback won't skip handlers.
- **FuseSimple version in attribution** — Esri attribution bar now shows
  `FuseSimple v0.11.x` alongside the OSM credit, injected automatically.

### Changed
- **`DebugLogger.log()` data param** narrowed from `any` to
  `Record<string, unknown>` for type safety.
- **`@protomaps/basemaps`** moved from `dependencies` to `devDependencies`
  (unused in source code).
- **Dead `sync.ts` scaffold removed** — the placeholder `startSync()` was
  never called. `esriRotationToMapLibreBearing` inlined into its test.
- **`_getRenderedZoom`** added to `BasemapAdapter` interface to match
  existing implementation.

## [0.11.0] — 2026-04-12

### Added
- **`setBasemapLayerVisibility(layerId, visible)`** — public method on
  `FuseSimple` to show/hide basemap style layers by MapLibre stylesheet
  ID. Backed by `setLayerVisibility()` on the `BasemapAdapter` interface
  and `MapLibreBasemapAdapter` implementation. No-ops silently if the
  layer doesn't exist. Logged under the `STYLE` debug tag.
- **`setLayerVisibility()`** added to the `BasemapAdapter` interface in
  `types.ts` so the contract is engine-agnostic.
- **Standalone CDN build** (`npm run build:cdn`) — produces a single
  self-contained ES module (`dist/fusesimple.standalone.js`, ~333 KB
  gzipped) with MapLibre GL JS and PMTiles baked in. Drop it into any
  HTML page alongside the Esri CDN script — no bundler, no
  `node_modules`. Consumer loads Esri from `js.arcgis.com`, imports
  FuseSimple from the standalone file, done.
- **Standalone demo** (`demo/standalone.html`) — full working example
  using only CDN script tags: Esri `<arcgis-map>` web component +
  FuseSimple standalone bundle + pre-built Protomaps Light style +
  King County parcels FeatureLayer with popup, Home widget, and layer
  toggle. Zero build step required.
- **Pre-built Protomaps Light style** (`demo/public/protomaps-light-full.json`)
  — static JSON export of the full `@protomaps/basemaps` Light style
  so the standalone demo (and any non-bundler consumer) can reference
  a complete basemap style without needing the npm package.
- **MapLibre adapter unit tests** (`src/adapters/maplibre.test.ts`) — 4
  tests covering `setLayerVisibility`: visible=true, visible=false,
  unmounted no-op, and missing-layer graceful handling.
- **`esriRotationToMapLibreBearing` tests** (`src/core/sync.test.ts`) —
  5 tests covering zero, positive, negative, fractional rotations, and
  the involution property.
- **JSDoc** on all public methods in `maplibre.ts`, `esri.ts`, and
  `FuseSimple.ts` (including `destroy()`, passthrough methods,
  `addAttribution`, `getBasemapContainer`).

### Changed
- **Migrated from deprecated `view.watch()` to `reactiveUtils.watch()`.**
  The Esri adapter and FuseSimple size watcher now use the modern
  `reactiveUtils.watch()` API instead of the deprecated
  `Accessor.watch()`. `reactiveUtils` is resolved dynamically at runtime
  (via `$arcgis.import` for CDN users, dynamic `import()` for npm users)
  so FuseSimple never has a static `@arcgis/core` import. Sync watchers
  use `{ sync: true }` for same-frame updates.
- **README.md rewritten** with two getting-started paths: Option A
  (drop-in script, no bundler) and Option B (npm install). Includes
  full API documentation for `create()`, `on()`/`off()`,
  `setBasemapLayerVisibility()`, `destroy()`, custom basemap styles,
  basemap interaction events, and debug logging.
- **Demo** — MapLibre test parcels toggle added to the main demo
  (`demo/index.html` + `demo/main.ts`) using the new
  `setBasemapLayerVisibility` API.

### Known Issues
- **`reactiveUtils.watch()` triggers deprecated `Accessor.watch`
  warnings.** Esri's own `reactiveUtils.watch()` implementation
  internally calls the deprecated `Accessor.watch()` method in SDK
  5.0.16. The warnings appear in the console with FuseSimple in the
  stack trace, but the root cause is Esri's internal implementation,
  not our usage. We are calling the correct recommended API.

## [0.10.0] — 2026-04-12

### Fixed
- **Drift fix: LOD scale/resolution consistency.** The `SCALE_AT_ZOOM_ZERO`
  constant (559,082,264) was independently computed and 5.83% inconsistent
  with the resolution formula. Esri derives `view.resolution` from LOD
  *scale* via the 96-DPI conversion, so the mismatch caused a scale
  difference between the two canvases: features aligned at center but
  diverged 10-16px at viewport edges. Replaced with
  `resolution / METERS_PER_PIXEL` so scale and resolution are always
  linked. Verified: 0.000px edge diff at all zoom levels.
- **Drift fix: fractional zoom interpolation.** Esri linearly interpolates
  resolution between LOD entries; MapLibre uses exponential. Passing
  `view.zoom` directly produced ~6% scale mismatch at fractional zooms.
  Added `zoomFromResolution()` to convert Esri's `view.resolution` to the
  equivalent MapLibre zoom, keeping both canvases pixel-locked at every
  fractional zoom level.
- **MapLibre maxZoom extended to 24.** MapLibre defaults to maxZoom 22,
  but our LODs extend to 24 for vector tile overzoom. Both engines now
  zoom to the same ceiling.

### Added
- Debug accessors on `FuseSimple` (`_getBasemapCenter`,
  `_getBasemapZoom`, `_getBasemapCanvasInfo`) and the MapLibre adapter
  for drift instrumentation.
- Debug overlay in demo (`demo/main.ts`, `demo/index.html`) showing
  real-time coordinate, zoom, canvas size, backing store, and CSS
  transform comparison between both engines.

### Changed
- `lod.ts`: Removed `SCALE_AT_ZOOM_ZERO` constant. Added `ESRI_DPI`,
  `METERS_PER_PIXEL`, `MAPLIBRE_TILE_SIZE`, and `zoomFromResolution()`.
  `scaleAtZoom()` now derives from `resolutionAtZoom()` instead of an
  independent constant.
- `FuseSimple.ts`: Sync handler and initial sync push now convert zoom
  via `zoomFromResolution(view.resolution)` instead of passing
  `view.zoom` directly to MapLibre.

## [0.9.0] — 2026-04-12

### Added
- **King County demo** (`demo/`) — first visual proof of FuseSimple in
  action. Esri MapView with transparent background over MapLibre +
  PMTiles basemap rendering full Protomaps Light style.
  - `demo/vite.config.ts` — dev server config with source alias for HMR.
  - `demo/main.ts` — wires `@arcgis/core` Map/MapView, builds Protomaps
    Light style via `@protomaps/basemaps`, calls `FuseSimple.create()`,
    adds King County parcels FeatureLayer with popup, Home widget, and
    layer toggle.
  - `demo/index.html` — full-viewport layout with parcels toggle UI.
  - `demo/actest.html` + `demo/style.json` — standalone MapLibre + PMTiles
    baseline test (no Esri, no FuseSimple) for comparison.
  - `demo/public/test-parcels.geojson` — 397 parcels extracted from
    the KC FeatureServer for alignment testing (rendered as red outlines
    on MapLibre alongside the same parcels from Esri).
- **Initial sync push** in `FuseSimple.create()` — pushes current Esri
  view state to MapLibre immediately after wiring the sync loop, so the
  basemap starts at the correct center/zoom instead of defaulting to
  world view.
- **`rotation` added to `DuckViewForCreate`** interface to support the
  initial sync push reading the view's current rotation.
- **LOD overzoom to zoom 24** — `MAX_OVERZOOM = 24` in `lod.ts`. Vector
  tiles re-render at every zoom (no stretching), so LODs now extend
  well beyond the PMTiles header's `maxZoom` to allow full street-level
  zoom, matching MapLibre's native max.
- **`overflow: hidden`** on the basemap container div — prevents a
  sub-pixel red gap at the right edge of the viewport.
- **`@protomaps/basemaps`** added as a devDependency for the demo
  (generates the full Protomaps Light style programmatically).

### Changed
- **Removed rAF batching from Esri sync** — `onViewStateChange` now
  calls the handler synchronously from the `view.watch` callback
  instead of deferring to the next `requestAnimationFrame`. This
  eliminates our side of the sync delay. MapLibre's `jumpTo()` +
  `redraw()` is called inline.
- **`setViewState` now calls `map.redraw()`** after `jumpTo()` to force
  a synchronous canvas paint. This is an attempt to eliminate the
  one-frame drift between the two WebGL render loops.

### Known Issues
- **Mid-pan drift between Esri and MapLibre canvases.** Despite
  synchronous watch callbacks and `map.redraw()`, the two independent
  WebGL render loops still produce visible drift during active panning.
  The parcels align at rest but shift by a few pixels mid-pan. This is
  the same fundamental problem maplibre-gl-leaflet addresses with CSS
  transform tricks and canvas padding. Further investigation needed:
  canvas padding (`padding: 0.1`), CSS transform sync, or
  `_getTransformForUpdate()` + `transform.apply()` pattern.
- **size-limit still reports ~293 KB** (vs 250 KB budget). Entirely
  from maplibre-gl v5 being larger than spec estimated.

## [0.8.0] — 2026-04-11

### Added
- **Real `src/core/FuseSimple.ts`** — the scaffolded stub is replaced
  with a working `FuseSimple.create()` async factory and `destroy()`:
  - `create()` is now `async` (returns `Promise<FuseSimple>`) because
    mounting the basemap (PMTiles header read + MapLibre style load) is
    inherently async. Call it **before** `await view.when()` per the
    spec.
  - **Step 1:** Web Mercator WKID check (`3857` / `102100`). Throws
    immediately with actionable message if the view uses a different SR.
  - **Step 2:** Instantiates both adapters, passing the shared
    `DebugLogger` with `isProduction` plumbed from options.
  - **Step 3:** Strips existing basemap (`view.map.basemap = null`)
    with a `LIFECYCLE` warning. Applies transparency via
    `view.background = { color: [0,0,0,0] }` + CSS canvas fallback.
  - **Step 4:** Reads PMTiles header, computes LODs via
    `computeLODsFromHeader()`, assigns to `view.constraints.lods` with
    `snapToZoom = false`. Logs a `HEADER` warning if the consumer's
    initial center falls outside the archive bounds.
  - **Step 5:** Mounts basemap into the sibling div.
  - **Step 6:** Injects OSM attribution (configurable via
    `basemap.attribution`, defaults to `"© OpenStreetMap"`).
  - **Step 7:** Starts the sync loop by subscribing to the Esri
    adapter's `onViewStateChange` and forwarding to the MapLibre
    adapter's `setViewState`.
  - **Step 8:** Wires click / hover passthrough: Esri miss → MapLibre
    `queryFeaturesAt` → public `basemap-click` / `basemap-hover` events.
  - `destroy()` unsubscribes sync + passthrough in reverse order,
    destroys the basemap adapter, clears listeners. Idempotent.
  - **Event emitter:** `on("basemap-click", handler)` /
    `on("basemap-hover", handler)` + matching `off()` for
    unsubscription. Uncaught handler errors are caught and logged under
    `BUG` rather than crashing the sync loop.
  - `BasemapClickEvent.features` and `BasemapHoverEvent.features`
    tightened from `any[]` to `BasemapFeature[]`. Dead `eslint-disable`
    comments removed. `OperationalOptions.view` tightened from `any` to
    `unknown`.
- **`src/index.ts`** now re-exports `BasemapFeature`, `LngLat`,
  `PMTilesHeader`, and `ViewState` from `src/adapters/types.ts` so
  consumers can type their event handlers without reaching into
  internal adapter paths.
- **`src/test-setup.ts`** — Vitest setup file that stubs
  `URL.createObjectURL` / `URL.revokeObjectURL` for jsdom. Required
  because `maplibre-gl` calls `createObjectURL` during module init to
  set up its web worker, which isn't available in jsdom.

### Changed
- **`vite.config.ts`** — `rollupOptions.external` now includes
  `"maplibre-gl"` and `"pmtiles"` alongside `@arcgis/core`. These are
  `dependencies` (not `devDependencies`), so the consumer's bundler
  resolves them. Without externalization the library bundle was 330 KB
  gzipped (the entire maplibre-gl source inlined).
- **`vitest.config.ts`** — added `setupFiles: ["src/test-setup.ts"]`
  for the `URL.createObjectURL` polyfill.
- **`.gitignore`** — excludes `docs/arcgis_js_v50_sdk/` (2.6 GB local
  SDK reference, not for git).

### Known Issues
- **Bundle budget exceeded.** `size-limit` reports ~293 KB gzipped
  (with all dependencies) against the 250 KB budget. FuseSimple's own
  code is 6.79 KB gzipped; the overage is entirely from `maplibre-gl`
  v5 being larger than the spec's ~210 KB estimate. Investigation
  deferred to `feature/demo` where we can measure real-world impact
  with the King County archive.

## [0.7.0] — 2026-04-11

### Added
- **Real `src/adapters/esri.ts`** — the scaffolded no-op stubs are
  replaced with a full `EsriOperationalAdapter` implementation:
  - **Duck-typed view narrowing** — the constructor validates that the
    `unknown` view quacks like a MapView (checks `container`, `center`,
    `zoom`, `rotation`, `watch`, `on`, `hitTest`, `toMap`) and throws
    with a clear message if any member is missing. No `@arcgis/core`
    import needed.
  - **`getBasemapContainer()`** creates a sibling `<div>` inside
    `view.container` with `position: absolute; inset: 0; z-index: 0;
    pointer-events: none;` and inserts it as the first child so it
    renders behind the Esri canvas. Cached on first call. CSS class
    `fusesimple-basemap-container` for DOM inspection / test targeting.
  - **`onViewStateChange(handler)`** subscribes to
    `view.watch("center, zoom, rotation")` with rAF batching: changes
    set a dirty flag, a single `requestAnimationFrame` callback reads
    the latest values, applies the `bearing = -rotation` sign-flip, and
    emits one `ViewState` per frame. Returns an unsubscribe function
    that removes the watch handle and cancels any pending rAF.
  - **`onPassthroughClick(handler)`** listens to `view.on("click")`
    and calls `view.hitTest()` at the click coordinates. If
    `results.length === 0` (no Esri feature hit), fires the handler
    with a `GeoScreenPoint` built from `event.x/y` and
    `event.mapPoint.longitude/latitude`.
  - **`onPassthroughHover(handler)`** listens to
    `view.on("pointer-move")` with rAF throttling: raw pointer-move
    events are coalesced to at most one `hitTest` per frame.
    `pointer-move` events don't carry `mapPoint`, so the adapter calls
    `view.toMap({x, y})` for coordinate conversion. Same
    passthrough-on-miss logic as click.
  - **`addAttribution(text)`** finds
    `.esri-attribution__sources` in the view container DOM, de-dupes
    against existing content, appends the text with a ` | ` separator,
    and installs a `MutationObserver` to re-inject if the Esri
    Attribution widget re-renders (e.g. layer changes).
  - Debug logging under `LIFECYCLE` / `SYNC` / `PASSTHROUGH` /
    `ATTRIBUTION` tags. Accepts an optional `logger` in options.

## [0.6.0] — 2026-04-11

### Added
- **`src/core/style-resolver.ts`** — pure helpers
  `deriveBasemapHost(pmtilesSource)` and
  `resolveStylePlaceholders(style, basemapHost)`. `deriveBasemapHost`
  strips `?query` and `#hash` before taking the directory portion of a
  PMTiles URL / path (a bare filename returns `""`, meaning "current
  origin"). `resolveStylePlaceholders` shallow-clones the input style
  and replaces every `{basemapHost}` occurrence in `sprite` / `glyphs`
  while leaving MapLibre's own `{fontstack}` / `{range}` tokens alone.
  No `maplibre-gl`, no `pmtiles`, no browser APIs — the whole module is
  pure so it runs under Vitest without loading the basemap stack.
- **`src/core/style-resolver.test.ts`** — 18 unit tests covering the
  derivation edge cases (https URL, subdirs, query string, hash
  fragment, relative path, absolute-from-root, bare filename, empty
  input, http+port, deep path) and the substitution semantics (sprite +
  glyphs, leaves `{fontstack}` / `{range}` alone, replaces every
  occurrence, passes pre-resolved URLs through unchanged, no-op on
  missing fields, non-mutation of the input, defensive pass-through for
  non-string `sprite` arrays, empty-host handling).
- **`src/styles/protomaps-light.json`** — minimal Protomaps-schema-
  compatible style stub that ships with FuseSimple. Declares `version
  8`, `sprite` / `glyphs` with `{basemapHost}` placeholders, an empty
  `sources` block (the adapter injects the PMTiles source at mount
  time), and five layers (background, earth, water, roads, buildings)
  referencing `source: "protomaps"` with the standard Protomaps
  source-layer names. Metadata marks it as a stub with a note to swap
  in the real Protomaps Light JSON before relying on FuseSimple for
  production cartography.
- **Real `src/adapters/maplibre.ts`** — the scaffolded throwing stubs
  are replaced with a full `MapLibreBasemapAdapter` implementation:
  - Module-level `sharedProtocol` singleton. `ensureProtocolRegistered`
    calls `maplibregl.addProtocol("pmtiles", sharedProtocol.tile)`
    exactly once per process. Subsequent adapters share the handler by
    calling `sharedProtocol.add(pmtiles)` with their own archive.
    `destroy()` does **not** call `removeProtocol` — doing so would
    break any other live adapter still using it.
  - `mount(container)` registers the protocol, builds a `PMTiles`
    archive, resolves the style (bundled stub vs inline object vs URL
    string), injects the PMTiles source into `sources.protomaps` if the
    style doesn't already define one, constructs
    `new maplibregl.Map({ interactive: false, trackResize: false,
    attributionControl: false })`, and awaits the `"load"` event. An
    `isDestroyed` flag is checked at every `await` point so
    `destroy()`-during-`mount` unwinds cleanly. A second call to
    `mount()` throws.
  - `readHeader()` caches after the first call and is safe to call
    before `mount()` — it lazily constructs a `PMTiles` archive if
    needed, so callers can read bounds / zoom range early for LOD
    computation without racing the Map constructor.
  - `setViewState({ center, zoom, bearing })` forwards via
    `map.jumpTo(...)`. Sign-flipping (`bearing = -rotation`) is the
    sync core's job — the adapter never sees raw Esri rotation.
  - `resize(width, height)` writes the container's CSS size and calls
    `map.resize()`. The caller is already responsible for rounding per
    Decisions §"CSS translate rounding".
  - `queryFeaturesAt(point)` calls `map.queryRenderedFeatures` and
    normalizes the hits into the leak-free `BasemapFeature` shape
    (`layer`, `properties`, optional `geometryType` via a positive
    allow-list that filters out `GeometryCollection`). MapLibre's
    `MapGeoJSONFeature` type never leaves this file.
  - `destroy()` calls `map.remove()`, drops the archive / header /
    container references, and flips `isDestroyed`. Idempotent in
    practice.
  - Debug logging under `LIFECYCLE` / `HEADER` / `STYLE` tags. The
    adapter accepts an optional `logger` in its options; if omitted it
    constructs its own `DebugLogger` so call sites never need a null
    check. `FuseSimple.create()` is expected to pass its own logger in
    so all adapter output lands under the same `?debug=` flag.
- **`docs/flows/basemap-lifecycle.md`** — TBD markers for the MapLibre
  adapter side closed. Entry points, state transitions (protocol
  registration rationale), and the destroy-during-mount failure mode
  now describe the shipped behavior. The "AbortSignal vs isDestroyed
  flag" open question is resolved in favor of the flag. Remaining TBDs
  all sit on the Esri adapter / `FuseSimple.ts` wire-up side.

### Changed
- **`src/core/style-resolver.ts` `StyleWithPlaceholders` interface** —
  `sprite?` and `glyphs?` widened from `string` to `unknown`. MapLibre
  5.x allows `sprite` to be an array of `{id,url}` entries for
  multi-sprite styles; the substitution logic already type-checks at
  runtime before rewriting, so accepting any shape on the type side
  lets the defensive test case (and real-world multi-sprite styles)
  compile cleanly.

## [0.5.0] — 2026-04-11

### Added
- **`src/core/lod.ts`** — pure PMTiles header → Esri `LOD[]` math helper.
  Exports `WEB_MERCATOR_CIRCUMFERENCE_METERS`, `TILE_SIZE_PIXELS`,
  `SCALE_AT_ZOOM_ZERO`, `MIN_ZOOM_FLOOR` constants; `resolutionAtZoom(z)`,
  `scaleAtZoom(z)`, `clampMinZoom(z)` helpers; and
  `computeLODsFromHeader({ minZoom, maxZoom })` which returns a
  structurally-`Esri.LOD`-compatible array covering `[clampMinZoom(minZoom),
  maxZoom]` inclusive. Throws on non-finite, negative, inverted, and
  below-zoom-1-floor ranges with explicit error messages. No Esri, no
  MapLibre, no browser APIs — the whole module is pure so it runs under
  Vitest without loading `@arcgis/core`.
- **`src/core/lod.test.ts`** — 20 unit tests pinning the math against
  Esri's published zoom-15 reference numbers (~4.777 m/px resolution,
  ~17061.84 scale denominator), verifying the zoom-0 floor, exercising
  King County's `[0, 15]` range, and covering every throw path.
- **`docs/flows/basemap-lifecycle.md`** — second real flow doc, stubbed
  with the 13-step bring-up sequence (SR check → basemap strip →
  transparency → sibling div → header read → LOD compute → constraints
  apply → protocol registration → Map construct → attribution → sync
  start → first paint). Mermaid sequence diagram included. Sections
  marked _TBD_ where behavior depends on adapter implementations still
  to land; LOD math section is concrete since the helper exists.

### Changed
- **`src/adapters/types.ts`** solidified. Extracted named interfaces
  `LngLat`, `ScreenPoint`, `GeoScreenPoint`, `PMTilesHeader`, and
  `BasemapFeature` from the inline shapes the original scaffold used.
  `BasemapAdapter.queryFeaturesAt` now returns `BasemapFeature[]` instead
  of `any[]` — MapLibre types stay inside the adapter layer, consumers
  get a minimal leak-free shape (layer / properties / optional geometry
  type). Added `BasemapAdapter.resize(width, height)` per
  `FuseSimple_v1_Decisions.md` §"Manual resize handling", with the
  CSS-rounding contract documented on the method. Removed the dead
  `eslint-disable` comment (Biome doesn't read it; `noExplicitAny` is
  off globally). Expanded JSDoc on every method and cross-referenced the
  relevant decisions doc sections.
- **`src/adapters/maplibre.ts`** updated to match the new `BasemapAdapter`
  contract: imports the named types from `./types`, adds the throwing
  `resize()` stub, changes `queryFeaturesAt` return type from `unknown[]`
  to `BasemapFeature[]`, inlines the PMTiles-protocol-ordering rationale
  where it matters (the constructor comment explaining why
  `addProtocol` is deferred to `mount`).
- **`src/adapters/esri.ts`** updated to match the new `OperationalAdapter`
  contract: `GeoScreenPoint` replaces the inline `{ x, y, lng, lat }`
  shape. Tightened `EsriAdapterOptions.view` from `any` to `unknown` with
  a documented reason; `getBasemapContainer()` now narrows the view with
  a duck check and throws a clear error if `view.container` isn't set
  instead of an opaque `TypeError`. Removed the two dead `eslint-disable`
  comments.

## [0.4.0] — 2026-04-11

### Added
- **`src/debug-logger.test.ts`** — 29 unit tests covering every documented
  `DebugLogger` behavior: construction (direct + factory + `isProduction`
  default), `?debug=all` / `?debug=false` / comma-list / no-param URL parsing
  including case-insensitivity, whitespace tolerance, and the `ALL` keyword
  inside a comma list; `BUG`-level semantics in dev (`console.warn`, default
  `UNKNOWN`/`GENERAL` fill-ins, custom-field preservation); `isProduction`
  hard lockdown silencing every tag including `BUG` and skipping the URL
  parse entirely; multi-instance isolation with independent feature lists,
  independent name prefixes, and one-sided lockdown; iframe parent-window
  fallback with `vi.stubGlobal` mocks for `location` / `parent` including
  cross-origin `SecurityError` handling; and `getConfig()` diagnostic
  surface in both dev and production modes. Tests use per-test URL stubs
  (`setUrl` / `setParent` / `setParentCrossOrigin` helpers) and
  `console.log` / `console.warn` spies installed in `beforeEach`.
- **`docs/flows/debug-logger.md`** — first real flow doc, replacing the
  template placeholder. Covers purpose, entry points, a mermaid sequence
  diagram for lazy `initialize()`, a per-instance state-transitions table,
  every documented failure mode (cross-origin parent, missing BUG fields,
  unregistered tags, production lockdown, no-iframe case), and an open
  question about per-instance URL namespacing deferred to v2.

### Changed
- **`src/index.test.ts`** no longer carries the only `DebugLogger`
  coverage — it stays as the tiny public-API sanity test (2 tests), and
  real semantics live in `src/debug-logger.test.ts`. Comment in that file
  updated to point at the new suite.

## [0.3.0] — 2026-04-11

### Added
- **Biome** (`@biomejs/biome`) for lint and format — single tool, one config,
  replaces the usual ESLint + Prettier + plugin stack. Config at `biome.json`:
  2-space indent, double quotes, semicolons, trailing commas, 100-char lines,
  recommended rules on, `noExplicitAny` off (logger and event emitter intentionally
  accept `any`).
- **Vitest** (`vitest` + `jsdom`) for unit tests. Config at `vitest.config.ts`:
  colocated `src/**/*.test.ts` pattern, jsdom environment (for `window`/URL
  parsing), explicit imports (no globals), `clearMocks`/`restoreMocks`, 5s
  default timeout.
- **`src/index.test.ts`** — minimal public-API sanity test (2 tests). Proves the
  `createDebugLogger` factory and `DebugLogger` class are exported, importable,
  and construct cleanly through the test infra. Real coverage lands in
  `feature/debug-logger-tests` next.
- **GitHub Actions CI** at `.github/workflows/ci.yml`: checkout, Node 20 via
  `.nvmrc`, `npm ci`, typecheck, lint, format check, tests, build, size check.
  Runs on pushes to `main`/`develop` and PRs into them. Concurrency group
  cancels in-flight runs on the same branch.
- **`.nvmrc`** pinning Node 20 for contributors.
- **simple-git-hooks + lint-staged** for pre-commit discipline. Pre-commit runs
  `biome check --write` on staged `.ts/.tsx/.js/.jsx/.json` files. `prepare`
  script auto-installs the hook on `npm install`.
- **size-limit** + `@size-limit/preset-small-lib` bundle size check wired to
  the 250 KB gzipped budget from spec acceptance criterion #8. Current: 725 B.
- **`docs/flows/` directory** with `README.md` (rules + index) and `_template.md`
  (purpose / entry points / sequence / state transitions / failure modes /
  debug tags / prior art references / open questions). One flow doc per
  subsystem, written alongside the code it documents.
- New `package.json` scripts: `lint`, `lint:fix`, `format`, `format:check`,
  `check`, `check:fix`, `test`, `test:run`, `size`, `prepare`.

### Fixed
- **`src/core/FuseSimple.ts`** — scaffold stub had an `on()` overload whose impl
  signature used `unknown`, which TypeScript rejected (function parameters are
  contravariant, so narrower handler types weren't assignable). Switched to
  `any`, matching the EventEmitter pattern used by Node, mitt, nanoevents, etc.
  Discovered the moment we wired `tsc --noEmit` into the pipeline — this is
  exactly why the tooling was added.
- **`src/adapters/maplibre.ts`** — scaffold stub constructor took options and
  threw them away, triggering Biome's `noUselessConstructor` rule. Constructor
  now stores options in a private field; unimplemented methods throw
  `"not implemented"` with context instead of silently no-oping, matching the
  existing `readHeader()` stub.

### Changed
- **`vite.config.ts`** — `vite-plugin-dts` now excludes `**/*.test.ts` from
  generated `.d.ts` emission so colocated tests never ship to consumers.
- **`src/debug-logger.ts`** — `.forEach()` calls converted to `for...of` loops
  per Biome's `noForEach` complexity rule. Behavior unchanged; `for...of` is
  faster and supports early return.

## [0.2.0] — 2026-04-11

### Added
- `DebugLogger` class ported from MapSimple ExB's URL-activated logger, adapted
  for a standalone library: `widgetName` renamed to `name`, `createDebugLogger`
  takes an options object.
- `isProduction` flag on `DebugLoggerOptions`. When `true`, the logger hard-locks
  down: URL params are ignored, `log()` no-ops for every tag, and `BUG` level is
  silenced as well. Intended for production builds where end-users could
  otherwise hit `?debug=all` and inspect internals.
- Public re-exports from `src/index.ts`: `DebugLogger`, `createDebugLogger`,
  `DebugLoggerOptions`, `DebugFeature`. Consumer apps can now register their own
  tags on the same `?debug=` URL flag FuseSimple uses internally.
- `CHANGELOG.md` (this file) and `TODO.md` at repo root. Both are updated in the
  same commit as the change they describe.

## [0.1.0] — 2026-04-11

### Added
- Initial repository scaffold: directory structure, `package.json`, `tsconfig.json`,
  `vite.config.ts`, `.gitignore`, MIT `LICENSE`, third-party `LICENSES.md`, public
  `README.md`, `CLAUDE.md` (project instructions).
- `docs/FuseSimple_v1_Spec.md` — canonical source of truth for what FuseSimple does.
- `docs/FuseSimple_v1_Decisions.md` — implementation decisions companion to the spec,
  including eight spec clarifications and a "Patterns from Prior Art" section.
- `docs/FuseSimple_v1_Prior_Art_Reference.md` — patterns from `maplibre-gl-leaflet`,
  `mapbox-gl-sync-move`, `maplibre-gl-compare`, `Leaflet.Sync`, and `@esri/maplibre-arcgis`.
- Stub source files for `src/core/FuseSimple.ts`, `src/core/sync.ts`,
  `src/adapters/types.ts`, `src/adapters/maplibre.ts`, `src/adapters/esri.ts`,
  `src/index.ts`.
- Stub demo files in `demo/`.
