# Poly-tract

> Write in the air. A premium, gesture-driven whiteboard that turns your webcam into a touchless drawing surface — built entirely in the browser, no installs, no servers, no accounts.

---

## What this is

Poly-tract is a single-page, camera-powered whiteboard. You point your webcam at your hand, and your index finger becomes a pen. Pinch to draw perfect shapes. Make a fist to erase. Open palm to pause. Everything runs locally in the browser — the camera feed never leaves the device, the hand tracking happens in WebAssembly, and the canvas state lives in memory.

It is not a productivity tool. It is a study in how natural a touchless interface can feel when the latency, filtering, and gesture model are tuned correctly.

---

## Why we built it

Touchscreens won. So did styli. But both assume you are holding something, touching something, or both. The moment you are cooking, painting, fixing a sink, or just have dirty hands, the interaction model breaks.

Poly-tract asks a smaller question: *what if the camera above your laptop was already a perfect input device, and the only thing missing was the software to make it feel good?*

Most "air drawing" demos fail in three predictable ways:

1. **Latency** — the cursor lags behind the finger by 100-300ms because the hand-tracking pipeline is naive.
2. **Jitter** — the drawn line looks like a seismograph because the raw landmark coordinates are noisy.
3. **Gesture ambiguity** — the app cannot tell "I am drawing" from "I am resting my hand", so it draws garbage when you did not mean to.

We treated those three problems as the actual product. The drawing features are secondary.

---

## How we got here

### v1 — Proof of concept

A single HTML file. MediaPipe Hands loaded from CDN, three canvas layers (skeleton overlay, committed ink, live preview), a One Euro filter on the index fingertip, and a quadratic-curve smoother. Black, dark-grey, and light-grey inks only. Three gestures: index to draw, peace to "convert to shape" (which was really just convex decomposition via poly-decomp), palm to clear.

It worked. It was also visibly laggy and the shape conversion was fake — it did not recognize shapes, it just split the stroke into convex polygons and called them shapes.

### v2 — The rebuild

We added the seven rainbow colors plus black, a thickness slider, pen/pencil/marker/eraser tools, perfect-freehand integration for buttery variable-width strokes, a settings panel, an accent color system (crimson, navy, emerald, amber, violet, teal, sunset, slate) applied as gradients and outlines rather than flooded fills, real shape recognition (circle, ellipse, rect, triangle, star, line, arrow, path) with a confirmation popup, and a full gesture map: index = draw, peace = select, pinch = grab/move, fist = eraser, palm = pause, three fingers = undo, shaka = redo, rock = fullscreen.

It was too much. The dock was crowded, the gesture set was hard to remember, and the popup appeared after every stroke.

### v3 — Simplification

We cut hard. Palette became a single horizontal row of eight dots. Tools dropped from four to two (pen + eraser). Gestures dropped from nine to five. Settings trimmed. One Euro filter retuned for lower lag (`minCutoff` 1.6 → 1.0, `beta` 0.012 → 0.02). Finger-up thresholds relaxed. Pinch thresholds widened. Gesture voting cut from 3-4 frames to 2-3. The result felt like the cursor was glued to your fingertip.

### v4 — Where we are now

Three big changes:

1. **Shape recognition actually works.** The classifier now properly detects hearts (emoji-style, two bumps + valley + bottom point), circles, ellipses, rectangles, triangles, stars, lines, and paths. Confidence is high enough that we no longer need a confirmation popup for every stroke. Drop poly-decomp — it was the wrong tool.

2. **Pinch became a drawing mode, not just a grab.** Pinch with nothing selected = auto-shape draw. You draw with your thumb-index midpoint, release the pinch, and the stroke snaps to its closest perfect shape. No popup. Index-finger drawing stays as raw ink unless the user opts into auto-shape-everything via the one-time first-stroke prompt.

3. **Select + pinch = convert.** Draw with index (raw ink) → peace sign to select → pinch → the selected stroke snaps to its recognized vector shape. This matches the mental model: "I drew something rough, now I want it clean."

4. **Gesture onboarding.** First launch shows a "Make it yours" screen listing every action and its currently-bound gesture. Tap any gesture button, make a new pose, and the binding swaps. Reachable later from Settings → Customize gestures.

---

## How it works

### Pipeline

```
Webcam → MediaPipe Hands (WASM) → 21 landmarks per frame
  → One Euro filter (per-axis, low-lag)
  → Gesture classifier (finger-up tests + pinch distance)
  → Action dispatcher (gesture → action via user-customizable map)
  → Ink engine (perfect-freehand) OR shape recognizer
  → Canvas compositor (hand skeleton / ink / live preview)
```

### Gesture model

| Gesture | Default action | Notes |
|---|---|---|
| Index finger up | Draw | Raw ink, kept as-is |
| Pinch (thumb + index touch) | Auto-shape draw | Snaps to recognized shape on release |
| Peace (index + middle) | Select | Hit-tests strokes and vector shapes |
| Fist | Erase | Deletes whole strokes on contact |
| Open palm | Pause | Cursor disappears, hand can rest |
| Three fingers | Undo | One-shot, 800ms debounce |
| Thumb up | Redo | One-shot, 800ms debounce |
| Rock (index + pinky) | Fullscreen | One-shot, 1200ms debounce |

All bindings are user-customizable via the onboarding screen or Settings → Customize gestures.

### Shape recognition

The classifier runs this pipeline on a finished stroke:

1. **Rammer–Douglas–Peucker simplification** (epsilon ~3.5px) to remove jitter.
2. **Closed-ness test** — if first-to-last point distance is less than 20% of bounding-box diagonal, treat as closed.
3. **Heart detection** — looks for two top bumps, a center valley lower than the bumps, and a bottom point near the horizontal center. If matched, returns an emoji-style 10-point heart polygon.
4. **Circularity** = `4π·area / perimeter²`. If > 0.82 and corner count > 6, returns circle or ellipse based on aspect ratio.
5. **Corner count** via simplified-polygon vertex count. 3 corners → triangle. 4-5 corners + high solidity → rectangle.
6. **Solidity** = stroke-area / convex-hull-area. If < 0.75 and corners >= 8, returns a 5-pointed star.
7. **Fallback** — open strokes become lines (if simple) or smoothed Catmull-Rom paths.

### Ink engine

Strokes are stored as point arrays with per-point width and pressure. Live preview and committed strokes both render through `perfect-freehand`'s `getStroke()` with `simulatePressure: false` (we feed it our own velocity-derived pressure). Velocity modulation gives strokes a natural taper — fast moves thin out, slow moves stay thick.

Fallback: if perfect-freehand fails to load from CDN, a quadratic-curve smoother takes over. Worse-looking, but functional.

### Filtering

The One Euro filter is the secret sauce. It is a low-pass filter with an adaptive cutoff — when the finger moves slowly, the cutoff drops and the output is buttery smooth; when the finger moves fast, the cutoff rises and the output tracks tightly with minimal lag. We run independent filters on X and Y with `minCutoff = 1.0` and `beta = 0.02`, retuned from the library defaults for this specific use case.

The "Steadiness" slider in Settings exposes `minCutoff` to the user. Lower = smoother but laggier. Higher = snappier but jitterier.

### Persistence

- `localStorage['poly-tract']` stores: color, thickness, tool, accent, autoShapeIndex, showSkeleton, sensitivity, gestureMap, hasSeenOnboard, hasSeenFirstStroke.
- Canvas state is in-memory only — refresh loses drawings. Intentional: this is a scratchpad, not a document editor.

---

## Tech stack

- **Hand tracking:** MediaPipe Hands 0.4.16 (WASM, CDN-loaded)
- **Stroke rendering:** perfect-freehand 1.2.0 (CDN)
- **Filtering:** One Euro filter (vanilla JS implementation, ~30 lines)
- **Shape recognition:** Vanilla JS (~150 lines: RDP, polygon area, convex hull, classifier)
- **Canvas:** Three stacked 2D canvas elements (hand skeleton / committed ink / live preview)
- **Camera:** `getUserMedia` with `facingMode: user`, 960×540 @ 30fps target
- **Fonts:** Figtree from Google Fonts
- **No build step.** No bundler. No framework. No dependencies installed locally. The entire app is one `index.html`.

---

## Browser support

- **Chrome / Edge desktop** — fully supported, best performance
- **Firefox desktop** — supported, slight tracking latency on Linux
- **Safari desktop** — supported on 14+, requires `https` or `localhost` for camera access
- **Chrome Android** — works but hand tracking is CPU-heavy; expect 15-20fps on mid-range phones
- **Safari iOS** — `requestVideoFrameCallback` not supported; falls back to `requestAnimationFrame` pump

Camera access requires HTTPS or localhost. Vercel deployment handles this automatically.

---

## Performance characteristics

On a 2020 MacBook Air (M1):

- **Hand tracking latency:** 40-60ms end-to-end (camera → landmarks → cursor position)
- **Filter lag:** 30-50ms additional, depending on `minCutoff` setting
- **Total cursor-to-finger latency:** ~80-110ms, perceptible but acceptable
- **Stroke render:** <2ms per frame for typical 200-point strokes
- **Memory:** ~80MB with camera on, ~40MB with camera off

On lower-end hardware (e.g. Chromebook):

- Hand tracking latency balloons to 150-250ms
- Recommend lowering `modelComplexity` to 0 in the MediaPipe options (tradeoff: less accurate landmarks)

---

## File structure

```
.
├── index.html          # The entire app (~900 lines: HTML + CSS + JS in one IIFE)
└── README.md           # This file
```

That is the whole project. No `src/`, no `dist/`, no `package.json`, no `node_modules`. Clone, open in a browser, done. Deploy by pushing `index.html` to any static host.

---

## Roadmap

### Near-term (next 2-4 weeks)

- **Two-hand support.** MediaPipe can track two hands; we currently cap at one for simplicity. Two hands enables pinch-to-zoom, two-finger pan, and simultaneous draw + erase.
- **Rotation gesture.** Pinch + wrist twist should rotate the selected shape. Implementation: track wrist angle (landmark 0 → 9 vector) during pinch-attach state.
- **Persistent canvas.** Optional localStorage save of canvas state (as vector items + serialized strokes) so refreshes do not lose work.
- **Export.** PNG export of the current canvas, plus SVG export of vector shapes only.

### Mid-term (1-3 months)

- **Multi-page.** A slide metaphor — swipe with three fingers to move between canvases. Each page is a separate items array.
- **Color picker.** Beyond the 8 swatches — a HSL wheel accessible from the dock.
- **Pressure simulation from finger depth.** Use the Z-coordinate of the index tip (relative to the wrist) as a fake pressure signal. Push forward = thicker stroke.
- **Custom shape library.** User draws a shape, labels it, and it becomes a snap-target for future strokes. "I drew my logo once, now every time I draw something close to it, it snaps to my logo."

### Long-term (3-6 months)

- **Multiplayer.** WebRTC peer-to-peer canvas sharing. Each peer's cursor is rendered on the other's canvas. Lock contention handled last-write-wins per-item.
- **Shape boolean operations.** Union, intersect, subtract on vector shapes. Lets users combine two circles into a Venn diagram, punch a hole in a rectangle, etc.
- **Mobile-native wrapper.** Capacitor or Tauri shell for App Store / Play Store distribution. Same codebase, native camera permissions, push notifications for shared canvases.
- **Voice commands.** "Undo", "clear board", "switch to red" — via the Web Speech API. Useful when your hands are already busy drawing.

### Things we will probably never do

- **Account system / cloud sync.** Poly-tract is a scratchpad, not a document store. If you want to save something, export it.
- **AI shape generation.** "Draw a cat" → AI generates a cat. That is a different product. Poly-tract is about *your* hand, not a model's imagination.
- **Touch input as primary.** Touch is supported as a fallback, but the entire UX is designed around the camera. Touch-first would require a different layout.

---

## Known issues

- **Pinch can trigger accidentally** during fast index-finger drawing if the thumb drifts close to the index. Mitigated by the 0.32× hand-size threshold and the gesture-voting cooldown, but not eliminated. Tune in `classify()` if needed.
- **Heart recognition has false negatives** on small hearts (<40px tall). The valley/bump detection needs a minimum feature size to be reliable.
- **Safari iOS** has no `requestVideoFrameCallback`, so the pump loop falls back to `requestAnimationFrame`. This works but burns more battery.
- **perfect-freehand CDN failure** silently falls back to the quadratic smoother. The visual difference is noticeable but the app stays usable.
- **Vercel auto-deploy sometimes fails** to trigger from GitHub pushes — see `TROUBLESHOOTING.md` (or just redeploy manually from the Vercel dashboard).

---

## Local development

```bash
# Clone
git clone <your-repo-url> poly-tract
cd poly-tract

# Open in browser — that's it
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows

# Or serve locally for proper HTTPS-like behavior
python3 -m http.server 8000
# then visit http://localhost:8000
```

Camera access requires either `https://` or `http://localhost`. Opening `index.html` as a `file://` URL will work for the UI but the camera will be blocked by the browser.

---

## Deployment

### Vercel (recommended)

1. Push `index.html` to a GitHub repo.
2. In Vercel: New Project → import the repo.
3. Framework preset: **Other**. Build command: *(none)*. Output directory: `./`.
4. Deploy.

Subsequent pushes to your production branch auto-deploy. If auto-deploy ever stops working, check the GitHub webhooks page (`repo/settings/hooks`) and the Vercel project's Git settings.

### Netlify / Cloudflare Pages / GitHub Pages

Same idea — any static host works. No build step needed.

### Self-hosted

Just serve `index.html` over HTTPS. Nginx, Caddy, whatever. The only requirement is HTTPS for camera access.

---

## License

MIT. Do what you want. Attribution appreciated but not required.

---

## Credits

- **MediaPipe Hands** — Google Research. The hand-tracking model that makes this possible.
- **perfect-freehand** — Steve Schoger. The stroke-rendering library that makes the ink look good.
- **One Euro filter** — Géry Casiez, Nicolas Roussel. The filtering algorithm that makes the cursor feel good.
- **poly-decomp** — Stefan Hedman. Used in v2/v3 for shape splitting, dropped in v4 when we wrote a proper classifier.

---

## Final note

Poly-tract is a small project with a small goal: make air drawing feel good. If it does, the rest is details. If it does not, no feature list will save it.

Tune the filter. Watch the cursor. When the cursor feels like an extension of your finger rather than a thing chasing your finger, everything else falls into place.
