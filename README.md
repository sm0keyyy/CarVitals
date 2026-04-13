# CarVitals

Self-hosted car telemetry dashboard. Started as a single-user, single-vehicle tool to give a pile of OBD CSVs somewhere to live — headed toward multi-user, multi-vehicle as the foundations solidify.

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

Actively developed. Currently scoped to one vehicle and one user (me) while the core is shaking out — per-sample degradation, map-matching, fuel pipeline, and the dashboard shell are the pieces being stabilized first.

On the roadmap:

- **Multi-user + auth** — Google SSO, admin invites, per-user settings. The API already separates read vs. write; this is mostly wiring the session layer.
- **Multi-vehicle** — `vehicle_id` FK through the schema, VIN-based folder discovery on the Dropbox side.
- **Real-measurement calibration** — oil UOA, coolant strips, brake moisture, tread depth, and battery CCA become the ground truth; OBD severity collapses to an interpolation weight.
- **Bidirectional dashboard** — WebSocket write path so fillups / services / file uploads happen in the UI instead of requiring an iOS Shortcut round-trip.
- **Layout picker** — three dashboard layouts (dashboard strip, canvas + inspector, instrument cluster) are already wired; a settings toggle is the last piece.

Issues and PRs are welcome, though response time will depend on whether I'm deep in a refactor when they land. If something you want doesn't fit where the project is heading, I'll tell you rather than leave it open forever.

## License

MIT.
