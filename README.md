# Prix essence QC

Single-page static web app that surfaces the cheapest gas stations in Quebec, based on the **Régie de l'énergie du Québec**'s live data. No backend, no build step — one `index.html` file plus a folder of brand icons.

## What it does

Three modes, same data:

- **Favoris** — stations you've starred, sorted by price.
- **Autour de moi** — stations within a configurable road-distance from your current location (GPS or typed address), sorted cheapest first. Capped at 20 to keep the list signal-to-noise high.
- **En route** — enter origin + destination, pick how far you're willing to detour (250 m → 2.5 km), and see the cheapest stations along the actual driving route (not as-the-crow-flies).

Each card links to Google Maps directions — if you're in trip mode, the station is inserted as a waypoint between origin and destination.

## Data source

**Primary:** `https://regieessencequebec.ca/stations.geojson.gz` — the same GeoJSON feed powering the Régie's official map. Re-fetched on every page load, ~60 s upstream cache. Contains ~2,400 stations across QC with geometry, brand, address, and per-fuel prices (Régulier, Super, Diesel). Updated continuously by retailers under the April 2026 pricing-disclosure law.

**Geocoding** (fallback when GPS is blocked or on `file://`): **Nominatim** (OpenStreetMap) — CORS-open, free, no key.

**Road distances & routing:** **Google Maps JavaScript API** (Distance Matrix + Directions). The API key is in the HTML but restricted by HTTP referrer in Google Cloud Console — harmless if leaked.

## How it picks stations

- **Nearby mode:** straight-line filter to (max-radius × 1.5) as a coarse pre-filter, capped at 100 candidates. Those are sent in a single batched Distance Matrix call (25 per request). Filter by actual road distance ≤ slider, sort by price, keep top 20. Favorites are force-included even if outside the radius.
- **Trip mode:** one Directions call gets the route polyline, then every station is tested against its perpendicular distance to the polyline (using equirectangular projection — accurate to a few meters at Quebec latitudes). Filter by detour ≤ slider, sort by price then by distance-along-route so on ties you hit cheaper stations earliest.
- **Savings estimate** (optional toggle): `(avg of other shown stations − this price) × tank size`, displayed under the L price.

All three modes cache results per origin (and per destination in trip mode). Moving the slider or changing fuel type is instant — no re-fetching.

## Persistent state

Stored in `localStorage` (no account, no sync):

- `gaz_favorites_v1` — starred station coordinates
- `gaz_places_v1` — saved Home / Work / etc. chips
- `gaz_lang_v1` — FR (default) / EN
- `gaz_theme_v1` — auto / light / dark
- `gaz_fuel_v1` — Régulier / Super / Diesel / Tous
- `gaz_radius_v1`, `gaz_detour_v1` — slider positions per mode
- `gaz_savings_enabled_v1`, `gaz_tank_size_v1` — savings feature

## Brand icons

`/icons/*.png` — 48×48 white-bg PNGs normalized via ImageMagick from sources around the web (Wikipedia, each brand's own site, etc.). Originals kept in `/icons-raw/` for re-processing.

31 brands covered, ~98% of branded stations. Unknown brands (including future additions to the data) render a colored text badge in the same 48×48 rounded box, so the layout stays consistent.

## UI niceties

- Theme uses CSS `light-dark()` with `color-scheme` — no flash on load, follows system by default.
- City names in addresses get title-cased client-side (source data is inconsistent, often `ALL-CAPS`).
- Visitor counter in the footer via CounterAPI (counterapi.dev), cached per tab.

## Hosting

GitHub Pages. Open the URL on a phone: HTTPS unlocks geolocation. On `file://`, GPS is blocked by most browsers — address search fallback works anywhere.

## Tech

- One `index.html` file (CSS + JS inline)
- No build step, no package.json, no framework
- Only runtime deps are the public APIs above
