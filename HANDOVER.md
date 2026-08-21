# Developer Handover — Face Mesh Visualiser

A short brief for a developer taking this prototype forward. It documents what the
project is, how it's built, how to run and deploy it, and the non-obvious decisions
(especially the GPU-init handling) that are easy to "simplify" away and shouldn't be.

---

## 1. What this is

Two self-contained browser prototypes that run a live **face landmark mesh** over the
front camera using [MediaPipe Face Landmarker](https://developers.google.com/mediapipe/solutions/vision/face_landmarker).
All processing is client-side; no video leaves the device.

| File | What it is | Live URL |
|------|-----------|----------|
| `index.html` | Full-screen face wireframe with a travelling "cuttlefish" colour flash | https://imtiazbahar.github.io/face-mesh-visualiser/index.html |
| `check-selector.html` | A product card (Helfie "Blood Pressure" check selector, built to a Figma spec) with a **Safe Grid** oval that gates a face-scan animation | https://imtiazbahar.github.io/face-mesh-visualiser/check-selector.html |

`check-selector.html` is the actively maintained one. `index.html` is the earlier
standalone visualiser and does **not** yet have the engine-selection hardening described
below.

## 2. Stack & constraints

- **Vanilla HTML + ES modules. No framework, no build step, no package.json.** Each file
  is a single self-contained page.
- MediaPipe is loaded from a CDN at runtime:
  - Library: `@mediapipe/tasks-vision@0.10.12` (jsDelivr)
  - WASM: same package `/wasm`
  - Model: `face_landmarker.task` (float16) from `storage.googleapis.com`
- **Camera requires a secure origin** (`https://` or `localhost`). It will **not** work
  from `file://` (double-clicking the HTML) — this is a browser security rule, not a bug.

## 3. Repo layout

```
index.html            # full-screen mesh prototype
check-selector.html   # check-selector card prototype (main)
assets/               # SVGs used by the card UI (safe-grid, mask, icons, etc.)
README.md             # short run instructions
HANDOVER.md           # this file
```

## 4. Run locally

Any static server over localhost works (localhost counts as a secure origin, so the
camera works):

```bash
cd face-mesh-visualiser
python3 -m http.server 8777
```

Then open `http://localhost:8777/check-selector.html`.

## 5. Deploy

**GitHub Pages, served from the `main` branch root.** Deploy = commit + push to `main`;
Pages rebuilds in roughly a minute. There is no CI and no build artifact.

```bash
git add -A && git commit -m "…" && git push origin main
```

## 6. Architecture of `check-selector.html`

One ES module (`<script type="module">`, ~line 244 to end). Key pieces:

- **Engine selection** (URL params + `USE_GPU_FIRST`, ~line 268) — see §7, read before touching.
- `createLandmarker(delegate)` — builds a `FaceLandmarker` for `"GPU"` or `"CPU"`.
- `ensureModel()` — memoised (single `modelPromise`) model init with the delegate strategy.
- `startCamera()` / `stopCamera()` — `getUserMedia` (`facingMode:"user"`), starts the loop.
- `loop()` → `tick(res)` — per-frame detect + draw; **wrapped so one bad frame can't kill
  the loop** (it logs and continues).
- `faceMetrics()` — computes face height + centre offset from landmarks.
- Alignment gate — face is "aligned" when it's within the size/offset tolerances (§8).
- `triangulate()` / `drawMesh()` — the low-poly mesh + colour "wash" render.
- `renderStat()` — the `?debug=1` status readout (top-left).

Card render size is fixed at `CARD_W=342 × CARD_H=570` (px), scaled to fit the viewport
by `fitStage()`. Canvas is drawn at devicePixelRatio (capped at 2).

## 7. ⚠️ The GPU-init handling — read before refactoring

**Symptom:** on some laptops/desktops, initialising MediaPipe with the **GPU delegate
hangs and freezes the page** — it doesn't throw, it blocks. Face detection never starts.
Phones are unaffected.

**Why the obvious fix doesn't work:** a `Promise.race([init, timeout])` fallback *cannot*
rescue this, because the hang blocks the main thread, so the `setTimeout` powering the
timeout never fires. (That timeout still exists as a secondary net for ordinary async
failures, but it is not what saves the desktop case.)

**Current strategy — avoid the GPU where it hangs, don't try to recover from it:**

```js
const IS_IPADOS  = navigator.platform === "MacIntel" && navigator.maxTouchPoints > 1;
const IS_MOBILE  = /Android|iPhone|iPad|iPod|Mobile/i.test(navigator.userAgent) || IS_IPADOS;
const FORCE_CPU  = params.has("cpu");
const FORCE_GPU  = params.has("gpu");
const USE_GPU_FIRST = !FORCE_CPU && (FORCE_GPU || IS_MOBILE);   // GPU on mobile, CPU on desktop
```

- Mobile (incl. iPadOS, which reports a desktop "Macintosh" UA) → GPU path (fast).
- Desktop → CPU path directly (reliable; never attempts the hanging GPU).
- If a proper feature test for the hang becomes available, that's the ideal replacement —
  until then, do **not** remove the device-based default in favour of race/fallback alone.

This logic and its comments were reviewed via an external second-opinion (OpenAI Codex)
pass; the iPadOS detection and the "CPU wins if both flags set" tie-break came out of it.

## 8. Configuration knobs

**URL params** (combine freely, e.g. `?cpu=1&debug=1`):

| Param | Effect |
|-------|--------|
| `?debug=1` | Show top-left status readout: `model:`, `video:`, `frames:`, `faces:`, alignment metrics, `ERR:` |
| `?gpu=1` | Force GPU delegate (override device default) |
| `?cpu=1` | Force CPU delegate. If both `?gpu=1` and `?cpu=1` are present, **CPU wins** |

**Constants** (top of the module):

| Const | Meaning |
|-------|---------|
| `CARD_W`, `CARD_H` | Card render dimensions (342 × 570) |
| `MIN_H`, `MAX_H` (80, 340) | Acceptable face height in card px → controls near/far distance gating |
| `TOL_X`, `TOL_Y` (78, 100) | Off-centre allowance for "aligned" |
| `WASH_MS` (4200) | Duration of the colour-wash sweep |

Alignment gate: `fh ∈ [MIN_H, MAX_H] && |dx| ≤ TOL_X && |dy| ≤ TOL_Y`. Loosening `MIN_H`
helps far-from-webcam users register as aligned; use `?debug=1` to read live values while tuning.

## 9. Known limitations / gotchas

- **Hung GPU init is not cancelled** — on desktop we simply never start it. A forced
  `?gpu=1` on a bad machine can still freeze the tab. That's the intended escape hatch, not
  the default.
- **CDN dependency at runtime** — offline or if jsDelivr/Google storage is blocked, the
  model won't load. Consider vendoring the library + model for a production build.
- **Element-id/global collision lesson**: a debug element was once given `id="dbg"`, which
  auto-created `window.dbg` and clobbered a name MediaPipe uses internally, breaking
  detection on every device. Avoid short generic element ids on these pages.
- `numFaces: 1` — single face only by design.
- `index.html` has **not** received the §7 engine hardening.

## 10. Suggested next steps

1. Apply the §7 engine-selection strategy to `index.html` too.
2. Vendor MediaPipe (library + `.task` model) into the repo to remove the runtime CDN
   dependency and make it offline-capable.
3. Add a real GPU capability probe (small offscreen WebGL handshake with a hard budget) to
   replace the device-type heuristic if/when reliable.
4. Extract the shared engine-init + mesh code into a small module imported by both pages.
5. Optional: a visible "using CPU (slower)" hint on desktop so testers understand the perf.

---

*Prototype owner: Imtiaz Bahar (design). Repo: `Imtiazbahar/face-mesh-visualiser`.*
