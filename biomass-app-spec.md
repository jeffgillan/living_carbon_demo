# Biomass Mapping Web Application — Project Specification

## Overview

Build a lightweight, statically-hosted web application that displays an interactive map with aerial imagery, allows users to visualize a biomass raster layer, and calculate total biomass within a user-defined area of interest. The app runs entirely client-side with no backend — all spatial analysis happens in the browser.

The app will be hosted on GitHub Pages.

---

## Repository Structure

```
repo/
├── index.html      # Landing page
├── map.html        # Interactive map application
└── README.md
```

The biomass COG (Cloud Optimized GeoTIFF) is hosted externally on CyVerse cloud storage and referenced by URL. It is NOT stored in the repo.

---

## Libraries (All via CDN — No Build Step)

All libraries are loaded via `<script>` and `<link>` tags in the HTML files. No npm, no bundler, no build process.

| Library | Purpose | CDN |
|---|---|---|
| **MapLibre GL JS** | Map rendering, basemap, layer display | `https://unpkg.com/maplibre-gl/dist/maplibre-gl.js` and `maplibre-gl.css` |
| **MapLibre GL Draw** | Polygon drawing tools on the map | `https://unpkg.com/@mapbox/mapbox-gl-draw/dist/mapbox-gl-draw.js` and `mapbox-gl-draw.css` (compatible with MapLibre) |
| **geotiff.js** | Reading pixel values from the Cloud Optimized GeoTIFF | `https://unpkg.com/geotiff` |
| **Turf.js** | Spatial operations (bounding box, point-in-polygon) | `https://unpkg.com/@turf/turf` |

---

## Page 1: Landing Page (`index.html`)

A simple, clean landing page with:

- **Title**: "Biomass Explorer" (or similar)
- **Short description**: 1–2 sentences explaining the tool, e.g., "Upload a polygon or draw an area of interest to calculate total biomass from remotely sensed data."
- **"Open Map" button**: Links to `map.html`
- **Minimal styling**: Clean, centered layout. No frameworks needed — just basic CSS in a `<style>` block. White or light background, readable font (system font stack is fine).

---

## Page 2: Interactive Map (`map.html`)

### Layout

- **Map**: Full-screen (`width: 100vw; height: 100vh`)
- **Floating control panel**: Positioned in the top-right corner, semi-transparent white background, rounded corners, ~300px wide. Contains:
  1. Layer toggle (checkbox) for the biomass raster
  2. File upload button for GeoJSON
  3. Results display area (hidden until analysis runs)
  4. A "Clear" button to reset the polygon and results
- **Draw toolbar**: Provided by MapLibre GL Draw plugin, appears on the map (default position is fine)

### Basemap

Use ESRI World Imagery as the basemap. This is a freely available raster tile service that does not require an API key.

Tile URL template:
```
https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}
```

Configure this as a MapLibre raster source and layer. Set the map's initial view to center on the area covered by the biomass COG (the exact center coordinates and zoom level will be determined by the COG's extent — use a reasonable default for a forested area and adjust once the COG URL is available).

### Biomass COG Layer

- **Default state**: Off (not visible when map loads)
- **Toggle**: A checkbox in the control panel labeled "Show Biomass Layer"
- **Implementation**: Use geotiff.js to read the COG and render it as a canvas overlay or as a MapLibre image source. The COG URL will be provided as a configurable variable at the top of the script (placeholder for now):
  ```javascript
  const COG_URL = 'https://your-cyverse-url/path/to/biomass.tif';
  ```
- **Visualization**: Render biomass values with a color ramp (e.g., green gradient — low biomass = light green, high biomass = dark green). Include a simple legend on the map or in the control panel showing the color scale.
- **Important**: The COG must be in **EPSG:4326** (WGS84 geographic coordinates) for simplicity, since GeoJSON uses lat/lon by default. If reprojection is needed, note it but keep the app expecting 4326.

### Polygon Input — Two Methods

#### Method 1: GeoJSON File Upload

- File input (`<input type="file" accept=".geojson,.json">`) in the control panel
- When a file is selected, read it with the FileReader API, parse as JSON
- Validate that it contains polygon geometry
- Add the polygon to the map as a MapLibre GeoJSON source/layer with a visible outline and semi-transparent fill
- Zoom/fit the map to the polygon extent
- Automatically trigger the biomass calculation

#### Method 2: Draw Polygon on Map

- MapLibre GL Draw plugin provides draw tools (polygon, edit, delete)
- When the user finishes drawing a polygon (listen for the `draw.create` event), extract the GeoJSON geometry
- Trigger the biomass calculation with the drawn polygon
- Also handle `draw.update` and `draw.delete` events to recalculate or clear results

**Only one polygon at a time.** If the user uploads a new GeoJSON or draws a new polygon, clear the previous one and its results.

### Biomass Calculation — Zonal Sum

This is the core analysis logic. It runs entirely in the browser.

**Step-by-step process:**

1. **Get the polygon geometry** (from upload or draw)
2. **Compute the bounding box** of the polygon using `turf.bbox(polygon)`
3. **Read COG pixels within the bounding box** using geotiff.js:
   - Open the COG with `GeoTIFF.fromUrl(COG_URL)`
   - Get the image with `tiff.getImage()`
   - Read the image's geo metadata (origin, resolution, dimensions) from the TIFF tags
   - Convert the polygon's bounding box from geographic coordinates to pixel coordinates
   - Use `image.readRasters({ window: [x0, y0, x1, y1] })` to read only the pixels within the bounding box
4. **For each pixel in the bounding box:**
   - Compute the pixel's geographic center coordinate
   - Use `turf.booleanPointInPolygon(point, polygon)` to check if it falls inside the polygon
   - If yes AND the pixel value is not NoData, add the value to the running sum
5. **Display the result** in the control panel:
   - Total biomass sum (with appropriate units label — placeholder until units are confirmed)
   - Optionally: number of pixels summed, polygon area (via `turf.area(polygon)`)

**Performance note:** For large polygons with many pixels, the point-in-polygon check for every pixel can be slow. For the prototype, this straightforward approach is fine. If performance becomes an issue, consider sampling or using a web worker.

**NoData handling:** The COG will have a NoData value (commonly -9999, 0, or NaN). Check for this and skip those pixels. The NoData value should be configurable:
```javascript
const NODATA_VALUE = -9999; // adjust based on actual COG
```

### Results Display

In the control panel, show:
- **"Total Biomass: X"** (formatted number with commas, units TBD)
- A loading indicator (simple "Calculating..." text) while the analysis runs
- The results section is hidden until an analysis is performed

### Clear/Reset

A "Clear" button in the control panel that:
- Removes the polygon from the map
- Clears the drawn polygon from MapLibre GL Draw
- Hides the results
- Resets the file input

---

## Styling

Keep it minimal and clean:

- **Font**: System font stack (`-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif`)
- **Control panel**: White background at ~90% opacity, `border-radius: 8px`, subtle box shadow, 16px padding
- **Polygon style**: Blue outline (`#3388ff`), semi-transparent blue fill
- **Buttons**: Simple styled buttons, nothing fancy
- **No CSS framework** — just a `<style>` block in each HTML file

---

## Configuration Variables

Group these at the top of the `<script>` section in `map.html` for easy adjustment:

```javascript
const COG_URL = 'https://your-cyverse-url/path/to/biomass.tif';  // CyVerse COG URL
const NODATA_VALUE = -9999;       // NoData value in the COG
const MAP_CENTER = [-111.0, 32.2]; // Initial map center [lng, lat] — adjust to COG extent
const MAP_ZOOM = 10;               // Initial zoom level
```

---

## Key Technical Constraints

1. **Entirely client-side** — no server, no API endpoints, no backend processing
2. **No build step** — no npm, webpack, vite, etc. Just raw HTML/JS/CSS files
3. **COG must support HTTP range requests** — CyVerse cloud storage should support this; verify CORS headers allow browser access
4. **CRS alignment** — COG and GeoJSON must be in the same coordinate reference system (EPSG:4326 preferred)
5. **CORS** — The COG URL must be accessible from the browser (CyVerse may need CORS headers configured to allow cross-origin requests from the GitHub Pages domain)

---

## Future Enhancements (Not in Prototype)

- Additional summary statistics (mean, min, max, std dev)
- Multiple polygon support
- Export results as CSV
- Time series biomass comparison
- Pre-loaded study area boundaries
- Responsive mobile layout
