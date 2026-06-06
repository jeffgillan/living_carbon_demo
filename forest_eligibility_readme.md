# Web App — `forest_eligibility.html` Workflow Documentation

> **Purpose of this document:** A complete, detailed description of how the
> Reforestation Eligibility **web frontend** works today — its structure, state,
> the full submit→poll→display lifecycle, and the exact points where it couples to
> the R pipeline. Written to inform an upcoming rewrite of both this page and the
> pipeline.
>
> **Companion doc:** The backend is documented in
> `reforestation_eligibility/pipeline1_readme.md`. Read both together — sections
> here cross-reference it. The single most important coupling fact: **this page
> drives its progress UI by regex-matching the pipeline's raw log text**, and
> **finds its result by guessing the output filename**. Both are brittle seams the
> rewrite should replace with the structured `status.json` the backend already
> writes.

---

## 1. Overview

`forest_eligibility.html` is a **single, self-contained static HTML file** (~1020
lines: markup + CSS + one inline `<script>`). It is served from GitHub Pages — no
build step, no framework, no bundler, no backend of its own. All logic is vanilla
JavaScript in one inline script block.

**What it does, in one sentence:** lets a user draw or upload an Area of Interest,
submits it to the pipeline (through a passcode-gated bouncer), shows live progress
parsed from the pipeline log, and renders the resulting eligible-area polygon on a
map.

### Third-party libraries (all via unpkg CDN, pinned versions)

| Library | Version | Role |
|---|---|---|
| **MapLibre GL JS** | 5.1.0 | The map engine (renders Esri/Google imagery + vector layers) |
| **Mapbox GL Draw** | 1.5.0 | Polygon drawing tools (patched onto MapLibre — see §5) |
| **Turf.js** | 7 | Client-side geometry math (`bbox`, `area`) |

There is **no API key** for the basemap — it pulls Google satellite tiles directly
(`https://mt1.google.com/vt/lyrs=y&...`). Worth noting for the rewrite (ToS / a
proper tile provider).

---

## 2. Page Layout (DOM Structure)

The page is a full-screen `#map` with a floating `#control-panel` (top-right,
scrollable). The panel has four sections:

```
#control-panel
├── Back link + header (logo + title) + one-line description
├── [1. Define AOI]      #draw-tools-container   (draw toolbar is moved here)
│                        #geojson-upload          (<input type=file>)
├── [2. Run Analysis]    #site-name, #passcode
│                        <details> Advanced params: #slope-threshold, #ndvi-threshold
│                        #calc-btn  (disabled until AOI + passcode present)
│                        #job-status   (the headline status line)
│                        #progress-tracker  (7 step rows + #elapsed)   ← hidden until run
│                        #live-metrics      (acres at each stage)      ← hidden until data
│                        <details> Show technical log → #log-pane (raw streamed log)
└── [Map Layers]         base toggles: #toggle-imagery, #toggle-aoi
                         #result-layers  (one TOC row per completed run)
                         #toc-empty      (placeholder when no results)
```

The 7 progress-tracker step rows are **hardcoded in the markup** with
`data-step` ids: `start, exports, terrain, wetlands, hydric, nci, integration`.
These correspond to (but are NOT identical to) the backend's six module step keys.

---

## 3. Configuration Constants (top of `<script>`)

| Constant | Value | Notes |
|---|---|---|
| `BOUNCER_URL` | `https://pipeline-bouncer-527567483631.us-central1.run.app` | All `/run` and `/logs` calls go here (passcode gate) |
| `GCS_BUCKET` | `livingcarbon_bucket` | Result GeoJSON is fetched **directly** from public GCS, bypassing the bouncer |
| `GCS_OUTPUT_PREFIX` | `outputs` | Path prefix in the bucket |
| `MAP_CENTER` / `MAP_ZOOM` | `[-85, 39]` / `4` | Initial US-centered view |
| `POLL_INTERVAL_MS` | `1000` | **Declared but unused** (the live poller uses `LOG_POLL_MS`) |
| `NOT_READY_TIMEOUT_S` | `30` | **Declared but unused** (relic of removed "no new output" warning) |
| `LOG_POLL_MS` | `1500` | Actual log-poll cadence |
| `COLD_START_MSG` | "System booting up…2-3 minutes…" | Shown until the first real log line appears |

> **Rewrite note:** two dead constants (`POLL_INTERVAL_MS`, `NOT_READY_TIMEOUT_S`)
> and a `TODO` comment about `GCS_BUCKET`. The result-fetch path also hardcodes the
> public-GCS URL scheme rather than asking the API where the result is.

---

## 4. Application State (module-level `let`s)

| Variable | Holds |
|---|---|
| `draw` | The MapboxDraw control instance |
| `currentAOI` | The active AOI as a **FeatureCollection in EPSG:4326** (or `null`). The single source of truth for "what will be submitted." |
| `activeJob` | `{ id, siteName, submittedAt }` for the in-flight job |
| `jobInFlight` | Boolean lock — disables the Calculate button while a job runs |
| `resultSeq` | Monotonic counter for unique result-layer ids |
| `resultLayers` | Array of completed-run entries (each owns its own map source/layers + TOC row) |
| `logBuffer` | The **accumulated raw pipeline log** — this string is re-parsed on every poll to drive the tracker and metrics |

**Session-only:** results live in memory. A page reload clears everything. There is
no persistence, no URL state, no way to reopen a past job.

---

## 5. Map & Draw Initialization

- A MapLibre map is created with a single raster source (Google satellite tiles).
- On `map.load`, **Mapbox GL Draw is monkey-patched** to use MapLibre's CSS class
  names (`MapboxDraw.constants.classes.* = 'maplibregl-*'`) — this is the standard
  hack to run the Mapbox draw plugin on MapLibre. A full custom `styles` array is
  supplied for the draw layers (blue fills/strokes/vertices).
- The draw toolbar is created at `top-left` then **physically moved** into
  `#draw-tools-container` in the control panel via `appendChild`.
- Three draw events are wired: `draw.create → onDrawCreate`,
  `draw.update → onDrawUpdate`, `draw.delete → onDrawDelete`.

---

## 6. Defining the AOI (two input paths)

Both paths converge on `setAOI(...)`, which normalizes to a clean EPSG:4326
FeatureCollection and stores it in `currentAOI`.

### Path A — Draw
- `onDrawCreate` **enforces a single drawn polygon** (deletes all but the latest),
  removes any uploaded AOI from the map, then calls `setAOI(e.features[0])`.
- `onDrawUpdate` re-runs `setAOI` on edit. `onDrawDelete` clears `currentAOI`.

### Path B — Upload (`#geojson-upload`)
- Reads the file, `JSON.parse`s it.
- **CRS check:** if a `crs` member declares anything other than CRS84/4326, it
  *warns* (alert) but proceeds — the pipeline reprojects internally, but the map
  may render the polygon in the wrong place.
- `validateGeoJSON()` extracts **every** Polygon/MultiPolygon feature (supports a
  `FeatureCollection`, a single `Feature`, or a bare geometry) so multi-stand AOIs
  are analyzed whole. Returns `null` if none found.
- Clears any drawn polygon, removes a prior uploaded AOI, adds the new one as map
  layers (`uploaded-aoi-fill` / `-outline`), fits bounds, and calls `setAOI(fc)`.

### `setAOI()` normalization (important for the pipeline contract)
- Accepts a single Feature (from draw) or a FeatureCollection (from upload).
- Strips MapboxDraw-specific feature ids; ensures each feature has a `properties`
  object; keeps **all** features.
- **Deliberately emits NO `crs` member** (RFC 7946: GeoJSON is implicitly WGS84;
  a deprecated `crs` member can make `sf::st_read` mishandle coordinates).
- Stores `{ type: "FeatureCollection", features: [...] }` in `currentAOI`.

---

## 7. Validation & the Calculate Button

- `refreshCalcButton()` enables `#calc-btn` only when:
  `!jobInFlight && currentAOI exists && passcode field non-empty`.
- Re-checked on passcode/site input events and after AOI changes.
- `sanitizeSite()` strips the site name to `[A-Za-z0-9_-]`, falling back to
  `aoi_<timestamp>` if empty.
- Advanced-param ranges (slope 0–100, NDVI −1–1) are validated **again** at submit
  time (the pipeline also re-validates server-side).

### Base-layer toggles
`#toggle-imagery` and `#toggle-aoi` flip MapLibre layer `visibility`. The uploaded
AOI toggle only affects uploaded polygons (drawn ones are governed by the draw
tools), per the label's tooltip.

---

## 8. Submit Flow — `runPipeline()`

1. Guard on `currentAOI` + passcode; set `jobInFlight = true`, disable button.
2. `clearLog()` and `resetTracker()` — **note: `resetTracker` does NOT touch saved
   results**; prior result layers persist and stack.
3. Build the request **body**: `{ passcode, ...currentAOI }` (the FeatureCollection
   spread alongside the passcode). The polygon coordinates are intentionally NOT
   logged — only the polygon count.
4. Build the request **URL** as a query string:
   `${BOUNCER_URL}/run?site_name=...[&slope_threshold_pct=...][&ndvi_canopy_threshold=...]`.
   Only the two exposed advanced params are sent; all other thresholds use backend
   defaults.
5. `POST` to the bouncer. Response parsing is defensive: it finds the first `{` in
   the response text and `JSON.parse`s from there (the bouncer may prepend text),
   and unwraps `job_id` whether it's a scalar or a 1-element array.
6. On success: store `activeJob`, show `COLD_START_MSG`, start the elapsed timer,
   and call `pollLogs(jobId, passcode)`.
7. On failure: surface the error in the status line and release the lock.

---

## 9. Polling — `pollLogs(jobId, passcode)`

A recursive `setTimeout(tick, 1500)` loop (not `setInterval`), `POST`ing to
`${BOUNCER_URL}/logs/<jobId>?offset=N` with the passcode in the body.

Per tick:
- **HTTP 404** → job folder not in GCS yet (earliest cold start). Keep the friendly
  booting message and keep polling. *Not* treated as an error.
- Parse the JSON response (same "find first `{`" defensive parse). Unwrap each field
  (`content`, `status`, `done`, `next_offset`) from possible 1-element arrays — an
  artifact of plumber's JSON serialization.
- Append any new `content` to both the raw `#log-pane` and `logBuffer`, then call
  `updateTrackerFromLog()`.
- Advance `offset` to `next_offset` (server-side byte offset into `pipeline.log`).
- When `done` is true:
  - `status === "completed"` → `markAllStepsDone()`, then
    `fetchAndDisplayResult(...)`.
  - otherwise → `markCurrentStepError()` and show a failure message.
  - Release the `jobInFlight` lock either way.
- Network/parse errors abort the loop, show a connection error, and release the lock.

The matching backend endpoint shape (`{offset, next_offset, status, content, done}`)
is documented in `pipeline1_readme.md` §2 (Phase B/C) — **the rewrite must keep
these shapes stable or change both sides together.**

---

## 10. Progress Tracker — the log-regex coupling ⚠️

This is the most fragile part of the app and the prime rewrite target.

`STEP_ORDER = [start, exports, terrain, wetlands, hydric, nci, integration]`.

`updateTrackerFromLog()` re-scans the **entire accumulated `logBuffer`** on every
poll and infers step completion by **regex-matching specific log strings the
pipeline happens to print**:

| Step | "finished" detected by regex |
|---|---|
| start | `GEE Exports` \| `Started GEE task` \| `AOI: <num>` |
| exports | `Downloaded:` \| `Terrain, Canopy` |
| terrain | `Terrain/canopy eligible area:` |
| wetlands | `After wetland exclusion:` \| `No (NWI )?wetlands` |
| hydric | `After hydric soil exclusion:` \| `No hydric soil` |
| nci | `NCI flag:` |
| integration | `Pipeline completed successfully` |

It then marks every step up to the last matched one `done`, the next one `running`,
the rest `pending`. **Live metrics** are scraped the same way (regex-capturing
acreage numbers): `N/7 exports in GCS`, `AOI: <n> total acres`,
`Terrain/canopy eligible area: <n>`, `After wetland exclusion: <n>`,
`After hydric soil exclusion: <n>`, `Final eligible area: <n> ac (<n>%)`,
`NCI flag: <word>`.

> **Why this is brittle:** any reword of a pipeline log line silently breaks the
> tracker and/or the live metrics. The backend **already writes a structured
> `status.json`** with per-step `running/completed/skipped` state and a `note`
> field — the rewrite should poll `/status` and render from that instead of
> reverse-engineering prose. (See `pipeline1_readme.md` §9 "Step tracking is
> log-regex-based.")

Supporting helpers: `setStep(step, state, extra)` swaps the row's CSS class + icon
(`○ ◐ ✓ ⏭ ✕`); `refreshHeadline()` keeps `COLD_START_MSG` until real output appears,
then shows "Running: <step>…"; `startElapsed`/`stopElapsed` drive the `Elapsed:
Xm Ys` timer.

---

## 11. Fetching the Result — `fetchAndDisplayResult()` ⚠️

Once `status === "completed"`, the page fetches the result GeoJSON **directly from
public GCS** (not via the API). Two pieces of guesswork are involved:

1. **Filename date guess.** The integration step names the file
   `<site>_eligible_area_<YYYY-MM-DD>.geojson` using the server's `Sys.Date()`
   (UTC). The client doesn't know that date, so `candidateDates()` builds **today,
   yesterday, and tomorrow** (UTC) to survive midnight crossings.
2. **Upload-lag retry.** The job is marked `completed` *before* the `gcloud storage
   cp -r` upload finishes, so the object can lag the signal. The client sweeps all
   candidate dates up to **12 times, 3 s apart (~36 s total)** until one URL returns
   `200`.

URL shape (`buildResultUrl`):
```
https://storage.googleapis.com/<bucket>/outputs/<jobId>/final/<site>_eligible_area_<date>.geojson
```
(The comment notes this assumes the *single-nested* path from the migrated pipeline
image — an older image produced a doubled `<jobId>/<jobId>/` path.)

On success → `addResultLayer()`. On exhaustion → an error listing every tried URL,
with a hint about GCS bucket path / **CORS** (the bucket must allow cross-origin
GET from the GitHub Pages origin).

> **Rewrite note:** this whole date-sweep + retry dance exists only because the
> client guesses the filename and the "completed" flag races the upload. The API
> already has a `GET /results/<id>` endpoint that *lists* the actual output
> filenames — using it (and flipping "completed" only after upload, which the
> backend already intends) would remove all the guessing.

---

## 12. Results — Persistent Layers + Table of Contents

Each completed run is **independent and additive**:

- `addResultLayer(geojson, meta)` assigns a unique id (`result-<seq>`), adds its own
  GeoJSON source + green fill + outline layers, computes eligible acres
  (`turf.area`) and % of the current AOI, pushes an entry to `resultLayers`, adds a
  TOC row, and fits bounds.
- `addTocRow(entry)` renders a row with a visibility checkbox and three per-result
  actions:
  - **Zoom** (`⤢`) — fit bounds to that result.
  - **Download** (`⭳`) — client-side Blob download of the GeoJSON.
  - **Remove** (`✕`) — removes the layers/source/row and the array entry.
- Results stack on the map and persist across subsequent runs; they are
  independent of the input AOI (editing/deleting the AOI never removes a result).
  All session-only — cleared on reload.

---

## 13. End-to-End Sequence (quick reference)

```
draw / upload ──▶ setAOI() ──▶ currentAOI (4326 FC)
                                   │  + passcode
                                   ▼
runPipeline() ── POST {passcode,...FC} ──▶ BOUNCER /run?site_name&slope&ndvi
                                   ◀── { job_id }
                                   ▼
pollLogs() ── POST /logs/<id>?offset ──▶ BOUNCER  (every 1.5s)
   │  appendLog + logBuffer
   │  updateTrackerFromLog()  ⚠ regex on log prose
   ▼  (until done)
status==completed ──▶ fetchAndDisplayResult()
                         └─ GET storage.googleapis.com/.../final/<site>_eligible_area_<date>.geojson
                            ⚠ date guess + 12× retry for upload lag
                         └─ addResultLayer() ──▶ green polygon + TOC row
```

---

## 14. Summary of Rewrite-Relevant Pain Points

These are the seams worth redesigning, in rough priority order:

1. **Progress is parsed from log prose** (§10). Replace with polling the structured
   `status.json` via `/status/<id>` — the backend already maintains per-step state
   and a human-readable `note`. This decouples UI from log wording.
2. **Result is found by guessing the filename + date and racing the upload** (§11).
   Use `GET /results/<id>` to list real filenames, and have the backend flip
   "completed" only after the upload lands (it already intends to). Removes the
   date sweep and the ~36 s retry loop.
3. **Two transport paths** — control plane goes through the bouncer; the result
   GeoJSON is fetched straight from public GCS (requires a CORS policy). Consider
   serving the result through the API too, or formalizing the GCS contract.
4. **Single drawn polygon enforced**, but uploads allow multi-feature AOIs — an
   inconsistency to resolve.
5. **No persistence / no shareable job URL** — everything is session-only and
   reload wipes it.
6. **Only 2 of ~8 tunable parameters are exposed** in the UI (slope, NDVI); the
   rest (canopy %, wetland buffer, hydric %, min patch acres, NAIP dates) use
   backend defaults and aren't surfaced.
7. **Dead constants & TODOs** (`POLL_INTERVAL_MS`, `NOT_READY_TIMEOUT_S`,
   `GCS_BUCKET` TODO comment) and the doubled-path migration comment to clean up.
8. **Basemap pulls Google tiles directly** with no API key — revisit the tile
   provider / ToS for production.
9. **Plugin coupling** — Mapbox GL Draw is monkey-patched onto MapLibre; a known
   hack, but worth a deliberate decision in the rewrite (native MapLibre draw or a
   maintained fork).
```
