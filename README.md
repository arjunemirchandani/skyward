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
| `B` | New mission (asks first if a flight is underway) |
| `N` | Free gate course (same) |
| `M` | Mute |
| `Z` | Minimap zoom (2.5 / 5 / 10 km) |
| `SPACE` | Wheel brakes (on the ground) |
| `L` | Logbook |
| `H` / `I` | Hide the controls panel / the instruments |

## The game

### Missions

Press `B`. You are put on the runway at one field, lined up on the into-wind end, engine idling. Take off, fly the gate course — which is laid along the route to another field, so flying it takes you there — and land at the far end. One score out of 100, balanced three ways:

| | |
| --- | --- |
| Gates flown | 40 |
| Touchdown grade | 40 |
| Time against a par for the route | 20 |

Skipping gates to get down sooner is a legitimate choice, not an exploit: you can go straight to the destination and still complete the mission, but the gates you dropped cost more than the time they saved. Landing at the wrong field is just a landing, and Center will remind you where you were meant to be going.

### Logbook

Press `L`. `R` from inside the logbook wipes the career — it asks first, and press `R` again to go through with it. Every flight is recorded and kept between sessions: hours in the air, landings and their average grade, gates flown, crashes, fields visited, and your best score on each route. While you are flying a route you have flown before, the mission panel shows *that route's* best rather than your overall best — the number you are actually chasing — and Center reminds you of it on the runway.

### Free flight

Press `N` for the original 12-gate course from wherever you happen to be, with no field to land at and your best time kept separately. Nothing about missions takes that away.

### The gate course

Fly through all 12 gold rings in order, as fast as you can. The next ring is gold and pulsing; the others are cyan. An on-screen marker tracks the active ring and pins itself to the screen edge when it is behind you. Your best time is kept in `localStorage`. Finishing generates a fresh course.

Crashing into terrain or water puts you back on approach to the ring you were heading for, so a mistake costs seconds rather than the run.

You are not alone up there. Forty-six aircraft fly their own routes across the islands. When one is close, near your level, and actually closing on you, the HUD reads `TRAFFIC`, the contact turns red on the minimap, and a blip sounds faster the closer it gets. Hit one and it is a mid-air.

### Landing

Three airfields are scattered across the islands, marked in amber on the minimap. Fly within 8 km of one and an approach panel appears with course and glidepath needles — each needle shows where the beam is, so you steer toward it. Get established on final and Center clears you to land.

Along the bottom of the screen are the three instruments that matter when you cannot see out: an artificial horizon for pitch and bank, a vertical speed indicator, and a turn coordinator. In clear air they are a convenience. In cloud they are the only thing telling you which way is up.

The weather changes. Conditions drift between clear, scattered, overcast, rain and fog — and they move together, so thicker cloud means a lower ceiling, poorer visibility and flatter light. Cloud forms a deck at the ceiling that you climb through: inside it there is nothing to see at all, and the approach needles are the only way down. Center gives you the conditions with your clearance and calls you when you are in cloud on final.

The air is moving. Wind drifts you off the centreline unless you correct, gusts and turbulence knock you about low over broken ground, and each field has a windsock plus a crosswind readout on the approach panel. Center reports the wind with your landing clearance and tells you when the other end of the runway would be better. Watch your shadow on the runway in the flare — it is the best cue you have for how high you actually are.

Touchdowns are scored out of 100 on four things: sink rate, how close to the centreline, how well aligned with the runway, and where in the touchdown zone you put it. You get a grade, and your best is kept. Arrive too hard, too crossed-up, or too banked and it is an accident instead. On the ground, steer with `A`/`D`, brake with `SPACE`, and pull back once you have speed to fly again.

They are all on the radio, and so are you — your callsign is `REDTAIL ONE`. Skyward Center clears you for the gate course, calls your progress, warns you about terrain and traffic (with a clock position), and notices when you go off the scope. The islands they all report against are named on the minimap.

## How it works

- **Terrain** — value-noise fBm heightfield (`terrainHeight`), 300×300 grid over 14 km. A continent layer decides land vs. ocean, a ridged layer adds mountains only where there is already land, and vertex colours come from height and slope. The same function is sampled directly for collision, so the physics and the mesh can never disagree.
- **Flight model** — arcade, not aerodynamic. Speed chases a throttle-driven target and trades against climb angle (`speed -= forward.y * GRAVITY * dt`), control authority falls off with airspeed, banking couples into yaw for coordinated turns, and below stall speed the nose drops and the plane sinks. Wings self-level when the stick is centred and the plane is upright.
- **Ring detection** — each frame the previous and current positions are transformed into ring-local space; a sign change in local `z` plus a radial hit inside the torus radius counts as a pass. Frame-rate independent, no tunnelling.
- **Minimap** — the whole world is baked once into a 320×320 offscreen canvas: land colours from the same palette as the 3D terrain, depth-shaded ocean, and a hillshade computed from central differences on the height grid (no extra noise sampling). Each frame the map blits a player-centred, north-up window out of that bake and overlays the ring course. `drawImage` clips a source rectangle that runs off the bake, so the destination rectangle is clipped to match — otherwise the map would smear near the world edge.
- **Traffic** — each aircraft is one merged geometry (a single draw call) flown by a waypoint autopilot: limited turn rate, bank proportional to turn rate, pitch derived from climb rate. The interesting part is altitude. Short-range lookahead cannot save an aircraft from a 1500 m peak — by the time the peak is in range there is no distance left to climb. So the *whole leg* is sampled when a waypoint is chosen, candidate legs whose terrain the aircraft could not out-climb are rejected outright, and a hard floor clamp guarantees nothing is ever seen inside a mountain. Over ten simulated minutes the clamp engages on 0.016% of samples; routing does the rest. Traffic streams around the player — anything beyond 9 km is recycled to a fresh bearing 3.2–6.2 km away, chosen by retrying bearings rather than clamping to the world bounds, since clamping drags the spawn point toward the player and pops an aircraft in at close range.
- **Not losing a flight to a stray keypress** — `B` and `N` both discard whatever you are in the middle of, which is easy to hit by accident. They are guarded, but only when there is something to lose: airborne on a mission, or partway through a free course. On the runway, or once a mission is finished, restarting costs nothing and happens immediately — a confirmation you always have to answer is just a second keystroke. Cancelling passes the key through to its normal job, so pressing `W` to back out still pitches.
- **Missions** — the spine that ties the rest together. Before this, gates and landings were two unrelated activities; you never flew gates *to* anywhere. A mission is a phase machine (`ready → enroute → arrival → complete`) over the systems that already existed, and the only genuinely new piece is `buildRouteCourse`, which lays gates along the track between two fields — swinging either side of the direct line — instead of wandering at random. Because they are placed along the route, flying the gates *is* the navigation. The clock starts when the wheels leave the ground, not on the runway. Respawning mid-mission puts you back on approach to whatever you were heading for: the next gate during the gate phase, the destination field once they are done.
- **Airfields** — there is no flat ground in a noise heightfield, so the terrain is made flat where it needs to be. `baseTerrain` is the raw landscape; sites are chosen from it by scanning for low, even coastal ground and orienting each strip along its flattest bearing. `terrainHeight` then blends those pads into the landscape with a smoothstep ramp. Because every system — the mesh, collision, the minimap bake, island finding, traffic routing — reads `terrainHeight`, levelling the ground in one function is enough to make the whole world agree; the strips measure dead flat and traffic still clears the terrain everywhere. Runway markings are painted into a canvas in runway-local units, with each designator placed at the threshold a pilot crosses when landing on it.
- **Seeing the traffic** — "I never see anyone" turned out to be a visibility problem, not a count problem. Measured against the actual projection, aircraft sat 2.2–7.1 km away and rendered 1.2 to 3.8 pixels across; at 7 km an airframe covered **zero** visible pixels once antialiasing was done with it. Three changes fixed it: a larger fleet on a tighter respawn ring, respawns weighted toward the arc the player is flying into and toward their altitude band, and a distance glint — a point sprite per aircraft that fades in exactly as the mesh becomes sub-pixel and out again with the fog, all in a single draw call. Density is cheap: the whole frame, 46 aircraft included, costs 0.37 ms, and 250 aircraft measured 3.5% of a 60 fps budget. Draw calls barely move with fleet size, because off-screen aircraft are frustum-culled and liveries share materials.
- **Conflict alerting** — a busy sky broke the traffic warning: on pure proximity it was lit 52% of the time, which is the same as not having it. A contact is now a threat only if it is close, within 200 m vertically, *and* the relative velocity says it is closing. That put the alert back to about 11% of flight time, where it means something again.
- **Resetting the logbook** — `R` is respawn in flight, so it only means reset while the logbook is open; the confirmation reuses the same press-again mechanism as abandoning a mission. It clears the three standalone bests too, not just the flight list — a book reading zero flights beside a personal best with nothing behind it is worse than not resetting at all. The prompt needed a `z-index` above the panel, or the question asking whether to erase the logbook rendered *behind* the logbook.
- **Logbook** — the game had depth but no memory: a perfect flight in fog with a crosswind left nothing behind but one number being overwritten. Totals accumulate forever while the flight list is capped at 60 entries, so storage stays bounded without losing the career. Loading is defensive on purpose — a corrupt or hand-edited file costs you your history and nothing else, verified by feeding it truncated JSON and confirming the game still starts and flies. Airborne seconds accumulate every frame but are flushed to storage on a 20-second timer rather than on each one. The detail that makes it more than a stats screen is per-route bests: chasing "your best on this route" is a target you can actually beat, where a single global best stops being reachable after a good day.
- **Instruments** — added because the weather created a gap rather than to decorate the screen: once you are inside cloud, the only attitude reference left is the aircraft model in chase view, and in cockpit view there is nothing at all. Bank comes from `atan2(-right.y, up.y)` and pitch from `asin(fwd.y)`, both taken straight off the orientation, so they cannot disagree with the aircraft — verified against known attitudes including that a 90° yaw registers zero pitch and bank. The vertical speed scale is deliberately compressed (`|fpm/6000|^0.6`): a linear dial either pegs constantly, because this flight model climbs at thousands of feet per minute, or is unreadable at the 300 fpm that matters on approach. The compressed one gives *more* deflection near zero and still fits a full climb-out. Both needles are damped towards their targets, which is what makes them read as instruments rather than raw telemetry.
- **Weather** — built to make instruments matter rather than to add scenery. One slowly-drifting master variable drives cover, visibility, ceiling and rain together, so conditions are always coherent: there is no clear sky with a 300 m ceiling. Fog distance and colour, sky tint, sun intensity and even the sea colour all follow it — a storm sky over bright blue water reads as two different days, so the water desaturates too. Clouds form a deck at the ceiling and stream around the player, and inside the deck fog collapses to 120 m. The sky dome renders unfogged, so it has to be whited out to match, or a horizon stays visible through what should be solid cloud. Verified landable in every condition down to 1.2 km visibility — harder, never impossible. Rain is camera-local line segments in a single draw call: drops fall and recycle in a small box around the eye instead of being advected through the world, which is what makes it read at 300 units/s as well as at a standstill. Worst-case frame with a full deck and heavy rain measured 1.9 ms, about 11% of a 60 fps budget.
- **Wind** — modelled as a moving airmass rather than a force on the aircraft: `state.speed` is airspeed, and the ground track is that plus the wind vector. Everything else follows from that one line. Drift on final, the need to crab, and the fact that traffic is carried along too all come free. The nicest consequence was not designed: because the alignment component of the landing score is measured at touchdown, the real technique — crab down final, then straighten in the flare — is what scores best. Measured on a 14-unit crosswind: crab held all the way **B 80**, de-crab at 12 m **B 85**, de-crab at 6 m **A 92**.
- **Turbulence** — intensity comes from wind strength, height above ground, and the roughness of the terrain underneath, so ridge flying in a gale is rough and a cruise is smooth. It is eased off in the last few metres deliberately: real mechanical turbulence is worst near the surface, but making the flare a lottery is not fun. A useful accident of siting airfields on flat ground is that approaches are naturally about a third as turbulent as ridge flying, with no special case for it.
- **Shadows** — the world is 14 km across, so one fixed shadow camera would be worthless. Instead a 2048² box follows the aircraft and tightens with height: at 30 m the airframe covers roughly 45 shadow texels. That is not decoration — the shadow closing on the wheels is the depth cue that tells you when to flare. Cost, measured: about 0.2 ms.
- **Landing score** — four independent components, each a taper between a "good" and a "bad" value: sink rate (40), centreline offset (20), alignment (20) and touchdown zone (20). Either runway direction is a legitimate landing; the judge picks whichever end you are flying toward and measures against that threshold. Outside the survivable envelope it is a crash, not a low score.
- **Radio chatter** — every transmission is generated from live simulation state, never from a canned script: an aircraft's reported altitude is its actual altitude, "climbing" means its climb rate is positive, and "direct MARROW ISLE" names the reporting point nearest its current waypoint. The reporting points themselves are discovered, not authored — the minimap's height grid is flood-filled into connected landmasses, the largest are named, and naming tiers are proportional to the biggest landmass so the scheme survives any terrain. Positions are phrased "over X" or "four miles northwest of X" against the nearest one. Calls to the player are rate-limited per advisory type and push the ambient chatter back, so a message meant for you is not buried by routine traffic.
- **Audio** — WebAudio only. Three detuned oscillators through a lowpass make the engine, its pitch and cutoff following throttle and speed; ring chimes climb a semitone ladder; the crash is a filtered noise burst.

## Development

`window.SKYWARD` exposes the sim state (`state`, `plane`, `rings`, `nextRing`, `terrainHeight`, `respawn`, `newCourse`, `mapSpan`, `cycleMapZoom`, `traffic`, `setTrafficCount`, `updateTraffic`, `updateFlight`, `airfields`, `judgeLanding`, `approachInfo`, `renderer`, `drawMap`) for poking at things from the console.

`setTrafficCount(n)` resizes the fleet live, which is how the cost curve above was measured — worth re-running before assuming a number is too expensive.

`weather.hold = true` pins the current sky so a specific condition can be reproduced — without it the render loop overwrites any value you set on the next frame.

**Measuring GPU cost.** `renderer.render()` only queues work, so timing it alone measures submission, not rendering — it under-reports by roughly 4×. `gl.finish()` is *not* a reliable sync point either; on this machine it returned times that implied a 180k-triangle shadow pass cost 0.015 ms, which is nonsense. Use a 1×1 `gl.readPixels()` after each render as the sync primitive. Measured that way the whole frame is about 1.2 ms, of which shadows are 0.2 ms — roughly 7% of a 60 fps budget. Numbers are viewpoint-dependent; measure from a high cruise with the archipelago in view, not a runway close-up.

Two things worth knowing when testing from the console:

- Wait on `requestAnimationFrame`, not `setTimeout`, before reading HUD text — the render loop is what writes it, and an unfocused tab throttles frames, which makes the HUD look one step behind and can read as a bug that isn't there.
- For anything needing more than a few seconds of simulation, step the systems directly (`updateTraffic(1/60)` in a loop) instead of waiting on real frames. Ten simulated minutes of traffic runs in a fraction of a second and is deterministic.
- `updateFlight(1/60)` can be stepped the same way, which is how the landing cases are tested — a full approach runs instantly instead of taking fifteen real seconds. When a test result looks wrong, check `state.onGround` / `state.crashed` first: a "steering" measurement that accelerates every frame is usually the crash tumble, not the thing you meant to measure.
- Where a test must use real frames — anything going through `updateFlight`, which the loop drives — budget them from distance and speed, not by feel. At 60 fps and 200 units/s a frame advances about 3.4 units, so covering 200 units takes ~60 frames, not 45. Loop until the condition is met with a generous cap rather than a fixed frame count.

Two three.js gotchas this code deliberately works around:

- `Object3D.lookAt` aims an object's **+Z** at the target, but the aircraft's nose and the flight model's forward vector are **−Z**. Respawn computes the yaw explicitly instead.
- `lookAt` reads `matrixWorld`, which is stale until the next render — another reason not to use it right after moving an object.
