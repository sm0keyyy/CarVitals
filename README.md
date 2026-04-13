# CarVitals

Self-hosted car telemetry dashboard. Personal project — one vehicle, one user, and a pile of OBD CSVs that wanted somewhere to live.

## What it does

- **Drive ingest** — OBD CSVs land in Dropbox, the server picks them up via webhook, parses every sample, and map-matches the GPS trace against Valhalla's road network. iOS Shortcuts (RoutePush, FuelPush, ServicePush) post drive/fillup/service records directly to the API.
- **Fuel tracking** — tank-level MPG, range, cost-per-mile, savings vs. advertised price. Parser-integrated Fuel Rate will eventually replace ECU trip-meter MPG.
- **Service health** — per-sample degradation across oil, transmission, differential, transfer case, brake fluid, coolant, spark plugs, and tires. Arrhenius / IPL / Eyring / WLF / Miner's Rule physics inside the parse loop, not on summary stats. Headed toward real-measurement calibration (UOA, coolant strips, tread depth, etc.) so the numbers eventually answer to reality instead of a spec sheet.
- **Drive replay** — per-drive telemetry chart + MapLibre trace with interpolated marker, timeline scrubbing, toggleable PIDs.
- **Dashboard** — Solid.js SPA with uPlot time-series, MapLibre GL map overlays, installable as a PWA on iOS (Add to Home Screen) for full-screen standalone use.

## Stack

| Layer | Tech |
|-------|------|
| API | Fastify 5 + Node 20 |
| Database | PostgreSQL 15 + PostGIS 3.3 |
| Routing / map-matching | Valhalla (containerized, `ghcr.io/valhalla/valhalla-scripted`) |
| Frontend | Solid.js + Tailwind v4 + uPlot + MapLibre GL + chroma-js |
| Ingest | Dropbox webhook (OBD), iOS Shortcut HTTP POST, LubeLogger REST sync |
| External APIs | Google Places (station lookup), SerpAPI (parts pricing), Open-Meteo (weather) |
| Runtime | Docker Compose, intended for Unraid |

## Architecture at a glance

```
ios/                 iOS Shortcut uploaders (phone-side, HTTP POST to server)
server-v2/
  src/
    server.js        Fastify entry, route registration, auto-backfill on boot
    routes/          drives, fillups, service, backfill, webhook, api-ui
    lib/             csv-parser, obd-parser, valhalla, event-detector,
                     geocoder, weather, drive-score, degradation,
                     dropbox, lubelogger, gas-station, tire-lookup,
                     price-lookup, vehicle-profile
    lib/cache/       PostGIS-backed caches (edges, places, routes,
                     speed/event anchors, traverse times)
    middleware/      Bearer-token auth
  sql/schema.sql     Authoritative table definitions
  web/               Solid.js dashboard (built by Vite into src/public)
  docker-compose.yml postgres + valhalla + app
```

Everything is cache-first: PostGIS tables are the cache, no Redis, no in-process maps. External API failures (Valhalla, Google, Open-Meteo, SerpAPI) are non-fatal — the drive still saves with whatever fields resolved.

## Running

The app is built to be deployed as a Docker stack on a home server.

```bash
cd server-v2
cp .env.example .env
# fill in POSTGRES_PASSWORD, API_KEY, Dropbox/Google/SerpAPI/LubeLogger creds
docker compose up -d --build
```

The frontend builds inside the `app` container (`Dockerfile` stage 2), then Fastify serves the static bundle from `src/public`. Valhalla builds its tile data on first boot — allow ~5 minutes for the healthcheck to pass (tile extract size depends on `VALHALLA_TILE_URL`, e.g. a single state OSM PBF).

Once up:

- Dashboard: `http://localhost:3001/` (auth: API key as `Bearer`)
- Health: `GET /health`
- Dropbox webhook: `POST /webhook` (configure in your Dropbox app console)

## Status

This is a personal project. It works for my car, on my hardware, with my habits. It is not a product, there is no roadmap for other people, and issues/PRs asking for features outside that scope will probably stay open forever.

You're welcome to read the code, lift pieces of it, or run your own copy. Both are fine. Just don't expect support.

## License

MIT.
