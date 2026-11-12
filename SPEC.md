# Fall Line Yeti: Spec and Handoff

Version 2 (3D carving build), 2026-11-11

## 1. What it is

A 3D endless snowboard runner. The rider swipes between five lanes on an open slope, dodging obstacles while a yeti chases from behind. Each crash lets the yeti close the gap. When the gap hits zero the run ends.

- **Platform:** browser, desktop and phone
- **Stack:** plain JavaScript (ES6+), Three.js r128 (WebGL), HTML and CSS for menus and HUD, Web Audio for sound
- **Build step:** none. One HTML file.
- **Persistence:** `localStorage` key `fall-line-yeti.v1` (looks, difficulty, best scores, mute)

## 2. Files in this handoff

| File | What it is |
|---|---|
| `index.html` | Current 3D game. Open it in a browser to play. Needs internet once for Three.js and fonts. |
| `v1-2d-canvas.html` | The first version (2D canvas, three lanes). Reference only. |
| `SPEC.md` | This document. |

To run with a local server instead of double-clicking: `npx serve .` in this folder.

## 3. Controls

| Action | Keyboard | Touch or mouse |
|---|---|---|
| Change lane | Press Left/Right or A/D (one lane per press) | Swipe left/right (22 px) |
| Jump | Up, W, or Space | Swipe up (22 px) |
| Tuck | Down or S | Swipe down (22 px) |
| Fast-drop in air | Down while airborne | Swipe down while airborne |

One action per swipe. Lift your finger and swipe again for the next one.
| Pause | P or Esc | Pause button |

## 4. Core mechanics

### 4.1 Lane model (`stepRun`)

Five lanes, `LANE_W = 2.1` apart (x = -4.2 to 4.2). `G.lane` runs -2 to 2. `move(dir)` changes it by one.

1. Wanted sideways speed = `(laneX - x) * 10`, capped at `latMax * 1.5` (halved while stumbling).
2. `G.vx` eases toward it at 20/s on snow, 10/s in the air. One lane takes about 0.2 s.
3. Board angle `G.yaw` follows `vx / latMax` for the carve look.

- `latMax = 6 + speed * 0.16` (units/s).
- `G.skid = wanted vx - actual vx`. It drives the visual board over-rotation and the spray amount.
- Faster lane changes: raise the `10` gain and the `20` rate together, keeping rate about 2x gain or the glide overshoots.

### 4.2 Jump and tuck

- Jump: launch velocity 7.6, gravity 26. Peak about 1.1 units, airtime about 0.58 s.
- Tuck lasts 0.6 s. Tucking in the air sets downward velocity to -14 and tucks on landing.

### 4.3 Yeti gap (the health system)

`G.gap` runs 0 to 1. The HUD bar shows it.

- Soft hit (rock, log, gate, skier): gap drops by `soft`, speed x0.7.
- Hard hit (tree, snowcat): gap drops by `hard`, speed x0.45.
- Snowball: gap drops by `soft * 0.75`.
- After any hit: 0.6 s stumble, 0.85 s of invulnerability.
- Recovery: gap refills at `rec` per second, only while not stumbling and above 88% of target speed.
- Gap at 0 triggers the caught sequence (1.3 s), then game over.

### 4.4 Difficulty table (`DIFFS`)

| | Blue | Black | Double Black |
|---|---|---|---|
| Start speed `v0` | 20 | 23 | 27 |
| Top speed `vMax` | 34 | 42 | 50 |
| Speed gain per metre `ramp` | 0.008 | 0.012 | 0.015 |
| Soft hit cost | 0.26 | 0.36 | 0.50 |
| Hard hit cost | 0.48 | 0.62 | 1.00 (run over) |
| Gap recovery per second | 0.06 | 0.04 | 0.028 |
| Row spacing in seconds, start to top speed | 1.25 to 0.80 | 1.08 to 0.62 | 0.95 to 0.52 |
| Seconds between snowballs | 11 to 16 | 6 to 10 | 4 to 7 |
| Starting gap | 0.82 | 0.72 | 0.62 |

`prog()` is 0 to 1 progress from `v0` to `vMax`. Most scaling keys off it.

### 4.5 Obstacles

| Kind | Hit type | How to avoid | Collision half-width |
|---|---|---|---|
| Tree | Hard | Steer | 0.55 |
| Snowcat | Hard | Steer | 1.45 |
| Rock | Soft | Steer, or jump (height 0.6+) | 0.70 |
| Log | Soft | Steer, or jump (height 0.45+) | 1.50 |
| Gate | Soft | Steer, or tuck | 1.70 |
| Skier | Soft | Steer, or jump (height 1.0+). Drifts sideways as it nears. | 0.45 |

The rider adds 0.25 to each half-width. Collision is a swept check on z, so fast frames cannot skip an obstacle.

### 4.6 Row generator (`spawnRow`)

Rows spawn at z = -135 and scroll toward the rider at z = 0.

1. Move the safe line `G.pathX` to a lane, a random number of lanes away (at least 1 allowed, more if `rowTime * latMax * 0.55` covers them).
2. Place `2 + round(prog * 2.5) + coin flip` obstacles at random x, rejecting any that overlap the safe line (half-width 1.55 shrinking to 1.2) or each other.
3. With probability `0.10 + 0.42 * prog`, put a log, gate, or rock on the safe line itself, forcing a jump or tuck.
4. Either drop a power-up (every 520 to 900 m) or, 65% of the time, a line of five snowflakes tracing the safe line.

This guarantees one survivable line per row. It does not guarantee the line is comfortable.

### 4.7 Snowballs (`throwBall`)

The yeti throws at the rider's current lane. A red ring marks the landing spot for 1.1 s. Being within 1.0 unit at impact (and below jump height 1.0) counts as a hit.

### 4.8 Pickups

- **Snowflake:** +1 flake. Score = metres + flakes x 10.
- **Shield:** absorbs one hit.
- **Hot wax:** 3 s of 1.3x speed and invulnerability, and +0.3 gap.

## 5. Customization

Defined by `SCHEMA` and `DEF`. Adding an option means adding to the schema and handling it in `buildRider` or `buildYeti`.

- **Rider:** name, jacket, pants, headgear (beanie, helmet, hood), hat/mitts/scarf color, goggle lens, skin tone, board color, board graphic (solid, stripes, split), extra (none, backpack, scarf)
- **Yeti:** name, fur, eye color, headwear (none, horns, mohawk, earmuffs)

`applyLooks()` rebuilds both models after any change. Geometries and materials are cached (`GEO`, `MAT`), so rebuilding is cheap.

## 6. Code map (`index.html`, single script)

| Section | Key names | Notes |
|---|---|---|
| Helpers and options | `$`, `clamp`, `lerp`, palettes, `DEF`, `SCHEMA`, `DIFFS` | Data only |
| Saved state | `S`, `load`, `save` | All `localStorage` access is wrapped in try/catch |
| Sound | `tone`, `sfx` | Oscillator beeps, no audio files |
| Model builders | `MAKE`, `makeTree`, `buildRider`, `buildYeti`, `pose` | Primitives only, no imported models |
| Scene setup | `initGL`, `resize`, `setView` | Sky, fog, lights, slope, decor, trail, particles |
| Game state | `G` | One mutable object |
| Simulation | `stepRun`, `stepCaught`, `stepIdle`, `step` | Fixed max dt of 34 ms |
| Render | `render` | Three camera modes: chase, customizer, game over |
| Flow and UI | `setState`, `startRun`, `gameOver`, `updateHud`, `buildFields` | States: `menu`, `custom`, `run`, `pause`, `caught`, `over` |
| Input | `wire`, `jump`, `duck` | Pointer and key handlers |

### World conventions

- Rider stays at z = 0. The world scrolls toward +z. Forward is -z.
- 1 unit is about 1 metre. Rider is about 1.5 units tall.
- The `world` group is tilted -0.13 rad on x to make the slope read as downhill.
- Fog runs 55 to 132 units and hides the spawn line at -135.
- The customizer and game-over screens render the scene into the top 40% of the canvas using a viewport and scissor.

## 7. Known issues and untested areas

- **Not hand-played.** Only an automated smoke test (menu, customizer, a run, game over) has been run. Difficulty and feel need a real tuning pass.
- **Close-yeti camera unverified.** The yeti was moved 1.7 units left of the rider so it stops blocking the view. That framing has not been visually checked at very small gaps.
- **Customizer panel background** is `rgba(20,23,58,.99)` on purpose. A fully opaque scrolling panel over the WebGL canvas blanked the canvas in headless Chromium. Keep it slightly transparent unless you confirm it is safe.
- **Three.js r128** is the last version with a plain global script build on cdnjs. Upgrading means switching to ES modules and a bundler.
- **No real shadows.** Blob shadows only.
- **Flakes have no use yet** beyond score.
- **Single file.** About 56 KB. Fine now, awkward past a few thousand lines.

## 8. Suggested next steps

### Engineering

1. Move to Vite with ES modules. Suggested split: `config.js` (DIFFS, SCHEMA), `models.js`, `scene.js`, `sim.js`, `render.js`, `ui.js`, `input.js`, `audio.js`.
2. Separate simulation from rendering fully, so `sim.js` has no Three.js calls and can be unit tested (row generator reachability, collision).
3. Pool entities instead of creating groups per spawn.
4. Add a debug overlay: speed, `prog()`, gap, safe line.

### Game feel

1. Tune carving rates and `latMax` by hand on a phone and a keyboard.
2. Add tricks in the air (spin or grab) for bonus flakes.
3. Terrain variety: rollers, jumps, narrow chutes, a second biome after 2,000 m.
4. Yeti behaviors: swipe attack at close range, more than one throw pattern.

### Monetization hooks (from the plan)

1. Flake economy: persist total flakes, price the customizer options, lock most by default.
2. Rewarded-ad points: "continue once" on the game-over screen, "double flakes" after a run. Put both behind one `ads.js` interface so portal SDKs (CrazyGames, Poki) and mobile SDKs can swap in.
3. Analytics events: `run_start`, `run_end` (distance, cause, difficulty), `item_unlocked`, `ad_shown`.
4. Daily missions and a leaderboard for retention.
5. Mobile wrap with Capacitor only after web retention looks healthy.

## 9. Tuning cheat sheet

| Want | Change |
|---|---|
| Snappier lane changes | Raise the `10` gain and `20` rate in the lane block of `stepRun` |
| More lanes or wider lanes | `LANE_MAX` and `LANE_W` (keep `LANE_MAX * LANE_W` under `HALF = 5.3`) |
| Easier rows | Raise the safe half-width `lerp(1.55, 1.2, p)` in `spawnRow` |
| Fewer forced jumps | Lower `0.1 + 0.42 * p` in `spawnRow` |
| Slower ramp | Lower `ramp` in `DIFFS` |
| Kinder yeti | Lower `soft` and `hard`, raise `rec` in `DIFFS` |
