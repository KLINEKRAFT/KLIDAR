# KLIDARKRAFT

Browser-based LiDAR terrain viewer and archaeological screening tool. Renders
bare-earth elevation data as a shaded 3D surface or a classified point cloud, and
runs a geomorphometry pass that measures, scores and ranks possible anthropogenic
terrain features.

Static site, no build step. Open `index.html`, or deploy the directory as-is.

## Using it

Open **Terrain Scan** to analyse the loaded model, or **Open DEM file** to load
your own. Reads GeoTIFF (`.tif`), ESRI ASCII grid (`.asc`), and classified point
clouds (`.las` / `.laz`).

Free 1 m bare-earth data: portal.opentopography.org → USGS 3DEP → draw a box →
GeoTIFF. Point clouds work straight from the USGS download — no PDAL step.

Rasters do not need to be square, NODATA voids are interpolated from surrounding
valid cells, and files in State Plane feet are converted to metres from their
CRS rather than assumed. Point clouds are gridded from ground (class 2) returns
only, so vegetation and buildings never enter the terrain model.

## What it is not

A screening tool, not an identifier. Nothing it marks is a confirmed
archaeological site; every candidate carries its counter-indications and an
alternative natural or modern explanation, and needs visual or professional
review.

Archaeological site locations in Oklahoma are legally confidential. Disturbing
sites is illegal under state law, ARPA and NAGPRA. Report possible sites to the
Oklahoma Archeological Survey at OU.

See `CLAUDE.md` for architecture and `ROADMAP.md` for planned work.
