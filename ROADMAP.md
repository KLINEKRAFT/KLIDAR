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

## Phase 3 — Real point clouds via COPC (effort: one to two days)

The point cloud path currently only has synthetic demo data.

- Add `laz-perf` (WASM decode) and `copc.js` (octree over HTTP range requests).
- Load a `.copc.laz` from a URL or a local file.
- Stream by octree node, respecting the current camera — do not load the whole
  tile.
- Wire real LAS classification codes: 2 ground, 5 high vegetation, 6 building.
  The existing three-class chip UI maps directly.
- Grid ground returns into the DEM using the code already in `demoDEM()`.

**Source:** Microsoft Planetary Computer serves USGS 3DEP as COPC with CORS.
That is the target. USGS rockyweb is not fetchable from a browser.

---

## Phase 4 — Better detection (effort: one to two days)

The original spec listed visualizations not yet implemented. These are the ones
archaeologists actually rely on:

- **Sky-view factor** — the single best product for shallow earthworks. Sample
  horizon angle in 16 directions per cell.
- **Positive / negative openness** — closely related, excellent for ditches and
  banks.
- **Multi-hillshade blend** — render 16 azimuths and combine, either PCA or
  simple max. Removes the directional bias entirely.
- Feed all of these into the confidence score as additional independent
  indicators, per the "should remain detectable across multiple representations"
  rule.

Also move `runScan` into a **Web Worker** with progress messages. The staged
progress UI already exists; it just needs real events instead of timed stages.

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
