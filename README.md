# Hormuz Fleet Command

Real-time fleet command system for the Strait of Hormuz (Code Rush – Web Dev Track).

## Run

```bash
npm install
npm start            # http://localhost:3000
# or
docker compose up --build
```

## Status (Step 1 of 6)

| Done | Feature |
|------|---------|
| ✅ | 15-ship simulator loaded from `data/fleet.json`, ticking at 2 Hz |
| ✅ | WebSocket push to all viewers, sequence-numbered snapshots (no polling) |
| ✅ | Map with ships, ports, navigable-water outline, click-for-details |
| ✅ | Client interpolation with a max-speed clamp (no teleporting) |
| ✅ | Fuel burn model + "insufficient fuel" flag |
| ⬜ | Roles (Command / Captain), zone drawing, geofence alerts |
| ⬜ | Routing + reroute triggers, edge cases |
| ⬜ | Directives, ACCEPT / ESCALATE_DISTRESS, AI distress parsing |
| ⬜ | Weather (Open-Meteo), proximity warnings, playback |

## Architecture

```
src/server.js          HTTP (static files) + WebSocket + tick loop
src/sim/simulator.js   owns fleet state, step() per tick, snapshot()/hello()
src/sim/ship.js        one ship: follows a route, burns fuel, updates status
src/geo.js             haversine, bearing, destination-point maths
src/config.js          every tunable value (env-overridable)
public/                vanilla-JS client (Leaflet map, list, detail panel)
data/fleet.json        fixed scenario data supplied by the organisers
```

The server is the single source of truth. Every tick it advances all ships and
broadcasts one full snapshot (`seq`, `t`, `ships[]`) to all clients, so everyone
sees identical state. Late joiners receive a `hello` message with static data
plus the current snapshot.

## Environment variables

See `.env.example`. None are required yet; an `ANTHROPIC_API_KEY` will be needed
for AI distress parsing in a later step.

## Assumptions (documented on purpose)

1. **Time scale.** Real-time ships (~14 kn ≈ 7 m/s) barely move on a regional map, so
   simulated time runs `TIME_SCALE`× faster than wall-clock time (default 60).
   Set `TIME_SCALE=1` for true real time. Distance per tick = speed × tick duration × scale.
2. **Fuel burn.** The brief gives fuel in tons but no burn rate. We assume
   `2.5 t per nautical mile` (`FUEL_TONS_PER_NM`), chosen so MV-7 *Gharial*
   (750 t, ~330 nm from Kuwait) cannot make its destination, exercising the
   "insufficient fuel" edge case. Adverse weather multiplies burn by 1.3 (per spec).
3. **Insufficient fuel.** Flagged when estimated fuel needed for the remaining route
   exceeds fuel on board; the ship keeps sailing until fuel reaches 0 (then `out_of_fuel`).
4. **Arrival.** A ship is `arrived` within 1 km of its destination port.
5. **Straight-line routes (temporary).** Step 1 sails straight to the destination.
   Land/zone-avoiding routing arrives in Step 3.
6. **Max speed for interpolation.** The client never draws a ship faster than
   `MAX_SPEED_KNOTS` (default 30 kn × time scale); fastest fleet ship is 22 kn.
7. **Polygon quirk.** In `navigableWater` the points `[22.00, 60.00]` and `[22.50, 60.00]`
   are consecutive along the eastern edge (a zero-width spike). It is kept as given;
   all 15 starting positions and all 10 ports lie inside the polygon.
8. **Map tiles.** CARTO Voyager raster tiles over the internet (basemap tiles are
   explicitly allowed to be external). Leaflet itself is served locally.
