# Flow Documents

Per-subsystem walkthroughs of how FuseSimple works at runtime. One markdown file
per subsystem, each following the template at `_template.md`.

## Why these exist

The spec (`../FuseSimple_v1_Spec.md`) says **what** FuseSimple does. The decisions
doc (`../FuseSimple_v1_Decisions.md`) says **how** we chose to implement it. These
flow docs say **what actually runs, in what order, with what state**.

They're the document I wish I had when debugging someone else's library at 11pm.

## Rules

1. **Written alongside the code they describe.** A flow doc is updated in the
   same commit as the behavior it documents. Never "flow docs pass" as a
   separate cleanup task.
2. **Living, not historical.** If the flow changes, the doc changes. Don't
   preserve old behavior in the body — the git history and CHANGELOG are the
   archive.
3. **Concrete over abstract.** Link to file paths and line-ish references when
   useful. Show the actual function names a reader can grep for.
4. **Include the debug tags.** Every flow doc lists which `debugLogger` tags
   cover which steps. A reader hunting a bug should be able to open the flow
   doc, see `?debug=HEADER,SYNC`, and know exactly what to light up.

## Index

| File | Subsystem | Status |
|---|---|---|
| `_template.md` | Template for new flow docs | -- |
| `basemap-lifecycle.md` | PMTiles protocol, style resolution, MapLibre mount, LOD application, sync startup, teardown | Current (0.13.0) |
| `click-passthrough.md` | hitTest filter → `queryRenderedFeatures` → event emission | Current (0.13.0) |
| `debug-logger.md` | URL-activated debug logger, lockdown, multi-instance isolation | Current (0.13.0) |
| `loading-splash.md` | Splash overlay mount, idle dismissal, timeout safeguard, teardown | Current (0.14.0) |
| `web-component-ready.md` | `<arcgis-map>` element detection, `viewOnReady()` race-free await, timeout and error paths | Current (0.14.1) |
| `adapter-contracts.md` | `BasemapAdapter` + `OperationalAdapter` interfaces, state ownership, destroy symmetry | Current (0.13.1) |
| `view-sync.md` | Esri watchers → rAF coalesce → MapLibre `jumpTo`, sync guard, resize path | Covered in `basemap-lifecycle.md` §11-13 |
| `create-lifecycle.md` | `FuseSimple.create()` from call to returned instance | Covered in `basemap-lifecycle.md` (the full create flow) |
| `destroy.md` | Teardown ordering and listener cleanup | Covered in `basemap-lifecycle.md` |

Entries are added as the corresponding subsystem lands. If it's not in the index,
it hasn't been written yet.
