# Biomass Explorer

A lightweight, client-side web application for visualizing biomass raster data on an interactive map and calculating zonal statistics within user-defined areas of interest. The entire application runs in the browser — no backend server required.

**[Live Demo](https://jeffgillan.github.io/living_carbon_demo/)**

---

## Application Architecture

```mermaid
flowchart TB
    subgraph GitHub["GitHub Pages"]
        index["index.html<br/><i>Landing Page</i>"]
        maphtml["map.html<br/><i>Interactive Map App</i>"]
    end

    subgraph CDN["CDN Providers"]
        unpkg["unpkg.com"]
        jsdelivr["jsdelivr.net"]
    end

    subgraph CyVerse["CyVerse Data Store"]
        cog["Biomass COG<br/><i>Cloud Optimized GeoTIFF</i>"]
    end

    subgraph Browser["User's Browser"]
        maplib["MapLibre GL JS<br/><i>Map rendering</i>"]
        draw["Mapbox GL Draw<br/><i>Polygon tools</i>"]
        geotiff["geotiff.js<br/><i>COG pixel reader</i>"]
        turfjs["Turf.js<br/><i>Spatial analysis</i>"]
    end

    GitHub -->|"serves static HTML"| Browser
    CDN -->|"delivers JS libraries"| Browser
    geotiff -->|"HTTP Range requests"| cog
```

---

## Repository Structure

```
living_carbon_demo/
├── index.html          # Landing page with app description
├── map.html            # Interactive map application (single-file)
├── biomass-app-spec.md # Application specification
├── README.md
└── LICENSE
```

The map application is a **single HTML file** (`map.html`) containing all markup, styles, and JavaScript. There is no build step, no bundler, and no `node_modules` — all third-party libraries are loaded at runtime from CDNs.

---

## Technology Stack

All libraries are loaded via `<script>` and `<link>` tags directly from Content Delivery Networks (CDNs). This means:

- **No installation required** — no `npm install`, no package manager
- **No build step** — no webpack, vite, or bundler
- **Instant setup** — clone the repo and serve the HTML files

| Library | Version | CDN | Purpose |
|---------|---------|-----|---------|
| [MapLibre GL JS](https://maplibre.org/) | 5.1.0 | `unpkg.com/maplibre-gl` | Map rendering, ESRI World Imagery basemap, layer management |
| [Mapbox GL Draw](https://github.com/mapbox/mapbox-gl-draw) | 1.5.0 | `unpkg.com/@mapbox/mapbox-gl-draw` | Polygon drawing and editing tools on the map |
| [geotiff.js](https://geotiffjs.github.io/) | 2.1.3 | `cdn.jsdelivr.net/npm/geotiff` | Reading Cloud Optimized GeoTIFF pixel data via HTTP range requests |
| [Turf.js](https://turfjs.org/) | 7.x | `unpkg.com/@turf/turf` | Spatial operations: bounding box, point-in-polygon, area calculation |

---

## How It Works

```mermaid
sequenceDiagram
    actor User
    participant Pages as GitHub Pages
    participant Browser
    participant CDN as CDN (unpkg / jsdelivr)
    participant CyVerse as CyVerse Data Store

    User->>Pages: Visit site
    Pages->>Browser: Serve index.html
    User->>Pages: Click "Open Map"
    Pages->>Browser: Serve map.html
    Browser->>CDN: Load MapLibre, Draw, geotiff.js, Turf.js
    CDN-->>Browser: JavaScript libraries

    Note over User,Browser: Toggle Biomass Layer
    Browser->>CyVerse: HTTP Range request (COG overview bytes)
    CyVerse-->>Browser: Overview raster data
    Browser->>Browser: Render to canvas with color ramp
    Browser->>Browser: Display as map overlay

    Note over User,Browser: Draw or Upload Polygon
    Browser->>CyVerse: HTTP Range request (pixels within polygon bbox)
    CyVerse-->>Browser: Full-resolution pixel data
    Browser->>Browser: Point-in-polygon test per pixel (Turf.js)
    Browser->>Browser: Sum valid pixel values
    Browser-->>User: Display total biomass, pixel count, area
```

### Key Steps

1. **Map loads** with ESRI World Imagery basemap centered on the study area
2. **Biomass layer toggle** reads a low-resolution overview from the COG and renders it as a colored overlay
3. **User defines an area** by drawing a polygon on the map or uploading a GeoJSON file
4. **Zonal calculation** reads only the full-resolution pixels within the polygon's bounding box, tests each pixel center for containment, and sums the valid values
5. **Results display** shows total biomass, number of pixels summed, and polygon area in hectares

---

## Biomass Data Layer

The biomass raster is stored as a **Cloud Optimized GeoTIFF (COG)** on the [CyVerse](https://cyverse.org/) Data Store with public anonymous access.

### What is a COG?

A Cloud Optimized GeoTIFF is a regular GeoTIFF organized so that clients can request just the portions they need via **HTTP Range requests** instead of downloading the entire file. Key features:

- **Internal tiling** — pixels are stored in tiles rather than strips, enabling spatial queries
- **Overview pyramids** — pre-computed lower-resolution copies for fast visualization at different zoom levels
- **Byte-range accessible** — standard HTTP servers can serve partial file reads without special software

### How the App Streams COG Data

```mermaid
flowchart LR
    subgraph COG["Cloud Optimized GeoTIFF"]
        direction TB
        full["Full Resolution<br/>29,742 × 26,734 px"]
        ov1["Overview 1<br/>14,871 × 13,367 px"]
        ov2["Overview 2<br/>7,435 × 6,683 px"]
        ov3["Overview 3<br/>3,717 × 3,341 px"]
        ov4["Overview 4<br/>1,858 × 1,670 px"]
        ov5["Overview 5<br/>929 × 835 px"]
        ov6["Overview 6<br/>464 × 417 px"]
    end

    viz["Visualization<br/><i>Toggle biomass layer</i>"]
    calc["Calculation<br/><i>Zonal sum in polygon</i>"]

    ov6 -->|"Range request<br/>small overview"| viz
    full -->|"Range request<br/>windowed read of bbox"| calc
```

- **For visualization**: The app reads the smallest overview pyramid (e.g., 464 × 417 px) — only a few hundred KB of data — and renders it to an HTML canvas with a green color ramp
- **For calculation**: The app reads only the full-resolution pixels within the polygon's bounding box using `image.readRasters({ window })` — avoiding loading the entire raster into memory

The COG is accessed using `geotiff.js`, which issues standard HTTP `GET` requests with `Range` headers to the CyVerse `dav-anon` endpoint. No special server-side software is needed.

**Current COG URL:**
```
https://data.cyverse.org/dav-anon/iplant/home/jgillan/living_carbon_demo/se_arizona_dem.tif
```

The COG must be in **EPSG:4326** (WGS84 geographic coordinates) to align with GeoJSON polygons and the web map.

---

## Local Development

Since the app fetches remote data via HTTP, it must be served from a local web server (not opened directly as a `file://` path due to browser CORS restrictions).

```bash
# Clone the repository
git clone https://github.com/jeffgillan/living_carbon_demo.git
cd living_carbon_demo

# Start a local server
python3 -m http.server 8000

# Open in browser
# http://localhost:8000
```

Press `Ctrl+C` to stop the server.

---

## Deployment

The app is hosted as a **static website on GitHub Pages**:

- **Source**: `main` branch, root directory (`/`)
- **Build step**: None — GitHub Pages serves the HTML files directly
- **URL**: `https://jeffgillan.github.io/living_carbon_demo/`

To deploy changes, simply push to the `main` branch:

```bash
git add .
git commit -m "Update app"
git push origin main
```

GitHub Pages will automatically rebuild and serve the updated files.

---

## License

[MIT](LICENSE)
