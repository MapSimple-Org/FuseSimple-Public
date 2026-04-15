# FuseSimple API Reference

> **Version:** 0.14.0 (pre-1.0 — no API stability guarantees until 1.0)
> **For:** consumers integrating FuseSimple into an application
> **See also:** [`README.md`](../README.md) for getting-started patterns · [`docs/flows/`](flows/) for runtime walkthroughs · [`docs/standalone-guide.md`](standalone-guide.md) for CDN-specific usage

---

## Contents

- [Overview](#overview)
- [Factory](#factory)
  - [`FuseSimple.create(options)`](#fusesimplecreateoptions)
- [Instance methods](#instance-methods)
  - [`on(event, handler)`](#oneventname-handler)
  - [`off(event, handler)`](#offeventname-handler)
  - [`setBasemapLayerVisibility(layerId, visible)`](#setbasemaplayervisibilitylayerid-visible)
  - [`destroy()`](#destroy)
- [Debug-only methods](#debug-only-methods)
- [Options](#options)
  - [`FuseSimpleOptions`](#fusesimpleoptions)
  - [`BasemapOptions`](#basemapoptions)
  - [`OperationalOptions`](#operationaloptions)
  - [`SplashOptions`](#splashoptions)
- [Events](#events)
  - [`basemap-click`](#basemap-click)
  - [`basemap-hover`](#basemap-hover)
- [Public types](#public-types)
- [Errors](#errors)
- [Debug logger](#debug-logger)

---

## Overview

FuseSimple exposes one class (`FuseSimple`) and a handful of supporting types. Consumers use the `FuseSimple.create()` factory to build an instance, listen for passthrough events, and eventually call `destroy()` to tear it down.

Everything on this page is exported from the package root:

```typescript
import {
  FuseSimple,
  // types
  type FuseSimpleOptions,
  type BasemapOptions,
  type OperationalOptions,
  type SplashOptions,
  type BasemapClickEvent,
  type BasemapHoverEvent,
  type BasemapFeature,
  type LngLat,
  type PMTilesHeader,
  type ViewState,
  // debug logger
  DebugLogger,
  createDebugLogger,
  type DebugLoggerOptions,
  type DebugFeature,
} from "@mapsimple/fusesimple";
```

Consumers never import from `maplibre-gl` or `pmtiles` — those are internal. `@arcgis/core` stays a peer dependency (you bring your own).

---

## Factory

### `FuseSimple.create(options)`

```typescript
static async create(options: FuseSimpleOptions): Promise<FuseSimple>
```

Creates a FuseSimple instance, mounts the PMTiles basemap under the Esri MapView, and wires the sync + passthrough pipelines. The returned promise resolves once the basemap's style has loaded and the initial sync push has run — the map is interactive by that point (though tiles may still be loading; the loading splash covers that window).

**Parameters:**

- `options` — [`FuseSimpleOptions`](#fusesimpleoptions)

**Returns:** `Promise<FuseSimple>`

**Throws:**

- `Error("FuseSimple v1 requires a Web Mercator MapView (WKID 3857/102100) …")` — the MapView's `spatialReference.wkid` is not Web Mercator. v1 is Web-Mercator-only.
- `Error("EsriOperationalAdapter … view.container is not set …")` — the MapView lacks a DOM container.
- `Error("MapLibreBasemapAdapter.mount() called twice …")` — internal invariant, shouldn't fire from consumer code.

**Order-of-operations notes:**

- **Classic API (`new MapView(...)`):** call `create()` *before* you `await view.when()` yourself. FuseSimple will drive init and its layer-stashing step protects your `center`/`zoom` from being overridden by attached layers' extents.
- **Web component (`<arcgis-map>`):** call `create()` inside the `arcgisViewReadyChange` event handler. Esri has already driven init by the time the event fires; layer-stashing is inapplicable but post-init layer adds don't re-fit, so it's fine.
- Add operational layers *after* `create()` resolves when using the classic API. See [`README.md` → "Initialization order"](../README.md#initialization-order) for the full rule.

---

## Instance methods

### `on(eventName, handler)`

```typescript
on(event: "basemap-click", handler: (e: BasemapClickEvent) => void): void;
on(event: "basemap-hover", handler: (e: BasemapHoverEvent) => void): void;
```

Subscribe to a FuseSimple event. See [Events](#events) for payload shapes and fire conditions.

**Idempotency:** no. Registering the same `(event, handler)` pair twice subscribes twice — the handler fires twice per event.

---

### `off(eventName, handler)`

```typescript
off(event: "basemap-click", handler: (e: BasemapClickEvent) => void): void;
off(event: "basemap-hover", handler: (e: BasemapHoverEvent) => void): void;
```

Remove a previously registered listener. The `handler` argument must be the **exact same function reference** passed to `on()`. Anonymous/inline handlers cannot be removed this way — store a reference first if you need to unsubscribe.

```javascript
const handler = (e) => console.log(e);
fuse.on("basemap-click", handler);
// ...
fuse.off("basemap-click", handler);
```

---

### `setBasemapLayerVisibility(layerId, visible)`

```typescript
setBasemapLayerVisibility(layerId: string, visible: boolean): void
```

Show or hide a MapLibre basemap style layer by its stylesheet ID. Mirrors MapLibre's `setLayoutProperty(id, 'visibility', …)` behind the adapter wall so consumers never import `maplibre-gl`.

**Parameters:**

- `layerId` — the `id` of a layer in the MapLibre style. For the bundled Protomaps Light style, layer IDs follow the Protomaps convention (`roads-highway`, `water`, etc.). For a custom style, use the `id` values from your style JSON.
- `visible` — `true` to show, `false` to hide.

**No-ops** silently if the layer doesn't exist (style not yet loaded, wrong ID). Check the console with `?debug=STYLE` to see a missing-layer log.

---

### `destroy()`

```typescript
destroy(): void
```

Tear down the FuseSimple instance: unsubscribe sync watchers, remove all passthrough listeners, destroy the MapLibre map instance, remove the basemap sibling `<div>` from the DOM, and clear all user-registered event listeners. Safe to call multiple times — subsequent calls are no-ops.

Does **not** destroy the Esri MapView itself — that's the consumer's lifecycle to manage.

If called mid-init (before `create()` has resolved), FuseSimple unwinds cleanly: the pending MapLibre load is cancelled, the splash (if mounted) is removed, and no DOM residue is left.

---

## Debug-only methods

These are underscore-prefixed to signal "not part of the stable API." They're handy when building instrumentation or debugging alignment bugs, but their signatures can change between patch versions without notice.

```typescript
_getBasemapCenter(): { lng: number; lat: number } | null
_getBasemapZoom(): number | null
_getBasemapCanvasInfo(): {
  width: number;
  height: number;
  centerX: number;
  centerY: number;
} | null
```

All return `null` if the basemap isn't mounted yet or has been destroyed.

Use these for diagnostics only. The internal demo's black overlay (Esri zoom vs MapLibre zoom diff) is built on `_getBasemapZoom()`.

---

## Options

### `FuseSimpleOptions`

Top-level options passed to `FuseSimple.create()`.

| Property | Type | Default | Description |
|---|---|---|---|
| `basemap` | [`BasemapOptions`](#basemapoptions) | — (required) | Basemap configuration. |
| `operational` | [`OperationalOptions`](#operationaloptions) | — (required) | Operational (Esri) view configuration. |
| `splash` | [`SplashOptions`](#splashoptions) | `{}` (enabled) | Loading splash overlay. |
| `isProduction` | `boolean` | `false` | When `true`, locks the internal debug logger off. Ignores `?debug=` URL parameter and suppresses **all** logging including `BUG` level. Use in production builds where end users could otherwise activate debug output. |

---

### `BasemapOptions`

```typescript
interface BasemapOptions {
  engine: "maplibre";
  source: string;
  style?: string | object;
  attribution?: string;
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `engine` | `"maplibre"` | — (required) | Basemap engine identifier. Only `"maplibre"` is supported in v1. |
| `source` | `string` | — (required) | URL or relative path to the PMTiles archive. Passed unchanged to `new PMTiles(...)`. |
| `style` | `string \| object` | bundled Protomaps Light | MapLibre style. Three shapes: omit for the bundled stub; pass a URL string to fetch; pass a style object to use inline. `{basemapHost}` placeholders in the object are resolved from `source`. |
| `attribution` | `string` | `"© OpenStreetMap"` | Attribution text injected into the Esri Attribution widget. |

---

### `OperationalOptions`

```typescript
interface OperationalOptions {
  engine: "esri";
  view: unknown;
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `engine` | `"esri"` | — (required) | Operational engine identifier. Only `"esri"` is supported in v1. |
| `view` | `unknown` (MapView \| `<arcgis-map>` element) | — (required) | Either an `@arcgis/core` `MapView` instance (classic API) or an `<arcgis-map>` element (web component, Esri Maps SDK 5.0+). When an element is passed, `FuseSimple.create()` awaits its `viewOnReady()` internally (15 s timeout). Typed as `unknown` so consumer apps don't need `@arcgis/core` resolvable at FuseSimple's compile time. Must resolve to a MapView with `spatialReference.wkid` of `3857` or `102100`. |

---

### `SplashOptions`

```typescript
interface SplashOptions {
  enabled?: boolean;
  background?: string;
  text?: string;
  textOpacity?: number;
  fadeDuration?: number;
  timeout?: number;
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `enabled` | `boolean` | `true` | Master switch. `false` skips mounting the overlay entirely. |
| `background` | `string` (CSS color) | `"#f5f5f5"` | Background of the overlay. Any valid CSS color. |
| `text` | `string` | `"FuseSimple"` | Centered label. Set `""` for a blank overlay. |
| `textOpacity` | `number` (0–1) | `0.4` | Opacity of the label element. |
| `fadeDuration` | `number` (ms) | `350` | Fade-out duration when the splash dismisses. `0` for instant removal. |
| `timeout` | `number` (ms) | `10000` | Max time before the splash force-removes itself if MapLibre never reaches `idle`. Logs a `BUG`-level warning if hit. |

**Not currently configurable** (planned): `logoUrl`, label font size/weight/color. Font size scales responsively via `clamp(18px, 2.6vw, 32px)`. Use a `[data-fusesimple-splash]` CSS selector to override label styling at the site level.

See [`docs/flows/loading-splash.md`](flows/loading-splash.md) for the runtime flow.

---

## Events

FuseSimple emits events when a click or hover on the Esri MapView **misses** all Esri features — the interaction "passes through" to the basemap. The event payload includes any MapLibre features hit at that point.

### `basemap-click`

```typescript
interface BasemapClickEvent {
  features: BasemapFeature[];
  lngLat: LngLat;
}
```

**Fires when:** user clicks on the MapView, Esri's hit-test returns no features, and FuseSimple queries MapLibre at the click point. The event fires regardless of whether MapLibre features were found — `features` is `[]` if none.

**Throttling:** none. Clicks are discrete events; no need to throttle.

**Example:**

```javascript
fuse.on("basemap-click", (e) => {
  if (e.features.length === 0) return;
  const parcel = e.features.find(f => f.layer === "parcels-fill");
  if (parcel) {
    console.log(parcel.properties.ADDR_FULL);
  }
});
```

---

### `basemap-hover`

```typescript
interface BasemapHoverEvent {
  features: BasemapFeature[];
  lngLat: LngLat;
}
```

**Fires when:** pointer moves over the MapView and Esri's hit-test returns no features. Same payload shape as `basemap-click`.

**Throttling:** one event per animation frame (~60Hz on most displays) to keep handlers cheap even during fast cursor sweeps.

---

## Public types

### `BasemapFeature`

A normalized representation of a MapLibre feature, surfaced through passthrough events.

```typescript
interface BasemapFeature {
  layer: string;
  properties: Record<string, unknown>;
  geometryType?:
    | "Point"
    | "LineString"
    | "Polygon"
    | "MultiPoint"
    | "MultiLineString"
    | "MultiPolygon";
}
```

| Property | Type | Description |
|---|---|---|
| `layer` | `string` | MapLibre style layer ID the feature belongs to. |
| `properties` | `Record<string, unknown>` | Feature attributes from the PMTiles archive. Shape depends on the data. |
| `geometryType` | GeoJSON geometry type | Optional — present when the underlying engine exposes it. |

---

### `LngLat`

```typescript
interface LngLat {
  lng: number;
  lat: number;
}
```

Geographic coordinate pair. Longitude first (MapLibre convention — note Esri uses `{longitude, latitude}` which is the opposite key names but same order).

---

### `PMTilesHeader`

```typescript
interface PMTilesHeader {
  bounds: [number, number, number, number]; // [west, south, east, north]
  center: LngLat;
  minZoom: number;
  maxZoom: number;
}
```

Archive metadata read from the PMTiles file. Not surfaced on any public method directly in v1 — exported for consumer-side diagnostics via the `_getBasemapCanvasInfo` debug escape hatch and for future features (e.g., fit-to-bounds helpers).

---

### `ViewState`

```typescript
interface ViewState {
  center: LngLat;
  zoom: number;
  bearing: number;
}
```

Post-conversion view state: center, MapLibre-style zoom, and bearing. `bearing` is already sign-flipped from Esri's `rotation` (Esri `rotation = -MapLibre bearing`, per spec §3.3).

Exported for consumers who want to build their own sync observers or debug overlays.

---

## Errors

Every exception `FuseSimple.create()` throws is a plain `Error` with a message prefix that makes the cause grep-able.

| Prefix | Cause | Fix |
|---|---|---|
| `"FuseSimple v1 requires a Web Mercator MapView (WKID 3857/102100)"` | The MapView's `spatialReference.wkid` is not Web Mercator. | Reconstruct your MapView with `{ spatialReference: { wkid: 3857 } }`. v1 is Web-Mercator-only. |
| `"FuseSimple: <arcgis-map> did not become ready within 15000ms"` | An `<arcgis-map>` element was passed but never reached ready state. Usually a missing `basemap` attribute, malformed `item-id`, or a blocked ArcGIS portal request. | Set `basemap` on the element; verify `center`/`zoom` attributes; check the console for `arcgisViewReadyError`. |
| `"FuseSimple: <arcgis-map> failed to become ready — …"` | Esri emitted `arcgisViewReadyError` before readiness completed. The `…` is the underlying Esri message. | Address the underlying Esri error — usually basemap or item-id misconfiguration. |
| `"EsriOperationalAdapter.getBasemapContainer(): view.container is not set"` | The MapView has no DOM container. | Pass a `container` string or element to the MapView constructor before calling `FuseSimple.create()`. |
| `"MapLibreBasemapAdapter.mount() called twice"` | Internal invariant — a single adapter instance was mounted twice. | Shouldn't fire from consumer code. If it does, file an issue. |
| `"MapLibreBasemapAdapter.mount() called after destroy()"` | `destroy()` ran before `create()` finished mounting. | Usually benign — FuseSimple unwinds gracefully. Only visible if you're instrumenting adapter internals. |

There is no `FuseSimpleError` custom class in v1. If we add one, it'll be additive (all existing throws will continue to work as plain `Error`).

---

## Debug logger

FuseSimple uses its own URL-activated debug logger internally. The same logger is **re-exported** for consumer apps so you can register your own tags and piggyback on the shared `?debug=` URL flag + production lockdown.

### `DebugLogger`

```typescript
class DebugLogger {
  constructor(options: DebugLoggerOptions);
  log(feature: DebugFeature, data: Record<string, unknown>): void;
  getConfig(): { name: string; features: DebugFeature[]; enabled: string[]; isProduction: boolean };
}
```

### `createDebugLogger(options)`

```typescript
function createDebugLogger(options: DebugLoggerOptions): DebugLogger
```

Factory helper. Identical to `new DebugLogger(options)`.

### `DebugLoggerOptions`

```typescript
interface DebugLoggerOptions {
  name: string;
  features: string[];
  isProduction?: boolean;
}
```

| Property | Type | Description |
|---|---|---|
| `name` | `string` | Prefix for all log entries from this logger. FuseSimple uses `"FUSESIMPLE"`. |
| `features` | `string[]` | Registered tags. FuseSimple registers `LIFECYCLE`, `HEADER`, `STYLE`, `SYNC`, `PASSTHROUGH`, `EVENTS`, `ATTRIBUTION`, `BUG`. Consumer apps register their own. |
| `isProduction` | `boolean` | When `true`, **all** logging is suppressed, including `BUG` level. `?debug=` is ignored. Default `false`. |

### Activation

When `isProduction` is `false`:

- `?debug=all` — enable every registered tag
- `?debug=LIFECYCLE,SYNC` — enable specific tags (comma-separated, case-insensitive)
- `?debug=false` — explicitly disable all logging
- (no param) — disabled, zero runtime cost

The logger is iframe-aware: if `?debug=` isn't on the current window URL, it checks `window.parent`. This matters because FuseSimple may run inside an Experience Builder widget or other iframed host.

### FuseSimple's internal tags

| Tag | Covers |
|---|---|
| `LIFECYCLE` | `create()`, mount, destroy, teardown ordering, listener cleanup, splash mount/dismiss |
| `HEADER` | PMTiles archive header parse, bbox/zoom bounds, LOD computation |
| `STYLE` | Basemap style resolution (bundled / URL / inline), load, apply |
| `SYNC` | Esri `view.watch` → MapLibre `setCenter`/`setZoom`/`setBearing` |
| `PASSTHROUGH` | hitTest filter, `queryRenderedFeatures` call, coordinate translation, hover throttling |
| `EVENTS` | `basemap-click` / `basemap-hover` `on/off/emit` activity |
| `ATTRIBUTION` | OSM attribution injection into the Esri Attribution widget |
| `BUG` | Always-on (until `isProduction: true`). For known issues with `bugId` / `category` / `description` fields |

See [`docs/flows/debug-logger.md`](flows/debug-logger.md) for the full runtime walkthrough.
