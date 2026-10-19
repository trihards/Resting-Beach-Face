# Sleep Sounds: Project Brief

Handoff notes for anyone picking this project up, whether a person or an AI assistant in a fresh session. Read this first, then `index.html`.

## What it is

A sleep sound machine built as a single-page PWA (an installable web app). It runs on a phone all night with the screen locked and replaces a bedside sound machine while traveling. Its job is to mask hotel noise (hallway voices, doors, HVAC) and road rumble.

All audio is generated in code. There are no audio files, no frameworks and no external dependencies.

## Status

- **v1 is complete.** It works in desktop Chromium, tested headless, for both playback engines (see below).
- **Not yet tested on a real iPhone or Android phone.** The whole-night, locked-screen behaviour has not been checked on a device yet.
- **Not tested by ear.** The sound design was tuned from measurements (level, frequency balance, loop seams); nobody has listened to it yet. Expect to adjust it after the owner listens.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app: CSS, UI, playback engines, and the synthesis Web Worker (its source is held in a `<script type="text/plain" id="worker-src">` block and loaded as a Blob worker). |
| `manifest.json` | PWA manifest: standalone display, dark theme colours, icons. |
| `sw.js` | Service worker. Caches the app files and serves them cache-first, refreshing in the background. Bump `CACHE` when shipping changes. |
| `icon-192.png`, `icon-512.png`, `icon-512-maskable.png`, `apple-touch-icon.png` | Icons: a dim moon over blue and green waves. |
| `dev/` | Test scripts (Node + Playwright). Not used by the app. See "Testing". |

## Requirements (from the owner)

**Sounds.** Each has an on/off toggle and a volume slider, and they can be layered.
1. **Fan:** deep, steady brown noise, gently low-passed, with a very subtle slow wobble and hum.
2. **Air purifier:** smoother, slightly brighter pink noise with a mild band-pass.
3. **Rain:** a pink-noise base plus soft random droplets. Steady, no thunder.
4. **Shower:** brighter, denser spray with fast random modulation.
5. **Oregon rainforest:** canopy-muffled rain plus irregular heavier drips. Optional faint distant creek. Optional "dawn birds", very quiet and off by default.
6. **Ocean:** waves that swell, break and wash back out over 8–14 s cycles, with random timing and strength, over a continuous low surf rumble. Two styles: "Distant surf" (soft, deep) and "Shoreline" (closer, with foam and pebble wash).

Default mixes should lean toward low frequencies, because those mask road rumble best.

**Presets** (one tap; everything stays adjustable afterwards):
- Coast: Ocean (distant surf) + soft fan. This is the default on first launch.
- Pacific Northwest: Rainforest + creek + low fan.
- Shoreline Storm: Ocean (shoreline) + rain.
- Hotel Shield: Fan + purifier, loud.
- Rainy Night: Rain + fan.
- Steam: Shower + purifier.

Users can also save the current mix as a named custom preset and delete custom presets.

**All-night playback (critical).**
- Keep playing for 8+ hours with the screen locked or the app in the background.
- Don't rely only on live Web Audio nodes; phones suspend them when locked. Render each sound to a buffer, encode it as WAV, and play it through looping `<audio>` elements.
- Loops must be seamless with no repetition that draws attention: at least 60 s for steady sounds and at least 3 min for rainforest and ocean.
- Show a "Preparing sounds…" indicator while rendering, and cache the results so later launches are instant.
- Use the Media Session API to put the preset name and play/pause on the lock screen.
- Never stop on its own unless a timer was set.

**Controls.**
- One large Play/Pause button.
- 10 s fade-in on start; fade-out on stop.
- About 5 s crossfade when switching presets.
- Sleep timer: Off by default, plus 1h, 2h, 4h and 8h, ending with a 60 s fade-out.
- Master volume.
- Remember the mix, volumes and custom presets in localStorage.

**Design.**
- Near-black, dim and muted, with no bright white.
- Dim ocean-blue and moss-green accents.
- Big touch targets that are easy to hit half-asleep.
- A "Dim" mode that blacks out the screen until tapped.

**Acceptance criteria.**
- Plays continuously with the screen locked for 8+ hours.
- No audible clicks, gaps or noticeable repeats.
- Each sound is distinct and recognisable, and the ocean and rainforest sound natural.
- Presets load in one tap and crossfade smoothly.
- Works offline after the first load.

## Architecture

### Synthesis (worker section of `index.html`)

- **Sample rate:** `SR = 22050` (Nyquist 11 kHz), stereo. Each sound is a "layer":
  - 60 s layers: `fan`, `purifier`, `rain`, `shower`, `creek`.
  - 180 s layers: `forest`, `birds`, `oceanD` (distant surf), `oceanS` (shoreline).
- **One function per layer:** `synthFan`, `synthPurifier`, `synthRain`, `synthShower`, `synthForest`, `synthCreek`, `synthBirds`, `synthOcean(shore)`.
- **Building blocks:**
  - `BQ`: biquad filter.
  - `SVF`: state-variable filter, used where the cutoff is swept (the waves).
  - Noise generators: `pink`, `brown`, `white`.
  - `wander`: a smooth random modulator.
  - `unit()`: calibrates a generator chain to unit RMS so the mixing weights mean something.
  - `burst()`: a filtered noise tick (drops, pebbles).
  - `plop()`: a damped sine with a pitch glide (drips, bubbles).
- **Seamless loops:**
  - Continuous "beds" are rendered with extra length X, and the tail is crossfaded over the head with equal power (`loopify`).
  - Discrete events (drops, waves, birds) are written with modulo wraparound.
  - Periodic modulations use whole-number cycles per loop (e.g. the fan's 58 Hz hum and 11.5 Hz blade flutter over 60 s).
  - Ocean wave periods are rescaled so they add up to exactly 180 s.
- **Output and caching:** `finalize()` normalises each layer to `TARGET` (RMS; peak for birds), applies a soft limiter (`soft()`, above 0.8), and stores it as interleaved Int16. `load()` caches layers in IndexedDB (`sleep-sounds` DB, key `name@v{VERSION}@{SR}`). **Bump `VERSION` whenever synthesis changes**, or users keep the old cached audio.
- **Memory:** a full cache is about 90 MB. The worker keeps in memory only the layers the current mix uses.

### Mixing

- `mix(spec)` renders a WAV. Frame `i` plays position `(p0 + i)` on the shared 180 s timeline (`LOOP`).
- Each layer's gain holds at `from[]` for `pre` seconds, then moves to `to[]` over `ramp` seconds. The interpolation preserves power, so layers present in both mixes stay at a steady level.
- A full 180 s mix takes about 170 ms on a desktop.
- **Main-thread file types:**
  - `loopFile(g)`: a steady 180 s loop.
  - `introFile(g)`: a 10 s fade-in from silence.
  - A small cache (`cached`) holds up to 4 rendered files; blob URLs are freed with `release()`.

### Playback: two engines (`engine` is chosen around line 862)

iOS Safari ignores `HTMLMediaElement.volume`; it always reads 1. That drives the split:

- **`makeVolumeEngine()`:** used on Android and desktop.
  - Each mix is one 180 s loop in an `<audio loop>`. Fades are `el.volume` ramps (`setInterval` at 50 ms).
  - A mix change starts the new loop on a fresh element at an unrelated position and crossfades with equal power. The unrelated position keeps the two copies uncorrelated, which avoids comb filtering.
  - Master volume is applied live through `el.volume`.
- **`makeBakedEngine()`:** used on iOS, or anywhere volume is read-only. Force it with `?engine=baked`.
  - Two fixed `<audio>` elements are "unlocked" on the first tap by playing a silent WAV.
  - Master volume and all fades are baked into the audio.
  - A transition renders a short non-looping file T. T starts exactly where the current loop is (`p0 = current position`), holds the old gains for `pre`, then ramps.
  - `swapTo()` starts the other element at the matching position and pauses the old one.
  - `planHandoff()` then swaps to the steady loop file once the ramp is done. It is triggered by `timeupdate`, a 400 ms poll, or the `ended` event as a fallback.
  - If `play()` on the second element is refused in the background, `swapTo` reuses the current element instead.
- **Known limitation:** each swap can produce a very brief blip, roughly a 50 ms overlap of the same audio. It is inherent to this approach, since iOS gives no volume control. The engine also assumes iOS keeps running the page's code while audio plays in the background.

### Rest of the app

- **State:** `state = {sounds, master, preset}` is stored in localStorage as `ssm.state`; custom presets are `ssm.custom`.
- **Gains:** `gains(withMaster)` maps state to the 9 layer gains:
  - Volume curve: `1.3·(v/100)^1.5`.
  - Creek is 0.55× and birds 0.4× the rainforest gain.
  - Master curve: `(m/100)^1.5`.
- **Sleep timer:** checked every second and on each `timeupdate`. At deadline − 60 s it calls `engine.stop(60)`.
- **Media Session:** handlers for play, pause and stop. The metadata title is the preset name.
- **Interruptions:** if the OS pauses the audio (unplugged headphones, phone call), the UI simply shows paused. It doesn't auto-resume, deliberately, to avoid blasting the speaker. When the page becomes visible again and audio should be playing but isn't, a "Tap to resume" button appears.

## Testing

You need Node 18+ and Playwright with Chromium. Serve the folder over HTTP first: `python3 -m http.server 8765`.

- `node dev/synth-stats.js index.html [wavOutDir] [layer,layer]` runs the worker code in Node and prints, per layer: render time, RMS and peak, how often the limiter engages, the loop-seam jump compared with an average sample step (about 1× or less means seamless), band energy split, and loudness range.
  - Last measured: every seam was within normal sample-to-sample change; distant surf swings about 9 dB and shoreline about 13 dB.
- `node dev/e2e.js volume|baked [screenshotDir]` covers: play, fade-in, preset crossfade, toggle, saving a custom preset, timer fade-out, and that settings persist after a reload. It also lists console errors (currently none).
- `node dev/loop-wrap.js` checks that both engines keep playing across the loop point and for about 6 minutes at 16× speed.
- `node dev/offline.js` checks that after one online visit, an offline reload starts playback from the cache (measured at about 0.3 s).

## Deployment

It is a static site and needs HTTPS for the service worker:
- **GitHub Pages:** Settings → Pages → Deploy from branch → `main` / root.
- **Netlify Drop:** drag the folder onto app.netlify.com/drop.

To install on a phone:
- **iPhone:** Safari → Share → Add to Home Screen.
- **Android:** Chrome → ⋮ → Install app.

**Shipping changes:** bump `CACHE` in `sw.js`. Also bump `VERSION` in the worker if any sound changed.

## Open items and next steps

1. **Real-device overnight test**, on both iPhone (home-screen PWA) and Android Chrome. Lock the phone and confirm 8+ hours of playback, the lock-screen controls, and that the timer fade fires while locked.
2. **Listening pass and sound tuning:** fan character, rain droplet density, ocean realism, rainforest drips (the `plop` sine may sound too "bloopy"), and bird volume. Tuning constants live inside each `synth*` function and in `TARGET`.
3. **iOS transition blips:** if they're audible, options include longer `pre` with fewer swaps, or an experiment with `navigator.audioSession.type = 'playback'` plus Web Audio, with the `<audio>` path kept as a fallback.
4. **First launch on slower phones:** generating the ocean layer takes about 3–5 s on a desktop and could reach 10–15 s on a phone. Possible fixes: lower the cost of the `synthOcean` inner loop, or generate in chunks with progress shown.
5. **Nice-to-haves:** a volume "boost" range for very loud rooms; showing the remaining-time countdown on the lock screen; deleting custom presets with a long-press instead of the × button.

## Constraints for future changes

- Keep it a single `index.html` with inline CSS and JS, plus `manifest.json` and `sw.js`. No frameworks, no external dependencies, no audio files.
- Never make playback depend only on live Web Audio nodes.
- Keep the UI dim: no bright white, and large touch targets.
