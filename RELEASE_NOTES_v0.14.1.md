# FuseSimple v0.14.1

First public release — April 15, 2026.

---

## What is FuseSimple?

A JavaScript library that renders a free, serverless [PMTiles](https://docs.protomaps.com/pmtiles/) basemap underneath an [ArcGIS Maps SDK for JavaScript](https://developers.arcgis.com/javascript/latest/) `MapView`. One import, one `FuseSimple.create()` call, working map.

FuseSimple stacks a hidden MapLibre GL JS basemap under a transparent Esri MapView and keeps them in sync. The developer never touches MapLibre directly. You get the full Esri ecosystem (widgets, feature layers, editing, authentication, analysis) without paying for basemap tile credits or running a tile server.

Pre-1.0, in active development. This public drop is a preview for testing and feedback — API surface may shift before 1.0.

---

## Try it without installing

Three hosted demos exercise the library against real data:

- **[King County Parcels](https://fusesimple.mapsimple.org/demos/king-county.html)** — dual-source parcel layers proving pixel-perfect alignment between the two engines. Web-component pattern.
- **[Feature Clustering](https://fusesimple.mapsimple.org/demos/clustering.html)** — Esri's built-in clustering on power-plant data, running on a FuseSimple basemap. Classic `MapView` pattern.
- **[LA Buses](https://fusesimple.mapsimple.org/demos/la-buses.html)** — real-time `StreamLayer` over a FuseSimple basemap. Rotating chevron markers, trail polling, live connection status.

See `demo/showcase/README.md` for what each one demonstrates and a testing checklist.

---

## Changes

First public release — nothing to compare against. See [`CHANGELOG.md`](CHANGELOG.md) for the full per-version history of the 0.x line that led here.

---

## Install

### Drop-in script (no bundler)

Grab `dist/fusesimple.min.js` (or `dist/fusesimple.js` for debugging) and load it alongside the Esri CDN. MapLibre and PMTiles are baked in — one file, nothing else to install.

```html
<script type="module" src="https://js.arcgis.com/5.0/"></script>
<link rel="modulepreload" href="path/to/fusesimple.min.js?v=0.14.1" />

<arcgis-map basemap="gray-vector" center="-122.33, 47.6" zoom="12">
  <arcgis-zoom slot="top-left"></arcgis-zoom>
</arcgis-map>

<script type="module">
  import { FuseSimple } from "path/to/fusesimple.min.js?v=0.14.1";

  const fuse = await FuseSimple.create({
    basemap: { engine: "maplibre", source: "/my-region.pmtiles" },
    operational: { engine: "esri", view: document.querySelector("arcgis-map") },
  });
</script>
```

The `<link rel="modulepreload">` is recommended but optional. The bundle is ~1.5 MB minified (MapLibre + PMTiles + library). Preloading starts fetching it in parallel with the Esri SDK, avoiding a visible "gray basemap flash" on cold loads. See [`docs/cold-load-ux.md`](docs/cold-load-ux.md) for the full rationale.

### npm (bundler projects)

The npm package isn't published yet. For now, use the drop-in script. Watch [`CHANGELOG.md`](CHANGELOG.md) for the `@mapsimple/fusesimple` npm release.

### Requirements

- **Esri Maps SDK 5.0+** — peer dependency, bring your own (CDN or `@arcgis/core` once npm lands)
- **Web Mercator** (`wkid: 3857`) — v1 requirement
- **Modern browser with WebGL** — current Chrome, Firefox, Edge, Safari

---

## Known limits (v1)

What v1 deliberately doesn't do:

- **No SceneView / 3D / pitch / tilt.** 2D `MapView` only.
- **One basemap source.** Single PMTiles archive per FuseSimple instance.
- **No runtime basemap swap.** Pass style + source at `create()`; re-mount to change them.
- **No Leaflet / OpenLayers adapter.** v1 ships MapLibre (basemap) + Esri (operational) only. The adapter interface is engine-agnostic, so more engines are possible later.
- **One-way sync.** Esri → MapLibre for center, zoom, rotation. MapLibre state doesn't flow back.

See `docs/FuseSimple_v1_Spec.md` in the private source tree for the full spec (not shipped publicly).

---

## Debugging

FuseSimple has a URL-activated debug logger — no console.log noise by default, detailed tracing when you need it.

```
?debug=all               → every tag
?debug=SYNC,PASSTHROUGH  → specific tags (case-insensitive, comma-separated)
?debug=false             → explicitly off
(no param)               → off, zero overhead
```

Tags: `LIFECYCLE`, `HEADER`, `STYLE`, `SYNC`, `PASSTHROUGH`, `EVENTS`, `ATTRIBUTION`, `BUG`.

The logger is iframe-aware (checks `window.parent` too), so it works inside embedded widgets.

---

## Reporting bugs

- Reproducer on one of the hosted demos is ideal (they're known-good surfaces).
- Screenshot + browser console output usually sufficient.
- Append `?debug=all` before reloading to capture detailed logs.
- Open an issue on the public repo: <https://github.com/MapSimple-Org/FuseSimple-Public/issues>

---

## License

MIT. See `LICENSE`. Third-party attributions in `LICENSES.md` (MapLibre GL JS, PMTiles, Protomaps).

© 2025–2026 MapSimple Organization
