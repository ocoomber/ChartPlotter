# Chart Plotter — AGENTS.md

## Stack
- Single `index.html` (no build step, no server-side code)
- Leaflet 1.9.4 (unpkg CDN), OSM tiles + OpenSeaMap seamark layer

## Mobile
- Panel collapses to a slide-out drawer at ≤768px viewport width
- Hamburger button (top-left) toggles `#panel.open`
- Close button (✕) in panel header also toggles
- Map fills full viewport when panel is closed
- All draggable markers use `bubblingMouseEvents: false` + 44px touch target (waypoints and measure markers)
- **Known issue**: Measure tool marker drag breaks on Android Chrome — touch/pointer events from marker icon still reach map pan handler despite `bubblingMouseEvents: false` and explicit `touchstart/touchmove/touchend stopPropagation`

## Serving
- **Must serve via HTTP** for OSM tile referrer policy — OSM blocks `file://`
- Use Python: `python -m http.server 8000` → `http://localhost:8000`

## Coordinates
- **WGS84 decimal degrees, lat-first**, stored as `{lat, lon}` (never `{lng, lon}`)
- All coordinates rounded to 5dp via `roundCoord()` before storage
- Leaflet default CRS: EPSG:4326 for data, EPSG:3857 for tiles
- Haversine: `R = 3440.065 NM`. Bearing: degrees true 0-360.

## Waypoints
- `const waypoints = []` — array of `{lat, lon, marker}`
- Click on map → `addWaypoint()` at click location (unless Measure tool is active)
- Markers: `L.marker` with `draggable: true` (not `L.circleMarker`)
- Drag updates `wp.lat/wp.lon` and calls `updatePolyline()`; `dragend` calls `updateRoute()`
- `removeWaypoint(index)` splices from array and renumbers markers
- `insertWaypointAtIndex(lat, lon, idx)` inserts a waypoint at a given index (used by clicking on a route leg)
- Route rendered as `segmentPolylines[]` — one `L.polyline` per leg, each with a sticky tooltip showing `dist NM @ brg°T`
- Clicking a segment polyline inserts a new waypoint between the two endpoints
- Leg list table replaces the old waypoint table — each row shows Leg #, Dist, Brg, and a remove button
- Header count: `3 wps, 2 legs`

## Measure tool
- Toggle button in actions bar. When active, map click does **not** add waypoints.
- Click point A (drops a red dot), then point B (draws dashed line).
- Distance and bearing are always visible in a label at the midpoint of the line, with a chevron arrow indicating direction from A to B.
- Endpoint markers are **draggable** — drag either one to adjust the measurement; line and label update in real time.
- Third click resets and starts a new measurement from that point.
- Toggle off or `clearAll()` clears all measurement markers and lines.

## Cursor coordinate readout
- Bottom-right corner of the map, shows `lat, lon` at 5dp.
- Updates on `mousemove` — no toggle needed, always on.

## Magnetic variation
- Section between wind overlay and waypoint list: number input (degrees) + select (W/E).
- When non-zero, bearing column shows both True and Magnetic: `045°T / 048°M`.
- Formula: `mag = (true + var) % 360` where West variation is positive, East is negative.
- `onVarChange()` updates `renderWaypointList()` immediately.

## Wind overlay (Open-Meteo)
- Toggle checkbox enables/disables. Fetches from `api.open-meteo.com/v1/forecast`
- Builds a 21×21 grid around route (or map center if no route)
- API returns array of per-location objects: `data[n].hourly.wind_speed_10m[hour]`
- 16-day forecast window — departure constrained to now + 13 days (3-day fetch must fit within 16-day window)
- Bilinear spatial interpolation (u/v), linear temporal interpolation
- Purple arrows: grid arrows (transparent, small) + waypoint arrows (opaque, offset windward, tooltip shows local time)
- Slider represents hours after departure time: `+0h (14:00)`, `+6h (20:00)`, etc. (local timezone, not Z)

## Export/Import
- GPX v1.1 (`trkpt lat/lon` at 5dp), CSV (`lat, lon` per line)
- Import CSV reads from file picker (not textarea)
- Downloads use Blob + object URL

## Build number
- Placeholder `<!--BUILD_NUMBER-->` in `index.html` (panel header)
- CI workflow `.github/workflows/deploy.yml` replaces it via `sed` before deploying to Pages
- Format: `Build {run_number} — {timestamp}` where run_number is GitHub Actions' auto-increment and timestamp is UK local time (BST/GMT)
- Invisible in local dev (HTML comment), appears only on deployed site

## Panel layout (top to bottom)
1. Header (Marine toggle checkbox, build number)
2. Actions bar (Clear All, Export GPX, Export CSV, Measure, Import CSV)
3. Wind overlay section (checkbox, departure time, speed, slider, note)
4. Variation section (degrees, W/E select)
5. Leg list (leg, dist, brg, remove button)
6. Total distance row

## Key constraints
- Everything is in a single file — no imports, no modules, no build step
- All globals (no module scoping)
- Functions referenced from HTML `onclick`/`onchange` attributes must be global
- Map center initialized to `[50.3, -5.0]` at zoom 10 (English Channel)
