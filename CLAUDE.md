# Sleep Sounds

Read [PROJECT_BRIEF.md](PROJECT_BRIEF.md) first: requirements, architecture, testing and open items. Then `index.html`.

## Constraints for future changes

(Repeated from the brief.)

- Keep it a single `index.html` with inline CSS and JS, plus `manifest.json` and `sw.js`. No frameworks, no external dependencies, no audio files.
- Never make playback depend only on live Web Audio nodes.
- Keep the UI dim: no bright white, and large touch targets.

## Shipping changes

- Bump `CACHE` in `sw.js` whenever anything the app serves changes.
- Bump `VERSION` in the worker (`worker-src` block in `index.html`) whenever sound generation changes, or users keep old cached audio.
- Deployed via GitHub Pages from `main` / root. Playwright (for `dev/` tests) is dev tooling only and must never become a runtime dependency.
