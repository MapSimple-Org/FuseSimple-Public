# Standalone Bundle Guide

How to use `fusesimple.js` / `fusesimple.min.js` — the zero-bundler,
drop-in path for adding a PMTiles basemap under an Esri MapView.

## What it is

A single ES module with MapLibre GL JS and PMTiles baked in. You load it
alongside the Esri 5.0 CDN and call `FuseSimple.create()`. No npm, no
bundler, no MapLibre configuration. The only external dependency is
`@arcgis/core`, which the Esri CDN provides.

Two variants ship side-by-side:

- `fusesimple.js` — unminified, for local debugging. Readable source maps.
- `fusesimple.min.js` — minified, for production CDN use. Same behavior.

## Building

```bash
npm run build:cdn
```

This runs the CDN Vite config twice (unminified, then `MINIFY=1`) and
outputs both `dist/fusesimple.js` and `dist/fusesimple.min.js`. The build
bundles `maplibre-gl` and `pmtiles` but externalizes `@arcgis/core`
(loaded from the Esri CDN at runtime). The sibling npm library build
lives at `dist/lib/`.

## Minimal example

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>FuseSimple Standalone</title>
  <style>html, body { height: 100%; margin: 0; }</style>
  <script type="module" src="https://js.arcgis.com/5.0/"></script>
</head>
<body>
  <arcgis-map basemap="gray-vector" center="-122.33, 47.6" zoom="12"
              style="height: 100%">
    <arcgis-zoom slot="top-left"></arcgis-zoom>
  </arcgis-map>

  <script type="module">
    import { FuseSimple } from "./fusesimple.min.js";

    const mapEl = document.querySelector("arcgis-map");
    mapEl.addEventListener("arcgisViewReadyChange", async () => {
      const fuse = await FuseSimple.create({
        basemap: {
          engine: "maplibre",
          source: "/my-region.pmtiles",
        },
        operational: {
          engine: "esri",
          view: mapEl.view,
        },
      });
    });
  </script>
</body>
</html>
```

## Key patterns

### Basemap attribute is required

`<arcgis-map>` won't initialize its view without a `basemap` attribute.
Set any value — `FuseSimple.create()` strips it automatically and replaces
it with your PMTiles basemap. Without this, `mapEl.view` stays `null` and
`arcgisViewReadyChange` never fires.

In the web component flow, `arcgisViewReadyChange` fires *after* the
view has already initialized, so FuseSimple's layer-stashing (used in
the npm/classic-API path) doesn't apply here — the view is already
past init by the time your handler runs. That's fine: Esri doesn't
re-fit an already-initialized view when you `map.add()` layers, so you
can add layers either before or after `FuseSimple.create()`. See the
README "Initialization order" section for the full story on classic
API timing.

### Use `arcgisViewReadyChange`, not `await mapEl.ready`

`mapEl.ready` is a boolean, not a promise. `await mapEl.ready` resolves
immediately to `false` and your code runs before the view exists. The
correct pattern is:

```javascript
mapEl.addEventListener("arcgisViewReadyChange", async () => {
  // mapEl.view is now a fully initialized MapView
});
```

### Loading Esri modules dynamically

In the standalone (CDN) context, use `$arcgis.import()` instead of ES
`import` for Esri modules:

```javascript
const FeatureLayer = await $arcgis.import("esri/layers/FeatureLayer");
const parcels = new FeatureLayer({ url: "https://..." });
mapEl.map.add(parcels);
```

This global is registered by the Esri CDN script. FuseSimple uses it
internally to resolve `reactiveUtils`.

### Custom basemap style

The bundled Protomaps Light stub works for basic cartography. For a full
style with labels, icons, and proper theming, generate a static JSON from
`@protomaps/basemaps` and fetch it at runtime:

```javascript
const styleResponse = await fetch("/protomaps-light-full.json");
const style = await styleResponse.json();

const fuse = await FuseSimple.create({
  basemap: {
    engine: "maplibre",
    source: "/my-region.pmtiles",
    style,
  },
  operational: { engine: "esri", view: mapEl.view },
});
```

The demo includes a pre-built `demo/public/protomaps-light-full.json` as
a reference. To generate your own:

```javascript
import { layers, namedFlavor } from "@protomaps/basemaps";
const style = {
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
};
```

### Basemap events

Clicks and hovers that miss Esri features pass through to the basemap:

```javascript
fuse.on("basemap-click", (e) => {
  console.log(e.lngLat);      // { lng, lat }
  console.log(e.features);    // BasemapFeature[]
});
```

### Layer visibility

Toggle basemap layers by their MapLibre stylesheet ID:

```javascript
fuse.setBasemapLayerVisibility("parks-fill", false);  // hide
fuse.setBasemapLayerVisibility("parks-fill", true);   // show
```

### Debug logging

Append `?debug=all` to your URL to see all internal logging, or target
specific subsystems:

```
?debug=LIFECYCLE,SYNC
```

Append `?debug=false` to explicitly disable even if other code enables it.

To lock out logging entirely in production, set `isProduction: true` in
your `FuseSimple.create()` call. This ignores the `?debug=` URL parameter
and suppresses all output, including `BUG` level.

**Standalone (CDN) — use the hostname or a global flag:**

```javascript
const isProduction = location.hostname !== "localhost";

const fuse = await FuseSimple.create({
  basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
  operational: { engine: "esri", view: mapEl.view },
  isProduction,
});
```

**npm (bundler) — use your build tool's environment variable:**

```javascript
const fuse = await FuseSimple.create({
  basemap: { engine: "maplibre", source: "/tiles.pmtiles" },
  operational: { engine: "esri", view },
  isProduction: import.meta.env.PROD, // Vite
  // or: process.env.NODE_ENV === "production"  // webpack / Node
});
```

When `isProduction` is `false` (the default), logging is still off unless
activated by `?debug=` in the URL. So the flag is only needed to prevent
end-users from discovering `?debug=all` in deployed builds.

### Cleanup

```javascript
fuse.destroy();
```

## Known issues

- **Sprite URL warning:** The bundled Protomaps Light stub has a relative
  `/sprite` path that MapLibre rejects as "Invalid sprite URL." The map
  renders without icons. Use a full style with an absolute sprite URL for
  proper cartography.
- **`reactiveUtils` deprecation warnings in console:** Esri's own
  `reactiveUtils.watch()` internally calls the deprecated
  `Accessor.watch()`. These warnings come from the Esri SDK, not from
  FuseSimple. Nothing actionable on our side.

## Running the standalone demo

```bash
npm run demo:standalone
```

This builds the CDN bundle first, then starts a Vite dev server. Open
`http://localhost:5173/demo/standalone.html` to see the full standalone
demo with King County parcels, popup, home button, and layer toggle.
