# KLIDARKRAFT

Browser-based LiDAR terrain viewer and archaeological screening tool.
Built to find possible Native American mounds and earthworks in Oklahoma
bare-earth elevation data.

Personal project under KLINEKRAFT. Static site, deployed on Vercel.

---

## What it does

1. Renders a bare-earth DEM as a shaded 3D surface, or a classified point
   cloud, or both.
2. Gives the operator the controls that actually reveal subtle earthworks:
   movable sun, vertical exaggeration, contour banding, local relief model,
   slope shading, elevation clipping.
3. Renders **sky-view factor** and **openness** — illumination-independent
   relief products that show shallow earthworks a hillshade buries.
4. Runs **Terrain Scan** — a real geomorphometry pipeline that detects,
   measures, scores and ranks possible anthropogenic terrain features, then
   marks them on the map with the evidence that flagged them.

Current build: `index.html` — one self-contained file, no build step,
three.js r128 from cdnjs, geotiff.js lazy-loaded only when a `.tif` is opened.

---

## Architecture of the current single file

Read top to bottom; it is organised in these blocks.

| Block | What lives there |
|---|---|
| Palettes | `PAL` ramps + `CSETS` class colors, both keyed by `dark`/`light`. 256-entry LUTs precomputed into `LUTS`. |
| Demo point cloud | Synthetic 100k-point tile with 5 planted mounds and 1 linear earthwork. Only dataset that has points. |
| Mutable terrain grid | `GW, GH, CELL, SPANX, SPANZ, SPAN, Zm, Hn, gx, gz, slope, zMin, zMax, zRange`. All rebuilt by `installDEM()`. |
| three.js setup | `scene` (points + surface), `overlay` (wireframe box + separation frames), `quadScene` (EDL composite). |
| `installDEM(dem)` | The single entry point for new terrain. Downsamples, normalises, computes gradients, packs the height texture, rebuilds surface geometry and bounding box, resets scan results, reframes the camera. |
| Horizon products | `HDIRS`, `HRADIUS`, `horizonScan`, `needHorizon` → `SVF`, `OPN`. Built on demand, invalidated by `installDEM`. |
| Scan | `SCALES`, `blur`, `components`, `analyse`, `classify`, `score`, `altFor`, `runScan`. |
| DEM readers | `parseASC` (zero-dependency), `parseTIFF` (lazy geotiff.js). |
| Display plumbing | `recolor`, `paintRamp`, `paintLegend`, `applyVex`, `updateSun`, `setTheme`. |
| UI | Glass sheet with drag-resize + snap, candidate detail sheet, marker projection. Above 900px the same sheet becomes a collapsible left rail — one open/close state, only the axis changes. `--rail`, `--plateL` and `--sheet` keep the plate readouts clear of it. |
| Loop | `tick()` — two render paths depending on mode. |

### Data conventions

- **World units are metres.** 1 three.js unit = 1 m. Feet are display-only
  (`M2FT`).
- **The grid is rectangular.** `GW × GH`, indexed `d = j*GW + i`, where `i` → world
  x (east), `j` → world z. Row 0 of a GeoTIFF/ASC is north, and north is `-z`.
  The compass depends on this. Never pad a raster to a square — real exports from
  OpenTopography almost never are, and padding fabricates terrain the scan will
  then dutifully analyse.
- **Half extents are `SPANX` and `SPANZ`**; `SPAN` is the larger of the two and is
  only for camera framing, zoom clamps and far plane. Anything measuring ground
  area must use `areaM2()`, not `SPAN`.
- `Zm` is elevation in metres **relative to `zMin`**. `Hn` is the same
  normalised 0..1. Shaders use `Hn`; analysis uses `Zm`.
- Surface mesh y is **absolute**: `zMin + Hn*zRange*vex`. Points use
  `uBase + (y-uBase)*uVex` with `uBase = pMin`. These must stay aligned or
  Both mode desyncs.

### Rendering

Two paths, chosen in `tick()`:

- **Points only** — render to `rt` with color in RGB and log-depth in alpha
  (`NoBlending`), then a fullscreen **EDL** pass (eye-dome lighting, 8 neighbour
  taps) composites to screen, then overlay lines with `clearDepth()`.
- **Surface / Both** — direct forward render, then overlay.

EDL is deliberately points-only. It is meaningless on a solid mesh, and mixing
the two would require a depth copy that is not worth it.

The surface shader computes its normal in the **fragment** stage from the packed
height texture, so exaggeration and lighting stay live without recomputing
vertex normals.

---

## Gotchas that will bite you

- **three.js r128** is pinned. `THREE.OrbitControls` is not bundled — the orbit
  camera is hand-rolled from pointer events. `CapsuleGeometry` does not exist.
- **Height texture is packed into RGBA8** (16-bit across R and G) and sampled
  with `NearestFilter`. Linear filtering corrupts packed channels — it would
  blend the high and low bytes independently. Do not change the filter. The
  fragment shader's `H()` instead decodes four texels and mixes them itself,
  with smoothstep weights so the reconstruction is C1 and the normal (a central
  difference of it) stays continuous across cell boundaries. Sampling the raw
  texture directly makes every cell shade as a flat block.
- **Projected GeoTIFFs are often in feet.** USGS OPR tiles for Oklahoma are
  State Plane ftUS — `NAD83(2011) / Oklahoma South (ftUS) + NAVD88 height
  (ftUS)` — and carry no `ProjLinearUnitsGeoKey`, because EPSG 6555 implies the
  unit. `tiffUnits()` falls back to the CRS citation strings. Reading those as
  metres inflates every distance and elevation by 3.28x, which also silently
  moves the scan's metre-based thresholds. Horizontal and vertical units are
  resolved separately, and the detected unit is shown in the source panel.
- **Uint16 index limit.** The surface mesh needs 32-bit indices once
  `GW*GH > 65536`. `MAXG` caps either dimension at 512 when
  `OES_element_index_uint`/WebGL2 is available, else 255; `installDEM` also
  halves until `GW*GH` fits when 32-bit indices are unavailable.
- **rockyweb.usgs.gov sends no CORS headers.** Raw `.laz` tiles cannot be fetched
  from the browser. Either convert with PDAL and self-host, or use a
  CORS-enabled source.
- **The scan blocks the main thread.** On a real 500² tile the horizon scan is
  ~590 ms and `runScan` ~4 s, both behind the progress overlay. The feature scan
  is the bottleneck, not the horizon products. Moving both to a Worker is the
  remaining Phase 4 item.
- **Horizon rays step at 1.35x growth, not every cell.** Near cells carry the
  angular detail; far ones barely move the horizon. Measured against a
  brute-force sweep on real data: 1.55x faster, SVF RMS error 0.0008 and
  openness RMS 0.07° against a ±20° range.
- **Sky-view and openness need a percentile stretch, not min/max.** One incised
  channel pins the low end and squeezes every subtle feature into the top few
  percent of the ramp. `pctRange` does 2–98%; do not replace it with min/max.
- **Sky-view and openness are rendered unlit on purpose.** Modulating them by a
  hillshade would put back exactly the directional bias they exist to remove.
- **Marker positions are DOM elements** projected each frame. Cheap at ~14, will
  not scale to hundreds.
- **`segBind` / `segSet`** drive every segmented control. Buttons carry the value
  in a `data-` attribute; keep that pattern.
- **The compass shows a bearing, not a projected ground vector.** North sits at
  `90° − yaw` clockwise from screen-up, because screen-right on the ground is
  `(sin yaw, −cos yaw)`. In oblique views the drawn needle will not line up with
  the foreshortened tile edge; that is correct and unavoidable for a flat dial.
- **Points mode colours by `pElev`**, normalised over the whole point cloud
  (vegetation and structures included) — not over the bare-earth DEM. The legend
  must label `pMin`/`pMax` in that mode, not `zMin`/`zMax`.

---

## Detection: how scoring actually works

Never label anything "Confirmed". Score caps at 96. Bands: ≥70 High, ≥45
Moderate, else Low.

**Adds** (base 6, weights in points): circularity 16, radial symmetry 11, local
relief 13 (saturating at 1.2 m), summit flatness 6, edge sharpness 6, hillshade
persistence across 8 azimuths 4 (cubic in `dirs/8`), **openness contrast against
the surrounding ring 10** (saturating at 5°), **sky-view contrast 7** (saturating
at 0.06), multi-scale persistence 6 per extra scale to a maximum of 12,
surrounding depression 7, clustering with similar neighbours 1.5 each to a
maximum of 5.

`dirs` comes back 8/8 for almost any relief, so hillshade persistence is worth
little on its own. The illumination-independent pair carries that weight
instead — that is what "detectable across multiple representations" has to mean
in practice. Both contrasts are multiplied by `sign`, so a positive value means
"shaped the way this candidate claims to be" for mounds and ditches alike.

Keep the constant floor small. An earlier build opened at base 28 and added a
near-constant 16 for hillshade persistence plus 12 for clustering — 56 points
before any evidence — which put every candidate in the High band and made the
bands meaningless. Clustering only counts neighbours of the same sign and within
2.5× the relief, so a mound cannot cluster with a ditch.

**Subtracts:** long axis following local downslope (−24), steep natural terrain
(−14), straight and asymmetric i.e. road/levee/field boundary (−14), visible
from ≤2 illumination directions (−12), openness contradicting the claimed form
(−12), weak relief (−10).

Every candidate must carry an **Alternative Explanation**. Every candidate must
carry its counter-indications. This is a screening tool, not an identifier.

---

## Ethics and law — non-negotiable

- Archaeological site locations in Oklahoma are **legally confidential**.
  Do not build features that publish candidate coordinates publicly, and do not
  add a public shared database of hits.
- Disturbing sites is illegal under state law, ARPA and NAGPRA.
- Any export or share feature should carry the confidentiality notice and point
  toward the **Oklahoma Archeological Survey** at OU as the reporting channel.

---

## Style

- Black and white chrome only. No red anywhere in the UI. No emojis.
- Helvetica Neue for display, IBM Plex Mono for all data and labels.
- Hairline rules, zero border radius, heavy letterspacing on small caps.
- Light and dark are both first-class; every color must be defined for both.
- KLINEKRAFT wordmark in the footer.
- iPhone first. 44px minimum touch targets, safe-area insets respected.
- Desktop is a second layout, not a second design: the bottom sheet becomes a
  left rail at 900px. Do not fork the markup for it.
