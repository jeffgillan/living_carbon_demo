# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Biomass Explorer — a client-side web app for visualizing biomass raster data on an interactive map and calculating zonal statistics within user-defined polygons. Hosted on GitHub Pages. No backend, no build step, no node_modules.

## Architecture

- **`index.html`** — Landing page linking to the map app
- **`map.html`** — The entire application in a single HTML file (markup, CSS, and JS inline)
- **`biomass-app-spec.md`** — Original application specification

All third-party libraries are loaded from CDNs at runtime (MapLibre GL JS, Mapbox GL Draw, geotiff.js, Turf.js). There is no package manager or bundler.

## Local Development

Must be served via HTTP (not `file://`) due to CORS:

```bash
python3 -m http.server 8000
# Open http://localhost:8000
```

## Deployment

Push to `main` branch — GitHub Pages serves from root `/` automatically. No build step.

```bash
git push origin main
```

Live at: `https://jeffgillan.github.io/living_carbon_demo/`

## Key Technical Details

- **COG data**: Biomass raster is a Cloud Optimized GeoTIFF on CyVerse, accessed via HTTP Range requests through geotiff.js. URL configured as `COG_URL` constant at top of `map.html` script.
- **Configuration constants** are grouped at the top of the `<script>` block in `map.html`: `COG_URL`, `NODATA_VALUE`, `MAP_CENTER`, `MAP_ZOOM`.
- **COG must be EPSG:4326** to align with GeoJSON and the web map.
- **Viewport-based dynamic tiling**: The biomass overlay selects the appropriate COG overview level based on zoom and re-renders on `moveend` (debounced 200ms). Canvas max width capped at 1024px.
- **Zonal calculation**: Reads full-resolution pixels within polygon bbox, then does per-pixel point-in-polygon test with Turf.js. Can be slow for large polygons.
- **Single polygon at a time**: Drawing or uploading replaces the previous polygon.
- **MapLibre/Mapbox Draw compatibility**: Draw constants are patched at load time to use `maplibregl-` CSS class prefixes.
