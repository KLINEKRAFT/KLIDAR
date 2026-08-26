# KLIDARKRAFT — Roadmap

Ordered by what unblocks the most. Each item notes rough effort and why it matters.

---

## Step 0 — Get it into a repo (do this first)

```
klidarkraft/
  index.html
  CLAUDE.md
  README.md
  ROADMAP.md
  vercel.json          # optional, static needs no config
```

Drop `klidarkraft-v07.html` in as `index.html` and deploy. It is fully static —
Vercel needs no build step. Confirm it works on the phone before refactoring
anything.

---

## Phase 1 — Structure (effort: half a day)

**Split the single file into modules.** ~1,700 lines in one file is fine for a
sketch and bad for agentic editing. Claude Code works far better against small
focused files.

Suggested split, Vite as the bundler:

```
src/
  main.js               boot + loop
  state.js              shared mutable state
  terrain/dem.js        installDEM, fillHoles, smoothGrid, packTex
  terrain/readers.js    parseASC, parseTIFF
  terrain/analysis.js   blur, gradients, hillshadeAt, components
  terrain/scan.js       SCALES, analyse, classify, score, runScan
  render/points.js      point cloud material + geometry
  render/surface.js     surface material + geometry
  render/edl.js         EDL pass
  render/camera.js      orbit + focus
  ui/sheet.js           glass sheet, resize, snap
  ui/markers.js         projection + detail card
  ui/plate.js           legend, compass, scale bar
  style.css
```

Keep `index.html` thin. Move three.js to an npm dependency so the version is
pinned in `package.json` instead of a CDN URL.

---

## Phase 2 — Georeferencing (effort: one day) — **highest value**

Right now candidates are reported in local metres from tile centre. That is
useless for actually going to look at one.

1. Keep the CRS and bounding box from the GeoTIFF in `installDEM`. The reader
   already computes cell size from the bounding box and now preserves the raster's
   true `GW × GH`, so the tie point plus `CELL` is enough to place any cell.
2. Add `proj4js`, convert candidate centroids to WGS84 lat/lon.
3. Show lat/lon in the detail card, with a copy button.
4. Add "Open in Maps" — Apple Maps and Google Maps deep links.
5. Export candidates as **GeoJSON** and **KML** so they open in QGIS,
   Google Earth, or a phone GPS app.

Export must include the confidence, evidence, alternative explanation, and the
"not a confirmed site" notice as properties. Keep exports local — download only,
no upload.

---

## Phase 3 — Real point clouds (local files done, streaming still open)

- ~~Add `laz-perf` (WASM decode)~~ — done, lazy-loaded like geotiff.js.
- ~~Load a `.las` / `.laz` from a local file~~ — done, via the file picker. No
  CORS problem, because nothing is fetched.
- ~~Wire real LAS classification codes: 2 ground, 5 high vegetation, 6
  building~~ — done; the three-class chip UI maps directly.
- ~~Grid ground returns into the DEM~~ — done, lowest class 2 return per cell.
- **Still open: streaming.** The whole file goes into the WASM heap, so peak
  memory is about twice the file size and a 10M point tile takes ~18 s. For
  tiles beyond ~30M points this needs `copc.js` and octree nodes over HTTP range
  requests, loading only what the camera needs. Microsoft Planetary Computer
  serves USGS 3DEP as COPC with CORS and is the target for that.
- **Still open: level of detail.** Display is capped at 1.2M points by striding.
  `c42f/displaz` has the right idea — fit observed frame time against vertex
  count and derive a quality factor, rather than a fixed cap.

---

## Phase 4 — Better detection (mostly done)

- ~~**Sky-view factor**~~ — done. 16 azimuths, 28 m search radius, rendered with
  a 2–98% percentile stretch.
- ~~**Positive / negative openness**~~ — done, rendered as the difference
  (`PO − NO`), symmetric about zero so mid-ramp is flat ground.
- ~~Feed these into the confidence score as independent indicators~~ — done, as
  contrast against a surrounding ring, worth 10 and 7 points, with a −12
  counter-indication when openness contradicts the claimed form.
- ~~**Multi-hillshade blend**~~ — done, as the Mark 1992 / Tait 2010 weighting
  GDAL uses for `-multidirectional`: four azimuths combined with a sin² weight
  against the slope aspect, so no single sun angle stripes the image.
- ~~**VAT composite**~~ — done. Sky-view base, positive openness overlaid,
  darkened by slope; the Relief Visualization Toolbox archaeological default,
  and greyscale, so it suits the house style. Gamma 1.35 is a tunable.

- ~~**MSTP**~~ — done. `DEV(σ) = (z − mean_σ)/std_σ` over three scale bands into
  R/G/B, clipped at ±2.2σ. Colour encodes the dominant scale of a structure, so
  a mound reads green-cyan against blue terraces.
- ~~**RRIM**~~ — done. Redness from slope, lightness from differential openness.
  Free, in the sense that both inputs already existed.

Still worth taking from `nico579/lidar2map`: a **line-sweep horizon kernel**
instead of per-cell ray marching. `runScan` is the bottleneck rather than
`horizonScan`, so this is not urgent.

~~Move `horizonScan` and `runScan` into a **Web Worker** with progress
messages.~~ Done. Everything expensive — gradients, blur, horizon products,
local relief, MSTP and the feature scan — runs in a Blob worker built by
stringifying `scanWorkerBody`, so there is still one file and one copy of the
analysis. Progress is driven by worker events rather than a timer. On a real
500² tile the longest main-thread stall during a scan fell from 718 ms to
59 ms, matching idle, and wall time went 4.5 s to 2.8 s.

**Phase 4 is complete.**

---

## Phase 5 — Working sessions (effort: one day)

- Persist loaded DEM metadata and scan results to IndexedDB.
- Named projects: "Verdigris terrace 1", "Spiro north".
- Mark a candidate as reviewed / dismissed / worth a visit, with a note field.
- Re-run scan with adjusted thresholds without reloading the DEM.

---

## Phase 6 — Scale and polish

- Multi-tile stitching for AOIs larger than one raster.
- Level-of-detail on the surface mesh so you can load a full 1 km² at 1 m.
- Compare mode: swipe between hillshade and NAIP aerial imagery to rule out
  modern features. This would kill most false positives from farm terraces.
- PWA manifest and offline caching — useful in the field with no signal.
- Screenshot / report export for a single candidate.

---

## Things deliberately not on this list

- No public database of candidate locations. Site locations are legally
  confidential in Oklahoma.
- No "confirmed site" labelling of any kind.
- No auto-posting or sharing of coordinates.

---

## Test data

- **OpenTopography** — portal.opentopography.org → USGS 3DEP → draw a box →
  GeoTIFF, 1 m, bare earth. Fastest route to real ground.
- **Microsoft Planetary Computer** — 3DEP as COPC, CORS enabled, range requests.
- **USGS rockyweb** — raw `.laz` for `OK_Statewide_D22`. Download once, then
  `pdal translate in.laz ground.copc.laz -f range --filters.range.limits="Classification[2:2]"`
  and self-host.

Good hunting ground: terraces above the Arkansas, Verdigris and Grand rivers in
eastern Oklahoma.
