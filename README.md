# FuseSimple

Render a free, serverless PMTiles basemap underneath an ArcGIS Maps SDK for JavaScript MapView. One import, one call, working map.

FuseSimple stacks a hidden MapLibre GL JS basemap under a transparent Esri MapView and keeps them in sync. The developer never touches MapLibre directly. They get the full Esri ecosystem (widgets, feature layers, editing, authentication, analysis) without paying for basemap tile credits or running a tile server.

> Status: pre-1.0, in active development. See `docs/FuseSimple_v1_Spec.md` for the full v1 spec.

**Reference:** [API docs](docs/API.md) · [Flow docs](docs/flows/) · [Changelog](CHANGELOG.md)

## Live demos

Three hosted demos exercise the library against real data. No install, just a browser:

- **[King County Parcels](https://fusesimple.mapsimple.org/demos/king-county.html)** — dual-source parcel layers proving pixel-perfect alignment between the two engines. Web-component pattern.
- **[Feature Clustering](https://fusesimple.mapsimple.org/demos/clustering.html)** — Esri's built-in clustering on power-plant data, on a FuseSimple basemap. Classic `MapView` pattern.
- **[LA Buses](https://fusesimple.mapsimple.org/demos/la-buses.html)** — real-time `StreamLayer` over a FuseSimple basemap, with rotating markers and live connection status.

Source for all three lives under [`demo/showcase/`](demo/showcase/) — single-file HTML, no build tools. See [`demo/showcase/README.md`](demo/showcase/README.md) for what each one demonstrates and a testing checklist.

## Getting Started

Three things to bring: a PMTiles file, an Esri MapView, and your data.

### Option A: Drop-in script (no bundler)

Grab `fusesimple.min.js` (or `fusesimple.js` for debugging) and load it alongside the Esri CDN. MapLibre and PMTiles are baked in — one file, nothing else to install.

```html
<html>
<head>
  <style> html, body { height: 100%; margin: 0; } </style>
  <script type="module" src="https://js.arcgis.com/5.0/"></script>
</head>
<body>
  <arcgis-map basemap="gray-vector" center="-122.33, 47.6" zoom="12"
              style="height:100%">
    <arcgis-zoom slot="top-left"></arcgis-zoom>
    <arcgis-home slot="top-left"></arcgis-home>
  </arcgis-map>

  <script type="module">
    import { FuseSimple } from "./fusesimple.min.js";

    // Pass the <arcgis-map> element directly — FuseSimple awaits its
    // viewOnReady() internally. No addEventListener needed; avoids a
    // real listener race on cache-warm loads where the event can fire
    // before the listener attaches.
    const fuse = await FuseSimple.create({
      basemap: {
        engine: "maplibre",
        source: "/my-region.pmtiles",
      },
      operational: {
        engine: "esri",
        view: document.querySelector("arcgis-map"),
      },
    });
  </script>
</body>
</html>
```

> **Note:** Set any `basemap` on the `<arcgis-map>` element so the view initializes. FuseSimple strips it automatically and replaces it with your PMTiles basemap.

#### Cache-busters when loading from a CDN

If you're loading FuseSimple from your own CDN or a shared host, pin the import with a version query string:

```html
<script type="module">
  import { FuseSimple } from "https://your-cdn.example.com/fusesimple.min.js?v=0.14.0";
</script>
```

The `?v=…` is a **cache-buster**: browsers treat every distinct query string as a different URL for caching purposes, so bumping the version (`?v=0.14.0` → `?v=0.14.1`) invalidates the cached copy and forces a fresh fetch the next time the page loads.

Two things worth knowing:

- The server doesn't parse the query string — it serves whatever file is currently deployed at that path. The cache-buster is purely a client-side cache-identity hack.
- So the ordering for a release is always: deploy the new `dist/` to the CDN **first**, then bump `?v=…` in your HTML. Otherwise you're forcing a fresh fetch of the old file.

Our showcase demos (`demo/showcase/*.html`) all use this pattern pinned to the current library version.

#### Preload the bundle

FuseSimple's standalone CDN bundle is ~1.5 MB minified (MapLibre + PMTiles + the library itself). By default the browser won't start fetching it until it parses the `<script type="module">` tag that imports it — typically in `<body>`, after the `<head>` and any stylesheets. On a cold CDN edge or slow connection, that delay is visible to the user: they see the bare Esri basemap for a beat before FuseSimple mounts.

Fix it with a `<link rel="modulepreload">` in `<head>`:

```html
<head>
  <link rel="stylesheet" href="https://js.arcgis.com/5.0/esri/themes/light/main.css" />
  <script type="module" src="https://js.arcgis.com/5.0/"></script>

  <!-- Start fetching FuseSimple in parallel with the Esri SDK -->
  <link rel="modulepreload" href="https://your-cdn.example.com/fusesimple.min.js?v=0.14.1" />
</head>
```

`modulepreload` is the right directive for ES modules — it preserves module parse caching and is cheaper than plain `preload as="script"`. Keep the `?v=…` cache-buster identical to the one on your `import` line so the preload and the actual import resolve to the same cached entry.

Especially worth it on fresh deploys (cold CDN edge, slow origin first-byte) and for users on high-latency connections.

### Option B: npm install (bundler projects)

```bash
npm install @mapsimple/fusesimple
```

`@arcgis/core` 5.0+ is a peer dependency — bring your own.

```javascript
import "@arcgis/core/assets/esri/themes/light/main.css";

import EsriMap from "@arcgis/core/Map";
import FeatureLayer from "@arcgis/core/layers/FeatureLayer";
import MapView from "@arcgis/core/views/MapView";
import { FuseSimple } from "@mapsimple/fusesimple";

// 1. Set up your Esri data as usual — feature layers, popups, renderers.
const parcels = new FeatureLayer({
  url: "https://services.arcgis.com/.../FeatureServer/0",
  popupTemplate: {
    title: "{ADDR_FULL}",
    content: [{ type: "fields", fieldInfos: [
      { fieldName: "ADDR_FULL", label: "Address" },
      { fieldName: "CTYNAME", label: "City" },
    ]}],
  },
});

// 2. Create an empty Map. No basemap, no layers yet — FuseSimple will
//    mount the PMTiles basemap in step 4, and operational layers go on
//    in step 5 (after the view initializes at your specified
//    center/zoom). See "Initialization order" below for the full rule.
const map = new EsriMap({ layers: [] });

// 3. Create the MapView. Web Mercator (3857) is required for v1.
const view = new MapView({
  container: "viewDiv",
  map,
  center: [-122.33, 47.6],
  zoom: 12,
  spatialReference: { wkid: 3857 },
});

// 4. Call FuseSimple.create(). It drives view init for you — no need
//    to await view.when() yourself. Point `source` at your PMTiles
//    file — local path or URL.
const fuse = await FuseSimple.create({
  basemap: {
    engine: "maplibre",
    source: "/my-region.pmtiles",
  },
  operational: {
    engine: "esri",
    view,
  },
});

// 5. View is now initialized at your specified center/zoom. Add
//    operational layers now — post-init adds don't re-fit the extent.
map.add(parcels);
```

That's it. Your Esri feature layers render on top with full popup and widget support. The PMTiles basemap renders underneath with zero configuration.

### Initialization order

Esri ships two patterns for constructing a `MapView`, and FuseSimple behaves a little differently under each. The patterns are not mutually exclusive — some apps mix them — but understanding which one you're in is the key to not fighting Esri's init.

**Esri init behaviors FuseSimple has to work around** (true under both patterns):

1. `MapView` needs something with a resolvable extent to initialize against — a basemap, or a layer with a compatible spatial reference. Without one, `view.when()` never resolves.
2. When a view initializes with layers already attached, Esri auto-fits the viewport to their combined extent — overriding whatever `center`/`zoom` you passed.
3. Adding layers to an already-initialized view does **not** re-fit. They appear at the current view position.

FuseSimple works around (1) and (2) with a layer-stashing maneuver during `create()`: strip the basemap, pull any attached layers off the map, await `view.when()` with an empty map (your `center`/`zoom` wins), then restore the layers and mount the PMTiles basemap. But this only fires if FuseSimple is the one triggering `view.when()`. Which pattern you pick decides whether that's possible.

#### Classic API (imperative) — `new MapView(...)`

This is the long-standing Esri pattern: construct the view in JavaScript and hand it to FuseSimple. You control when `view.when()` is awaited. Used by the LA Buses and Clustering demos.

**Rules:**

1. **Call `FuseSimple.create()` before any `await view.when()` in your own code.** This lets FuseSimple own init. Layer-stashing can then protect your `center`/`zoom`.
2. **Add operational layers *after* `FuseSimple.create()` resolves.** Start with `new EsriMap({ layers: [] })` and `map.add(...)` after `create()`. Post-init adds don't re-fit the extent.
3. **If you must await `view.when()` yourself first**, set a basemap on the Esri Map at construction. Any throwaway (`"gray-vector"` is our default) — FuseSimple will strip it. Without one, Esri has nothing to init against and hangs.

**Recommended pattern:**

```javascript
const map = new EsriMap({ layers: [] });                // empty
const view = new MapView({
  map,
  center: [-122.33, 47.6],
  zoom: 12,
  spatialReference: { wkid: 3857 },
});

const fuse = await FuseSimple.create({                  // <-- FuseSimple drives init
  basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
  operational: { engine: "esri", view },
});

// View is now initialized at your specified center/zoom. Safe to add layers:
map.add(parcels);
map.add(streamLayer);
```

#### Web component (declarative) — `<arcgis-map>`

The Esri 5.0+ pattern: declare the map in HTML with attributes and let Esri drive init. Used by the King County demo.

Under this pattern, **Esri owns init — FuseSimple can't.** The `<arcgis-map>` custom element starts initializing the moment Esri's SDK registers it, independent of your code.

As of 0.14.1, pass the `<arcgis-map>` element directly as `operational.view`. FuseSimple detects the element and awaits its `viewOnReady()` hook internally — a Stencil-style promise that chains `componentOnReady()` + `view.whenReady()` and resolves immediately if the component is already ready. No listener wiring, no race.

**Rules:**

1. **Always set the `basemap` attribute on `<arcgis-map>`.** The component refuses to initialize without it. Any value works — FuseSimple strips it inside `create()`.
2. **Set `center` and `zoom` as attributes on `<arcgis-map>`** (not imperatively later). These attributes are Esri's input to the init pass that has already run by the time FuseSimple takes over.
3. **Layer-add timing doesn't matter here** — post-init adds don't re-fit, and the init has already happened regardless. You can `mapEl.view.map.add(...)` before or after `FuseSimple.create()` resolves, though the recommended pattern is to grab `mapEl.view` after `create()` returns (below).

**Recommended pattern:**

```html
<arcgis-map basemap="gray-vector" center="-122.33, 47.6" zoom="12">
  <arcgis-zoom slot="top-left"></arcgis-zoom>
</arcgis-map>

<script type="module">
  import { FuseSimple } from "./fusesimple.min.js";

  const mapEl = document.querySelector("arcgis-map");

  const fuse = await FuseSimple.create({
    basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
    operational: { engine: "esri", view: mapEl },
  });

  // mapEl.view is populated once create() resolves. Add operational
  // layers here — post-init adds don't re-fit the extent.
  const view = mapEl.view;
  const FeatureLayer = await $arcgis.import("esri/layers/FeatureLayer");
  view.map.add(new FeatureLayer({ url: "..." }));
</script>
```

**Upgrading from the old `arcgisViewReadyChange` pattern** is optional but recommended:

```diff
- const mapEl = document.querySelector("arcgis-map");
- mapEl.addEventListener("arcgisViewReadyChange", async () => {
-   const fuse = await FuseSimple.create({
-     basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
-     operational: { engine: "esri", view: mapEl.view },
-   });
- });
+ const mapEl = document.querySelector("arcgis-map");
+ const fuse = await FuseSimple.create({
+   basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
+   operational: { engine: "esri", view: mapEl },
+ });
```

The old pattern still works (passing `mapEl.view` from inside the event handler), but it has a latent race on cache-warm loads: if `arcgisViewReadyChange` fires before your `addEventListener` attaches, your handler never runs. The new pattern sidesteps this entirely because `viewOnReady()` is a promise, not an event — it captures state, not a moment.

If an `<arcgis-map>` element never becomes ready (missing `basemap` attribute, malformed `item-id`, blocked portal), FuseSimple rejects after a 15 s timeout with a diagnostic message. It also listens for Esri's `arcgisViewReadyError` event and fails fast with the underlying error if that fires first.

#### What goes wrong if you ignore the rules

| Symptom | Likely cause |
|---|---|
| View hangs, `view.when()` never resolves | Classic API, no basemap, no layer, and FuseSimple isn't driving init. Esri has nothing to init against. |
| Viewport lands somewhere other than your specified `center`/`zoom` | Classic API, layers were in the map when the view initialized and Esri auto-fit to their extent. Move the `map.add()` calls to after `FuseSimple.create()`, or rely on FuseSimple to drive init. |
| `<arcgis-map>` renders nothing | Web component, no `basemap` attribute. The element won't initialize without one; FuseSimple will reject after 15 s with a hint. |
| Pre-0.14.1 web-component demo fails silently to load | `arcgisViewReadyChange` fired before your listener attached (cache-warm race). Upgrade to the element-passing pattern above. |

### Custom basemap style

FuseSimple ships a minimal Protomaps Light stub. For production cartography, pass a full MapLibre style object:

```javascript
import { layers, namedFlavor } from "@protomaps/basemaps";

const fuse = await FuseSimple.create({
  basemap: {
    engine: "maplibre",
    source: "/my-region.pmtiles",
    style: {
      version: 8,
      glyphs: "https://protomaps.github.io/basemaps-assets/fonts/{fontstack}/{range}.pbf",
      sprite: "https://protomaps.github.io/basemaps-assets/sprites/v4/light",
      sources: {
        protomaps: {
          type: "vector",
          url: "pmtiles:///my-region.pmtiles",
          attribution: "© OpenStreetMap",
        },
      },
      layers: layers("protomaps", namedFlavor("light"), { lang: "en" }),
    },
  },
  operational: { engine: "esri", view },
});
```

### Basemap interaction events

Clicks and hovers that miss Esri features pass through to the basemap. Use this to query basemap data (POIs, parcels, etc.) without adding them as Esri layers:

```javascript
fuse.on("basemap-click", (e) => {
  console.log(e.lngLat);           // { lng, lat }
  console.log(e.features);         // BasemapFeature[] at that point
  console.log(e.features[0].layer);       // MapLibre layer ID
  console.log(e.features[0].properties);  // feature attributes
});

fuse.on("basemap-hover", (e) => {
  // Same shape as basemap-click. Throttled to one per animation frame.
});
```

### Layer visibility

Toggle basemap layers on and off by their MapLibre stylesheet ID:

```javascript
fuse.setBasemapLayerVisibility("parks-fill", false);   // hide
fuse.setBasemapLayerVisibility("parks-fill", true);    // show
```

### Loading splash

FuseSimple covers the basemap with a neutral overlay during initialization so the consumer never sees an unsynced or partially-tiled map. It fades out automatically when MapLibre reports `idle` (all tiles loaded, camera settled).

Defaults: `#f5f5f5` background, `"FuseSimple"` label at 0.4 opacity, 350ms fade, 10s timeout safeguard. The label size scales with the viewport — smaller on phones, larger on desktop.

```javascript
const fuse = await FuseSimple.create({
  basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
  operational: { engine: "esri", view },
  splash: {
    text: "Loading map",
    background: "#fafafa",
    textOpacity: 0.5,
    fadeDuration: 300,
    timeout: 15000,
  },
});
```

Pass `splash: { enabled: false }` to opt out entirely.

#### `SplashOptions` reference

All properties are optional — omit an object or a single field to take the default.

| Property | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | Master switch. `false` skips mounting the overlay entirely; `onceIdle` subscription and teardown still run but are no-ops. |
| `background` | `string` (CSS color) | `"#f5f5f5"` | Background of the overlay. Any valid CSS color — hex, `rgb()`, `hsl()`, named. |
| `text` | `string` | `"FuseSimple"` | Centered label. Set to `""` for a blank overlay (no text, no logo yet). |
| `textOpacity` | `number` (0–1) | `0.4` | Opacity applied to the label element. Below ~0.2 the text becomes hard to read; above 0.6 it starts to compete with the background. |
| `fadeDuration` | `number` (ms) | `350` | Fade-out duration when the splash dismisses. `0` for instant removal, no transition. Industry-standard dismissal range is 200–400ms. |
| `timeout` | `number` (ms) | `10000` | Safety net. If MapLibre never reaches `idle` (silent failure, network stall), the splash removes itself after this many milliseconds and logs a `BUG`-level warning. |

**Not currently configurable** (future additions): label font size/weight/color, `logoUrl`. The font size scales responsively via CSS `clamp(18px, 2.6vw, 32px)`; override at the site level with a `data-fusesimple-splash` CSS selector if you need to break the default (see below).

#### Styling via CSS

The overlay root carries a `data-fusesimple-splash` attribute, so CSS can override anything the options don't expose:

```css
[data-fusesimple-splash] > div {
  font-family: "Inter", sans-serif;
  color: #0066cc;
  letter-spacing: 0.08em;
}
```

Note this is a second-class path — use the `SplashOptions` above when they cover what you need. CSS overrides are for the edges.

### Cleanup

```javascript
fuse.destroy();
```

Unsubscribes all watchers, removes the basemap, and clears event listeners. Safe to call multiple times.

### Debug logging

Append `?debug=all` to your URL to see internal sync, passthrough, and lifecycle logging. Or target specific subsystems:

```
?debug=SYNC,PASSTHROUGH
```

For production builds, suppress all logging (including the `?debug=` URL check):

```javascript
const fuse = await FuseSimple.create({
  basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
  operational: { engine: "esri", view },
  isProduction: true,
});
```

## Development

```bash
npm install
npm run dev        # library dev (Vite)
npm run demo       # run the King County demo
npm run build      # build the npm library to dist/lib/
npm run build:cdn  # build the CDN bundles to dist/fusesimple.js + .min.js
npm run build:all  # build both library and CDN bundles
npm run typecheck  # tsc --noEmit
npm test           # run unit tests
```

## License

MIT. See `LICENSE`. Third-party attributions in `LICENSES.md`.
