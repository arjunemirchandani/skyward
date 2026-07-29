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
| `SPACE` | Wheel brakes (on the ground) |

## The game

Fly through all 12 gold rings in order, as fast as you can. The next ring is gold and pulsing; the others are cyan. An on-screen marker tracks the active ring and pins itself to the screen edge when it is behind you. Your best time is kept in `localStorage`. Finishing generates a fresh course.

Crashing into terrain or water puts you back on approach to the ring you were heading for, so a mistake costs seconds rather than the run.

You are not alone up there. Fourteen aircraft fly their own routes across the islands. Come within 750 m and the HUD reads `TRAFFIC`, the contact turns red on the minimap, and a blip sounds faster the closer you get. Hit one and it is a mid-air.

### Landing

Three airfields are scattered across the islands, marked in amber on the minimap. Fly within 8 km of one and an approach panel appears with course and glidepath needles — each needle shows where the beam is, so you steer toward it. Get established on final and Center clears you to land.

Touchdowns are scored out of 100 on four things: sink rate, how close to the centreline, how well aligned with the runway, and where in the touchdown zone you put it. You get a grade, and your best is kept. Arrive too hard, too crossed-up, or too banked and it is an accident instead. On the ground, steer with `A`/`D`, brake with `SPACE`, and pull back once you have speed to fly again.

They are all on the radio, and so are you — your callsign is `REDTAIL ONE`. Skyward Center clears you for the gate course, calls your progress, warns you about terrain and traffic (with a clock position), and notices when you go off the scope. The islands they all report against are named on the minimap.

## How it works

- **Terrain** — value-noise fBm heightfield (`terrainHeight`), 300×300 grid over 14 km. A continent layer decides land vs. ocean, a ridged layer adds mountains only where there is already land, and vertex colours come from height and slope. The same function is sampled directly for collision, so the physics and the mesh can never disagree.
- **Flight model** — arcade, not aerodynamic. Speed chases a throttle-driven target and trades against climb angle (`speed -= forward.y * GRAVITY * dt`), control authority falls off with airspeed, banking couples into yaw for coordinated turns, and below stall speed the nose drops and the plane sinks. Wings self-level when the stick is centred and the plane is upright.
- **Ring detection** — each frame the previous and current positions are transformed into ring-local space; a sign change in local `z` plus a radial hit inside the torus radius counts as a pass. Frame-rate independent, no tunnelling.
- **Minimap** — the whole world is baked once into a 320×320 offscreen canvas: land colours from the same palette as the 3D terrain, depth-shaded ocean, and a hillshade computed from central differences on the height grid (no extra noise sampling). Each frame the map blits a player-centred, north-up window out of that bake and overlays the ring course. `drawImage` clips a source rectangle that runs off the bake, so the destination rectangle is clipped to match — otherwise the map would smear near the world edge.
- **Traffic** — each aircraft is one merged geometry (a single draw call) flown by a waypoint autopilot: limited turn rate, bank proportional to turn rate, pitch derived from climb rate. The interesting part is altitude. Short-range lookahead cannot save an aircraft from a 1500 m peak — by the time the peak is in range there is no distance left to climb. So the *whole leg* is sampled when a waypoint is chosen, candidate legs whose terrain the aircraft could not out-climb are rejected outright, and a hard floor clamp guarantees nothing is ever seen inside a mountain. Over ten simulated minutes the clamp engages on 0.016% of samples; routing does the rest. Traffic streams around the player — anything beyond 9 km is recycled to a fresh bearing 3.2–6.2 km away, chosen by retrying bearings rather than clamping to the world bounds, since clamping drags the spawn point toward the player and pops an aircraft in at close range.
- **Airfields** — there is no flat ground in a noise heightfield, so the terrain is made flat where it needs to be. `baseTerrain` is the raw landscape; sites are chosen from it by scanning for low, even coastal ground and orienting each strip along its flattest bearing. `terrainHeight` then blends those pads into the landscape with a smoothstep ramp. Because every system — the mesh, collision, the minimap bake, island finding, traffic routing — reads `terrainHeight`, levelling the ground in one function is enough to make the whole world agree; the strips measure dead flat and traffic still clears the terrain everywhere. Runway markings are painted into a canvas in runway-local units, with each designator placed at the threshold a pilot crosses when landing on it.
- **Landing score** — four independent components, each a taper between a "good" and a "bad" value: sink rate (40), centreline offset (20), alignment (20) and touchdown zone (20). Either runway direction is a legitimate landing; the judge picks whichever end you are flying toward and measures against that threshold. Outside the survivable envelope it is a crash, not a low score.
- **Radio chatter** — every transmission is generated from live simulation state, never from a canned script: an aircraft's reported altitude is its actual altitude, "climbing" means its climb rate is positive, and "direct MARROW ISLE" names the reporting point nearest its current waypoint. The reporting points themselves are discovered, not authored — the minimap's height grid is flood-filled into connected landmasses, the largest are named, and naming tiers are proportional to the biggest landmass so the scheme survives any terrain. Positions are phrased "over X" or "four miles northwest of X" against the nearest one. Calls to the player are rate-limited per advisory type and push the ambient chatter back, so a message meant for you is not buried by routine traffic.
- **Audio** — WebAudio only. Three detuned oscillators through a lowpass make the engine, its pitch and cutoff following throttle and speed; ring chimes climb a semitone ladder; the crash is a filtered noise burst.

## Development

`window.SKYWARD` exposes the sim state (`state`, `plane`, `rings`, `nextRing`, `terrainHeight`, `respawn`, `newCourse`, `mapSpan`, `cycleMapZoom`, `traffic`, `updateTraffic`, `renderer`, `drawMap`) for poking at things from the console.

Two things worth knowing when testing from the console:

- Wait on `requestAnimationFrame`, not `setTimeout`, before reading HUD text — the render loop is what writes it, and an unfocused tab throttles frames, which makes the HUD look one step behind and can read as a bug that isn't there.
- For anything needing more than a few seconds of simulation, step the systems directly (`updateTraffic(1/60)` in a loop) instead of waiting on real frames. Ten simulated minutes of traffic runs in a fraction of a second and is deterministic.
- `updateFlight(1/60)` can be stepped the same way, which is how the landing cases are tested — a full approach runs instantly instead of taking fifteen real seconds. When a test result looks wrong, check `state.onGround` / `state.crashed` first: a "steering" measurement that accelerates every frame is usually the crash tumble, not the thing you meant to measure.
- Where a test must use real frames — anything going through `updateFlight`, which the loop drives — budget them from distance and speed, not by feel. At 60 fps and 200 units/s a frame advances about 3.4 units, so covering 200 units takes ~60 frames, not 45. Loop until the condition is met with a generous cap rather than a fixed frame count.

Two three.js gotchas this code deliberately works around:

- `Object3D.lookAt` aims an object's **+Z** at the target, but the aircraft's nose and the flight model's forward vector are **−Z**. Respawn computes the yaw explicitly instead.
- `lookAt` reads `matrixWorld`, which is stale until the next render — another reason not to use it right after moving an object.
