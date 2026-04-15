# FuseSimple Showcase Demos

Three single-file HTML demos that exercise FuseSimple against real data. Each one pulls the library from `fusesimple.mapsimple.org` and a PMTiles archive from `tiles.mapsimple.org`. No build tools, no npm install — just a browser.

## Try them hosted

The easiest way to see the demos running is to open the hosted versions:

- King County Parcels — <https://fusesimple.mapsimple.org/demos/king-county.html>
- Feature Clustering — <https://fusesimple.mapsimple.org/demos/clustering.html>
- LA Buses — <https://fusesimple.mapsimple.org/demos/la-buses.html>

## Try them locally

Each `.html` file in this folder is self-contained and points at the hosted library + tiles. To run locally, either:

1. Open the file directly in a browser (`file://` works), or
2. Serve the folder with any static server:

   ```bash
   cd demo/showcase
   python3 -m http.server 8080
   # then open http://localhost:8080/king-county.html
   ```

You only need the library source if you want to rebuild FuseSimple itself. For testing the demos, the CDN-hosted library is the intended path.

## What each demo shows

### King County Parcels — `king-county.html`

Dual-source parcel demo that proves pixel-perfect alignment between the two engines. Two independent parcel layers trace the same King County boundaries:

- **Red outlines** — GeoJSON rendered by MapLibre on the basemap
- **Shaded polygons** — Esri `FeatureLayer` from ArcGIS Online

Toggle each layer independently and zoom in to verify alignment. Clicking a red outline (a basemap-only feature) opens an Esri popup through FuseSimple's `basemap-click` passthrough — a good demonstration that data from either engine can surface through the same popup UI.

Uses the `<arcgis-map>` web component pattern with the Esri Maps SDK 5.0.

PMTiles archive: `king_county.pmtiles` (~104 MB), tight county-scale bounds around Seattle / Snoqualmie, zoom 0–15.

### Feature Clustering — `clustering.html`

Esri's built-in feature clustering running on top of a FuseSimple basemap. Power plant data from ArcGIS Online is clustered with Arcade expressions that filter and aggregate coal-fueled plants inside each cluster. Includes a Legend widget inside an Expand panel.

Based on the Esri sample: [featurereduction-cluster-popup-filter](https://developers.arcgis.com/javascript/latest/sample-code/featurereduction-cluster-popup-filter/). The only change from the sample is that the Esri basemap is replaced by a FuseSimple + PMTiles basemap — the clustering code is identical.

Uses the classic `MapView` + `<div>` pattern with the Esri Maps SDK 5.0.

PMTiles archive: `pnw.pmtiles` (~1.2 GB), regional bounds covering Washington, Oregon, Idaho, and slivers of neighboring states, zoom 0–15.

### LA Buses — `la-buses.html`

Real-time `StreamLayer` fed by a WebSocket from Esri's GeoEvent sample server, pushing fictitious LA Metro bus positions over a FuseSimple basemap. Chevron markers rotate by heading, colors are derived from the route ID via an Arcade hash into a 12-color palette, and a trail system caps each bus at 15 dots by polling the `StreamLayerView` every 3 seconds.

A collapsible stats panel shows bus / route / active / stale counts alongside connection status, and a toggle adds or removes the stream at runtime.

Classic `MapView` + `<div>` pattern with the Esri Maps SDK 5.0.

PMTiles archive: `la.pmtiles` (~199 MB), metro-area bounds from Malibu to the San Gabriel foothills, zoom 0–15.

## What to look for when testing

A quick checklist for verifying FuseSimple is behaving correctly:

- **Basemap renders underneath Esri.** You should see vector roads, labels, and landcover from the PMTiles archive, not an Esri basemap.
- **Pan, zoom, and rotate stay in sync.** Drag to pan, scroll to zoom, right-drag (or two-finger rotate on a trackpad) to rotate. The MapLibre basemap should track the Esri view frame-for-frame with no visible lag.
- **Attribution shows OSM + FuseSimple.** Click the attribution widget (bottom-right on the Esri view). You should see OpenStreetMap credit injected alongside Esri's.
- **Demo-specific features work.** Layer toggles in King County, cluster popups in Clustering, the live bus stream in LA Buses.
- **First load is responsive.** The FuseSimple library is ~1.5 MB minified (MapLibre + PMTiles are bundled). On a cold CDN, you may briefly see the bare Esri basemap before the PMTiles basemap mounts. If you want to avoid this, preload the bundle in your own pages — see the `README.md` at the repo root.

## Reporting issues

If you hit a bug or unexpected behavior, a screenshot plus the browser console output is usually enough to diagnose. To get detailed logs, append `?debug=all` to the demo URL and reload — FuseSimple's internal debug logger will print what it's doing at each stage (header parse, style load, sync, passthrough, attribution).

## Hosted assets reference

Two hosts split by content type:

**`fusesimple.mapsimple.org`** — site, library, demos, static assets:

- `/demos/{king-county,clustering,la-buses}.html` — hosted copies of these demos
- `/dist/fusesimple.js` / `/dist/fusesimple.min.js` — CDN bundle (MapLibre + PMTiles included). Unminified for debugging, minified for production.
- `/assets/json/protomaps-light-full.json` — Protomaps Light basemap style
- `/assets/data/test-parcels.geojson` — sample parcel boundaries for the King County demo

**`tiles.mapsimple.org`** — PMTiles archives only:

- `king_county.pmtiles` — King County, WA (~104 MB, zoom 0–15)
- `la.pmtiles` — Los Angeles metro (~199 MB, zoom 0–15)
- `pnw.pmtiles` — Pacific Northwest WA/OR/ID (~1.2 GB, zoom 0–15)
