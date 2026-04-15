# Cold-Load UX: Avoiding the Gray-Basemap Flash

## The Problem

On a cold cache (or fresh CDN edge, or slow connection), users sometimes
see a brief window where the bare Esri `gray-vector` basemap is visible
before FuseSimple mounts and swaps it out for the PMTiles basemap. The
sequence:

1. HTML parses, browser starts fetching `js.arcgis.com/5.0/`.
2. Esri SDK arrives, registers `<arcgis-map>` as a custom element.
3. `<arcgis-map>`'s `connectedCallback` fires; the view starts
   initializing immediately.
4. Esri paints the `gray-vector` basemap. **User sees gray streets.**
5. Eventually `fusesimple.min.js` finishes loading.
6. `FuseSimple.create()` runs, strips the basemap, makes the view
   transparent, mounts the splash, and reveals the PMTiles basemap
   underneath.

Steps 4 → 6 are the flash. On a fast cache it's invisible. On a cold
cache or slow connection it can be a full second or more of "wrong-
looking map," which feels like a bug to first-time visitors.

## When It Matters

| Setup | Flash visible? |
|---|---|
| Local file:// or local dev server | No — bundle loads in milliseconds |
| Production CDN, warm browser cache | No — bundle hits disk cache |
| Production CDN, warm CDN edge, cold browser | Brief, often <200ms |
| Production CDN, **cold CDN edge** (fresh deploy, distant region) | Visible — sometimes seconds |
| Slow connection (3G, congested wifi) | Visible regardless of cache state |

The 1.5 MB CDN bundle is the dominating factor. The flash is
proportional to how long that fetch takes.

## Mitigations

Four approaches. They're not mutually exclusive.

### 1. Preload the bundle (recommended default)

**Status:** Shipped in all three showcase demos. Documented in README.

Add a `<link rel="modulepreload">` in `<head>` so the browser starts
fetching FuseSimple in parallel with the Esri SDK, instead of waiting
to discover the `<script type="module">` in `<body>`:

```html
<head>
  <script type="module" src="https://js.arcgis.com/5.0/"></script>
  <link rel="modulepreload"
        href="https://your-cdn.example.com/fusesimple.min.js?v=0.14.1" />
</head>
```

**Pros:** one line of HTML, no library change, no behavior change,
saves the time it would have taken the parser to reach the inline
module script (typically 100s of ms, sometimes a full second).

**Cons:** doesn't eliminate the flash on truly cold loads — it just
shrinks it. If the bundle takes 5 seconds to fetch cold, preload makes
it 4.5 instead of 5.

**When to use:** always. It's free.

### 2. Defer Esri SDK loading until FuseSimple is ready

**Status:** Documented here; not implemented in any demo.

Drop the static Esri SDK script tag, dynamic-import it after FuseSimple
resolves. The `<arcgis-map>` element sits in the DOM as an unknown
element until Esri registers it — no init, no gray flash.

```html
<head>
  <link rel="modulepreload" href="https://js.arcgis.com/5.0/" />
  <link rel="modulepreload"
        href="https://your-cdn.example.com/fusesimple.min.js?v=0.14.1" />
</head>
<body>
  <arcgis-map basemap="gray-vector" center="..." zoom="..."></arcgis-map>

  <script type="module">
    const { FuseSimple } = await import(
      "https://your-cdn.example.com/fusesimple.min.js?v=0.14.1"
    );
    await import("https://js.arcgis.com/5.0/");   // Esri registers NOW
    await FuseSimple.create({ /* ... */ });
  </script>
</body>
```

Both bundles are preloaded in parallel via `<link rel="modulepreload">`,
so total wall time stays at `max(fuse, esri)`, not `fuse + esri`.

**Pros:** completely eliminates the flash. Pure visual win.

**Cons:** sequences the *register* step instead of running both in
parallel — adds a tiny amount of latency (probably 10s of ms once both
are preloaded). Slightly more complex HTML pattern.

**When to use:** when the flash is genuinely user-visible and bothering
people. Not the default because the cost is paid every load, while the
flash only happens on cold loads.

### 3. Hide `<arcgis-map>` with CSS until FuseSimple is mounted

**Status:** Documented here; not implemented in any demo.

```html
<style>
  arcgis-map { visibility: hidden; }
  arcgis-map.fuse-ready { visibility: visible; }
</style>

<arcgis-map basemap="gray-vector" ...></arcgis-map>

<script type="module">
  import { FuseSimple } from "...";
  const mapEl = document.querySelector("arcgis-map");
  await FuseSimple.create({ operational: { engine: "esri", view: mapEl } });
  mapEl.classList.add("fuse-ready");
</script>
```

Esri still inits underneath, but the user sees nothing until FuseSimple
is in place.

**Pros:** smallest possible code change, no library involvement.

**Cons:** the screen is **blank** during the gap, with no splash and no
indicator of progress. Worse perceived UX than option 1 unless paired
with a CSS-only loading message.

**When to use:** rarely. Mostly listed for completeness — most apps
should pick option 1 or 2 instead.

### 4. Mount the splash earlier in the library lifecycle (library change)

**Status:** Not implemented. Proposed.

Today the splash is mounted inside `FuseSimple.create()` *after* we've
awaited the Esri view's readiness — which is too late, Esri has already
painted gray. A library change could either:

- (a) Mount the splash as the **first** thing `create()` does, before any
  awaits. Catch: `create()` doesn't run until the consumer reaches the
  `await FuseSimple.create()` line in their code. So this only helps if
  the consumer's IIFE runs early enough.

- (b) Expose a separate `FuseSimple.mountSplash(target)` API that the
  consumer can call **immediately** on import, before `create()`:

  ```js
  import { FuseSimple } from "...";
  FuseSimple.mountSplash(document.querySelector("arcgis-map"));
  // ... time passes, Esri inits underneath the splash ...
  await FuseSimple.create({ /* ... */ });   // splash already up
  ```

  The splash positions itself absolutely over the target element. When
  `create()` resolves and MapLibre reaches `idle`, the splash fades out
  as it does today.

**Pros:** works for both classic and web-component patterns. Doesn't
fight the framework. Generalizable across all consumers without HTML
gymnastics.

**Cons:** requires library work — splitting splash mount from
`create()`, exposing a new entry point, documenting the new API.

**When to use:** the right long-term answer. Worth doing in a future
release once we have time to design the API properly.

## Why the Bundle Is Big — Bundle Architecture Notes

The CDN bundle weighs ~1.5 MB minified (335 KB gzipped) because we
package MapLibre GL JS + PMTiles inside it. The library code itself is
tiny:

| Artifact | Size | What's in it |
|---|---|---|
| `dist/fusesimple.min.js` | 1.4 MB | FuseSimple + MapLibre + PMTiles |
| `dist/lib/fusesimple.js` (npm) | 36 KB | FuseSimple only — MapLibre + PMTiles are external |

So the CDN bundle is **97% MapLibre + PMTiles, 3% FuseSimple**. The
size is the price of "no build tools required" — consumers get one URL
to import and everything works.

### A "lite" CDN variant

We could ship a third artifact, `fusesimple.lite.min.js`, that
externalizes MapLibre and PMTiles too. Consumers would load all three
from CDNs and use an import map to wire them together:

```html
<script type="importmap">
{
  "imports": {
    "maplibre-gl": "https://unpkg.com/maplibre-gl@5.0.0/dist/maplibre-gl.js",
    "pmtiles": "https://unpkg.com/pmtiles@4.0.0/dist/pmtiles.js"
  }
}
</script>

<link rel="modulepreload" href="https://unpkg.com/maplibre-gl@5.0.0/dist/maplibre-gl.js" />
<link rel="modulepreload" href="https://unpkg.com/pmtiles@4.0.0/dist/pmtiles.js" />
<link rel="modulepreload" href="https://your-cdn.example.com/fusesimple.lite.min.js" />

<script type="module">
  import { FuseSimple } from "https://your-cdn.example.com/fusesimple.lite.min.js";
</script>
```

**Implementation:** add a third invocation in `vite.config.cdn.ts` that
extends `rollupOptions.external` with `maplibre-gl` and `pmtiles`, with
a different output filename. ~5 lines of build config.

**Tradeoffs:**

- **Smaller FuseSimple file** (~36 KB instead of 1.4 MB) — fast cold-
  load on its own.
- **Three CDN URLs to fetch instead of one** — more DNS lookups, more
  parallel HTTP requests. Mitigated by preloading all three.
- **Shared cache wins** — if a user's browser has loaded MapLibre from
  unpkg for any other site recently, that fetch is free. Same for
  PMTiles. The mono bundle never gets that benefit.
- **Version coupling** — consumers have to know which MapLibre and
  PMTiles versions to pin. We'd document the supported range. Risk of
  consumers pinning incompatible versions and breaking. Less of a
  concern than it sounds: MapLibre 5.x and PMTiles 4.x are both stable.
- **Import maps are required** — supported in all modern browsers
  (Chrome 89+, Firefox 108+, Safari 16.4+) but not in older
  enterprise/IE-era browsers. The mono bundle has no such requirement.

### Recommendation

Ship both. `fusesimple.min.js` stays the default — one URL, works
everywhere, no setup. `fusesimple.lite.min.js` is the variant for
consumers who care about bundle size or already have MapLibre loaded
for other reasons. README documents both with guidance on when to pick
which.

Not urgent. Worth doing once we have a real consumer with a bundle-
size complaint to validate the demand.

## Per-Demo Status (as of 0.14.1)

| Demo | Preload (1) | Defer (2) | Hide (3) | Splash early (4) |
|---|---|---|---|---|
| `king-county.html` | ✅ | — | — | — |
| `la-buses.html` | ✅ | — | — | — |
| `clustering.html` | ✅ | — | — | — |

If we add a `lite` variant, we'd probably add a fourth showcase demo
(`king-county-lite.html` or similar) that uses the import-map pattern,
to prove the path works.
