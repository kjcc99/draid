# Devlog — kjcc99/draid

Working notes for this fork of [kaiyoti/open-dungeon-raid](https://github.com/kaiyoti/open-dungeon-raid).
Newest entries first. Each entry says what changed, why, how it was checked, and what is
still open, so a later session can pick up without re-deriving it.

## Status at a glance

| Item | State |
|---|---|
| Upstream base | `kaiyoti/open-dungeon-raid@e81efcf` (imported 2026-10-30 as `b97989e`, history not carried over) |
| Default branch | `master` (the Pages workflow triggers on `main` or `master`) |
| GitHub Pages | **Not live yet.** Runs #1 and #2 built fine but failed at deploy with a 404 because Pages was not enabled. Fix: *Settings → Pages → Source: GitHub Actions*, then *Actions → Deploy GitHub Pages → Run workflow*. |
| Expected URLs | Original <https://kjcc99.github.io/draid/> · HD <https://kjcc99.github.io/draid/hd/> |
| Tests | 18/18 `rebuild/web/test_*.js` pass; `test_u16.py` and `test_u17.py` pass (see *Testing* below) |
| Real-device check | **Not done.** All measurements so far are headless Chromium on a desktop CPU. |

---

## 2026-10-30 — Pre-shrink HD sprites, cache enemy number strips (`544657a`)

**Problem.** One board redraw (`draw()`) took about 4.7 ms of main-thread time with a chain held.
It runs on every `pointermove` and every animation frame. A CPU profile showed the time going to
the board tiles: each one was a high-quality *downscale* from the 4096×4096 HD atlas, on every
frame. The profile charged that time to `ctx.restore()`, because that is where the deferred
raster flushes. A micro-benchmark showed `save`/`restore` on its own costs nothing.

**Changes** (`rebuild/web/template.html`):
- `drawScaled()` (next to `hdSmooth`) shrinks each `(image, source rect, device-pixel size)`
  once into a canvas, then blits it 1:1. It keeps an LRU cache capped at about 24 MB. It only
  runs in the HD build (`ASSET_SCALE !== 1`), only when shrinking, and only under axis-aligned
  transforms. Anything else falls through to a plain `drawImage`. Used by `drawTile` for the
  tile sprite and the mortal-enemy glow, and by `drawOverlay` for frozen/flaming.
- `fontNumDraw()` reuses finished number strips through `fontStripCache`, keyed on all inputs,
  with x/y entering as offsets from the strip origin. It previously allocated a new canvas and
  context for every enemy number on every draw.

**Result.** `draw()` with a chain held went from 4.67 ms to 0.40 ms. Board pixels differ from
the previous build by a mean of 0.5/255, at tile edges only, from the different resample.
Number strips are pixel-identical.

**Not done.** HUD panels (`drawOriginalHud`, about 0.5 ms) still redraw on every `draw()` and
still downscale X7 patches by about 1.5×. Under 4× CPU throttle, frames remain slow in headless
mode, but nearly all of that is software compositing, so it is not representative of a phone GPU.

## 2026-10-30 — Smoother tracing and animation (`29eb1de`)

The user reported three problems: tracing "doesn't want to trace", occasional hiccups, and
animation that looked capped at 30 fps.

**Causes found and changes made** (`rebuild/web/template.html`):

1. **First-trace stall: 0.8 s on desktop, about 3.8 s at 4× throttle.** `arrowTintedAtlas()`
   tinted the *entire* HD `tiles.png` green (a 16.7 M-texel `getImageData` plus a per-pixel
   loop) the first time a chain arrow was drawn. Now it copies and tints only `ARROW_UV`
   (u 266–330, v 360–384 in 1× px, which includes a 4 px margin for bilinear sampling). The
   cropped canvas carries `_ox`/`_oy`, and `drawArrowFan` subtracts them from the UVs. The arrow
   is pixel-identical (screenshot diff = 0).
2. **Sound churn.** Every effect, including one per tile added while tracing, created a new
   `new Audio(url)` element. There is now a Web Audio path: `sfxUnlock` creates or resumes the
   `AudioContext` on the first pointerdown/keydown/touchend, `sfxPreload` decodes all 70
   variants once, and `sfxPlay` plays from an `AudioBuffer`. It falls back to `<audio>` until a
   buffer is ready, or when Web Audio or Vorbis is missing (the Node tests, or Safari without
   Vorbis).
3. **Strict hit zone.** `HIT_R` went from 0.36 to 0.45 cells. A straight diagonal still clears
   the side neighbours by about 0.26 cells.
4. **Dropped presses.** A `pointerdown` while `busy()` (falls, tokens, particles, flares,
   attack state) was ignored outright. Now it is stored in `pendingPress`, and
   `pendingPressTry()` starts the chain on the next pointermove once the board is idle, or at
   animation settle. The drag code was split into `beginDrag()` and `dragWalk()`.
5. **30 fps motion.** The timeline steps at a fixed 30 Hz (`FX.DT`, matching the original),
   and each frame drew the latest step. Render interpolation was added: `lerpMarkAll()` stores
   pre-step values at the top of `animStep()`, then `lerpApply(k)` and `lerpRestore()` wrap the
   draw calls in `animTick` and `idleLabelTick`. This covers falls (y), tokens (x/y), particles
   (x/y and timer), flares/labels (age) and shards (t). The step logic and timings are
   untouched, and so are the headless test path and its `animStep` loop.
6. `drawTintedSprite()`: cache hits now blit straight to the destination instead of resizing
   the shared `_tint` canvas, which reallocates it.

**Checks.** On falling tiles, the share of frames with new positions went from about 45% to
90%+. A press held through the animation started its chain once the board settled. All 70
sounds decoded in Chromium, with no `<audio>` elements created afterwards.

**Departures from the original game (deliberate, UX).** The `HIT_R` change and the
held-press start are page-side input choices, not reproductions of the original's decompiled
behaviour. The original *does* poll touch state every frame, so a held touch starting a chain
after the busy gate is at least plausible, but this was not verified against the binary. If
strict fidelity matters more, revert those two pieces only.

## 2026-10-30 — Import (`b97989e`)

Copied the tree of `kaiyoti/open-dungeon-raid@e81efcf` (the HD page is served from
`kaiyoti.github.io/open-dungeon-raid/hd/`) into this then-empty repo as a single commit.
Upstream's own `.odr-export` marker shows that tree was itself exported from a private
`dungeon-raid` workspace on 2026-10-28. Upstream comments reference `tasks/…`, `AGENTS.md`,
`CAPTURES.md`, decomp addresses and capture tools that are **not** in the public tree. Treat
those as citations, not as files to look for.

---

## Working notes

### Where things are
- Almost all web-game work happens in `rebuild/web/template.html`, about 11k lines of vanilla
  JS, CSS and HTML. `rebuild/web/build.sh` inlines it, together with the committed wasm glue
  `rebuild/web/dr.js`, into `demo_art.html` (1×) and `demo_art_hd.html` (HD,
  `ASSET_SCALE=4`). Both outputs are gitignored.
- The game rules live in the C++ core (`rebuild/src`, `rebuild/include`, `rebuild/web/api.cpp`),
  compiled to `dr.js`. Changes to `template.html` alone do **not** need a wasm rebuild.
- The code comments cite the original binary (`Function @0xaddr:line`) for every behaviour.
  The project's convention is that presentation changes are tagged "presentation only", and
  game logic stays in the engine.

### Building without Emscripten (template-only changes)
`build.sh` aborts if `em++` is missing, even though `dr.js` is committed. For template-only
work, put a no-op `em++` on `PATH`:
```bash
mkdir -p /tmp/fakebin && printf '#!/bin/sh\nexit 0\n' > /tmp/fakebin/em++ && chmod +x /tmp/fakebin/em++
PATH=/tmp/fakebin:$PATH ./rebuild/web/build.sh --all
```
For any change under `rebuild/src` or `api.cpp`, install emsdk 3.1.74 for real (see README).

### Testing
```bash
for t in rebuild/web/test_*.js; do node "$t" || exit 1; done            # 18 Node tests
pip install websockets                                                  # once
CHROME=/path/to/chrome python3 rebuild/web/test_u16.py                  # drag under help tip
CHROME=/path/to/chrome python3 rebuild/web/test_u17.py                  # pointer->cell mapping at zooms
```
In the Claude Code cloud container, Chrome lives at
`/opt/pw-browsers/chromium-1243/chrome-linux64/chrome`, and Playwright is preinstalled.

### Profiling recipe used so far
Serve the repo root (`python3 -m http.server 8765`) and drive
`rebuild/web/demo_art_hd.html` with Playwright:
- Click `#mmNew`, then call `setHelpOn(false); newGame(12345, 1, 0, 0)` for a deterministic board.
- Find a legal chain with `pathOk([...])`. Cell centres are at
  `rect.left + (c + 0.5 + FX.BOARD_LEFT/FX.CELL) * rect.width * FX.CELL`, and the same with
  `FX.BOARD_TOP` for y.
- Measure with a `requestAnimationFrame` timestamp log, a `longtask` PerformanceObserver, a CDP
  `Profiler` CPU profile, and `Emulation.setCPUThrottlingRate` for phone-ish speed.
Top-level `let`/`function` names (`api`, `anim`, `drag`, `draw`, `busy`, …) are reachable from
`page.evaluate`.

### Ideas not yet done
- Profile on a real phone (Android Chrome → `chrome://inspect`) before going further.
- Skip `drawOriginalHud()` when nothing in the HUD changed (it currently redraws on every `draw()`).
- Pre-shrink the HUD's X7 patches and the menu screens the same way as the tiles.
- Menu and screen fades still tick on fixed 33 ms timers (`setInterval(step, 33)` and similar),
  so they would need the same interpolation treatment to look smooth.
