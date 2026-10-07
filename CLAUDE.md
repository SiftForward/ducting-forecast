# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A client-side static web application for tropospheric ducting forecasts targeting **5G n41 (2.5 GHz)** propagation and amateur radio over CONUS. No backend, no build system, no npm. Deployed to GitHub Pages at `https://siftforward.github.io/ducting-forecast`.

## Development

No build step. Open `index.html` or `site.html` directly in a browser.

Deploy by committing and pushing to `main` — GitHub Pages serves it automatically.

## File Structure

| File | Role |
|------|------|
| `index.html` | CONUS animated loop — 28-frame heatmap (6h intervals) over a 231-point 1° grid |
| `site.html` | Single-site drilldown — M-profile chart, duct table, 0–100 risk score, hop distance |

All JavaScript is embedded in `<script>` blocks inside each HTML file. No separate `.js` source files.

## Data Pipeline

```
Open-Meteo API (GFS forecast / ERA5 archive)
  → fetchRawOM() / fetchSoundings()
  → extractLevels()   ← parse 13 pressure levels + 2m/80m BL + evaporation duct
  → computeProfile()  ← build vertical layer stack (N, M, dM/dh, dN/dh)
  → detectDucts()     ← find dM/dh < 0 regions
  → interferenceRisk() ← score 0–100, classify none/low/moderate/high/severe
  → render (Leaflet heatmap, Chart.js timeline, M-profile canvas)
```

**Open-Meteo endpoints (public, no API key):**
- `https://api.open-meteo.com/v1/forecast` — NOAA GFS, 7-day
- `https://archive-api.open-meteo.com/v1/archive` — ERA5, 5-day publication lag
- `https://geocoding-api.open-meteo.com/v1/search` — city/state autocomplete

**Pressure levels:** 1000, 975, 950, 925, 900, 850, 800, 750, 700, 600, 500, 400, 300 hPa

## Physics Engine (ITU-R P.453-14)

All physics is pure JavaScript:

| Function | Purpose |
|----------|---------|
| `satVP()` | Saturated vapor pressure (Magnus formula) |
| `calcN()` / `calcM()` | Refractivity N and modified refractivity M |
| `computeProfile()` | Build layer stack with dM/dh and dN/dh per layer |
| `detectDucts()` | Find layers where dM/dh < 0 (ray trapping) |
| `_bduct()` | Characterize each duct: base/top height, thickness, critical frequency, max hop |
| `interferenceRisk()` | Score ducts 0–100; handle super-refractive layers |

**Key constants:**
- Ducting threshold: `dN/dh < -157 N/km` (≡ `dM/dh < 0`)
- n41 band: `N41_MHZ = 2593.0`
- Risk scoring: delta-M strength (40 pts) + max hop (30 pts) + duct type (10–20 pts) + super-refractive bonus

## Caching

SessionStorage keys (30-min TTL):
- `index.html`: `kd9wxy_gl_v4_[mode]_[date]` — full grid × 28 frames
- `site.html`: `kd9wxy_grid_v3_[date]` — 231 points × multiple hours

Bump the version suffix (e.g. `v4` → `v5`) when changing data structure or API parameters to avoid stale cache reads.

## External Libraries (CDN only)

- **Leaflet 1.9.4** + **leaflet-heat 0.2.0** — map and heatmap
- **Chart.js 4.4.0** — risk timeline chart
- **CartoDB dark tiles** — basemap
