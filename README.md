# Skyward

An arcade flight simulator in a single HTML file — [`index.html`](index.html). Three.js via CDN import map, no build step, no assets (terrain, clouds, and every sound are generated at runtime).

## Play

The page uses ES modules, so it needs to be served rather than opened from `file://`:

```bash
python3 -m http.server 8123
```

Then open http://localhost:8123.

## Controls

| Key | Action |
| --- | --- |
| `W` / `S` | Pitch down / up |
| `A` / `D` | Roll left / right |
| `Q` / `E` | Rudder |
| `↑` / `↓` | Throttle |
| `C` | Chase / cockpit camera |
| `R` | Respawn |
| `N` | New course |
| `M` | Mute |
| `Z` | Minimap zoom (2.5 / 5 / 10 km) |

## The game

Fly through all 12 gold rings in order, as fast as you can. The next ring is gold and pulsing; the others are cyan. An on-screen marker tracks the active ring and pins itself to the screen edge when it is behind you. Your best time is kept in `localStorage`. Finishing generates a fresh course.

Crashing into terrain or water puts you back on approach to the ring you were heading for, so a mistake costs seconds rather than the run.

You are not alone up there. Fourteen aircraft fly their own routes across the islands. Come within 750 m and the HUD reads `TRAFFIC`, the contact turns red on the minimap, and a blip sounds faster the closer you get. Hit one and it is a mid-air.

## How it works

- **Terrain** — value-noise fBm heightfield (`terrainHeight`), 300×300 grid over 14 km. A continent layer decides land vs. ocean, a ridged layer adds mountains only where there is already land, and vertex colours come from height and slope. The same function is sampled directly for collision, so the physics and the mesh can never disagree.
- **Flight model** — arcade, not aerodynamic. Speed chases a throttle-driven target and trades against climb angle (`speed -= forward.y * GRAVITY * dt`), control authority falls off with airspeed, banking couples into yaw for coordinated turns, and below stall speed the nose drops and the plane sinks. Wings self-level when the stick is centred and the plane is upright.
- **Ring detection** — each frame the previous and current positions are transformed into ring-local space; a sign change in local `z` plus a radial hit inside the torus radius counts as a pass. Frame-rate independent, no tunnelling.
- **Minimap** — the whole world is baked once into a 320×320 offscreen canvas: land colours from the same palette as the 3D terrain, depth-shaded ocean, and a hillshade computed from central differences on the height grid (no extra noise sampling). Each frame the map blits a player-centred, north-up window out of that bake and overlays the ring course. `drawImage` clips a source rectangle that runs off the bake, so the destination rectangle is clipped to match — otherwise the map would smear near the world edge.
- **Traffic** — each aircraft is one merged geometry (a single draw call) flown by a waypoint autopilot: limited turn rate, bank proportional to turn rate, pitch derived from climb rate. The interesting part is altitude. Short-range lookahead cannot save an aircraft from a 1500 m peak — by the time the peak is in range there is no distance left to climb. So the *whole leg* is sampled when a waypoint is chosen, candidate legs whose terrain the aircraft could not out-climb are rejected outright, and a hard floor clamp guarantees nothing is ever seen inside a mountain. Over ten simulated minutes the clamp engages on 0.016% of samples; routing does the rest. Traffic streams around the player — anything beyond 9 km is recycled to a fresh bearing 3.2–6.2 km away, chosen by retrying bearings rather than clamping to the world bounds, since clamping drags the spawn point toward the player and pops an aircraft in at close range.
- **Audio** — WebAudio only. Three detuned oscillators through a lowpass make the engine, its pitch and cutoff following throttle and speed; ring chimes climb a semitone ladder; the crash is a filtered noise burst.

## Development

`window.SKYWARD` exposes the sim state (`state`, `plane`, `rings`, `nextRing`, `terrainHeight`, `respawn`, `newCourse`, `mapSpan`, `cycleMapZoom`, `traffic`, `updateTraffic`, `renderer`, `drawMap`) for poking at things from the console.

Two things worth knowing when testing from the console:

- Wait on `requestAnimationFrame`, not `setTimeout`, before reading HUD text — the render loop is what writes it, and an unfocused tab throttles frames, which makes the HUD look one step behind and can read as a bug that isn't there.
- For anything needing more than a few seconds of simulation, step the systems directly (`updateTraffic(1/60)` in a loop) instead of waiting on real frames. Ten simulated minutes of traffic runs in a fraction of a second and is deterministic.

Two three.js gotchas this code deliberately works around:

- `Object3D.lookAt` aims an object's **+Z** at the target, but the aircraft's nose and the flight model's forward vector are **−Z**. Respawn computes the yaw explicitly instead.
- `lookAt` reads `matrixWorld`, which is stale until the next render — another reason not to use it right after moving an object.
