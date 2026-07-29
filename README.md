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

## The game

Fly through all 12 gold rings in order, as fast as you can. The next ring is gold and pulsing; the others are cyan. An on-screen marker tracks the active ring and pins itself to the screen edge when it is behind you. Your best time is kept in `localStorage`. Finishing generates a fresh course.

Crashing into terrain or water puts you back on approach to the ring you were heading for, so a mistake costs seconds rather than the run.

## How it works

- **Terrain** — value-noise fBm heightfield (`terrainHeight`), 300×300 grid over 14 km. A continent layer decides land vs. ocean, a ridged layer adds mountains only where there is already land, and vertex colours come from height and slope. The same function is sampled directly for collision, so the physics and the mesh can never disagree.
- **Flight model** — arcade, not aerodynamic. Speed chases a throttle-driven target and trades against climb angle (`speed -= forward.y * GRAVITY * dt`), control authority falls off with airspeed, banking couples into yaw for coordinated turns, and below stall speed the nose drops and the plane sinks. Wings self-level when the stick is centred and the plane is upright.
- **Ring detection** — each frame the previous and current positions are transformed into ring-local space; a sign change in local `z` plus a radial hit inside the torus radius counts as a pass. Frame-rate independent, no tunnelling.
- **Audio** — WebAudio only. Three detuned oscillators through a lowpass make the engine, its pitch and cutoff following throttle and speed; ring chimes climb a semitone ladder; the crash is a filtered noise burst.

## Development

`window.SKYWARD` exposes the sim state (`state`, `plane`, `rings`, `nextRing`, `terrainHeight`, `respawn`, `newCourse`) for poking at things from the console.

Two three.js gotchas this code deliberately works around:

- `Object3D.lookAt` aims an object's **+Z** at the target, but the aircraft's nose and the flight model's forward vector are **−Z**. Respawn computes the yaw explicitly instead.
- `lookAt` reads `matrixWorld`, which is stale until the next render — another reason not to use it right after moving an object.
