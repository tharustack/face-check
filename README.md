# face-check

> Browser-only, prompt-driven face check. Turn left, right, look straight,
> blink — verified on-device with [MediaPipe Face Landmarker][mp]. Zero
> uploads. Demo-quality liveness UX, **not** an anti-spoofing system.

**[▶ Try it live](https://Ytharustack.github.io/face-check/)**

---

## ⚠️ This is not an anti-spoofing system

This project is a **demo / UX experiment**, not a security control. It cannot
distinguish a live person from:

- a photo held up to the camera,
- a video replay on another screen,
- a mask, cutout, or printout,
- a deepfake or face-swap.

Please don't deploy it as an authentication gate. If you need real liveness
detection, look at purpose-built SDKs (e.g. FaceTec, iProov, Incode) and treat
this repo as a reference for the *interaction design* only.

---

## Features

- **Runs entirely in the browser.** No backend, no uploads, no telemetry.
- **Personal calibration.** Baseline yaw and eye-aspect-ratio (EAR) are
  sampled from *you* for ~1.5 s, so thresholds adapt instead of using fixed
  constants.
- **Randomised prompts.** Four challenges per run, no two consecutive the same,
  guaranteed to include at least one turn and a blink.
- **Face-mesh overlay** drawn live on the video for visual feedback.
- **Debug mode** (`?debug`) showing yaw delta, EAR, and blink score in real time.
- **Single HTML file.** Everything — markup, styles, module script — in
  `index.html`. Read it in one sitting.

---

## How it works

1. **Camera + model load.** `getUserMedia` grabs the front camera. The
   MediaPipe `FaceLandmarker` task loads from a CDN (WASM) with model weights
   from Google's storage bucket. GPU delegate is tried first, CPU is the
   fallback.
2. **Calibration (~1.5 s).** You're asked to look straight. The script samples
   your neutral yaw and EAR, and sanity-checks face size (too close / too far)
   and head angle. The EAR baseline uses the **75th percentile** of samples so
   a stray blink during calibration doesn't drag it down.
3. **Challenge sequence.** For each prompt:
   - **left / right** — yaw must deviate from the calibrated baseline by more
     than `turnDelta`, and hold for `hold` ms.
   - **straight** — `|yaw − baseline| < straightDelta` and eyes open, held.
   - **blink** — EAR must drop below `baseline × blinkRatio` (or the MediaPipe
     `eyeBlink*` blendshape score exceeds `0.6`) and then *reopen*.
4. **Result.** Pass or fail, with a per-step log and the reason for any
   failure (multiple faces, face left the frame, timed out, etc.).

### Why measure in pixels?

MediaPipe returns normalized landmark coordinates. `measure()` multiplies x by
video width and y by video height before computing distances, so ratios like
EAR and yaw are correct even on non-square aspect ratios (e.g. 960×720).

### Why mirror the video?

`video` and `canvas` both use `transform: scaleX(-1)` so the on-screen view
feels like a mirror. **Yaw is computed in the raw (unmirrored) camera space**,
which is why a physical turn to *your* left *increases* the yaw ratio. The
code comments call this out.

---

## Running locally

The page uses ES modules and requests the camera, so it must be served over
**HTTPS or `localhost`** — `file://` will not work.

Any static server will do:

```bash
# Python 3
python -m http.server 8000

# Node
npx serve .
```

Then open <http://localhost:8000>.

### Debug mode

Append `?debug` to the URL to show a live readout of yaw delta, EAR, and blink
score in the corner of the video. Useful when tuning `CFG`.

---

## Tuning

All thresholds live in one object at the top of the script:

```js
const CFG = {
  steps: 4,            // number of random challenges
  stepTimeout: 7000,   // ms allowed per challenge
  hold: 350,           // ms a pose must be held (turns / straight)
  turnDelta: 0.12,     // yaw change from baseline that counts as a turn
  straightDelta: 0.05, // yaw tolerance for "look straight"
  blinkRatio: 0.70,    // eye closed when EAR < baseline * this
  calibMs: 1500,       // neutral-face calibration time
  lostMs: 1200,        // face missing this long = fail
};
```

Common adjustments:

| Symptom | Try |
| --- | --- |
| Too easy to pass a turn | Increase `turnDelta` (e.g. `0.15`) |
| Turns failing for people with limited neck mobility | Decrease `turnDelta` |
| Blinks not registering | Raise `blinkRatio` toward `0.85`, or lower the blendshape threshold from `0.6` |
| Blinks firing on squints | Lower `blinkRatio` toward `0.55` |
| "Look straight" flapping on/off | Increase `straightDelta` slightly |
| More / fewer challenges | Change `steps` (values ≥ 3 recommended) |

---

## Browser support

Tested on current Chrome, Edge, Safari, and Firefox desktop. Mobile Safari and
Chrome for Android work but expect the GPU delegate fallback to CPU on older
devices. Requires:

- `getUserMedia`
- WebAssembly (with SIMD for the GPU path)
- ES modules

---

## Deployment (GitHub Pages)

1. Push this repo to GitHub.
2. **Settings → Pages → Source:** deploy from branch `main`, folder `/ (root)`.
3. Wait for the Pages URL, then update the "Try it live" link at the top of
   this README.
4. HTTPS is provided by Pages, so the camera prompt will work.

The entry file must be named `index.html`.

---

## Project layout

```
.
├── index.html    # everything: markup, styles, and the module script
├── LICENSE
└── README.md
```

Intentionally single-file — the point is that you can read the whole thing in
one sitting.

---

## Credits & license

- Face detection: [MediaPipe Tasks Vision][mp] (Apache-2.0).
- Model weights are fetched at runtime from `storage.googleapis.com`; nothing
  is bundled in this repo.
- Everything else: **MIT** — see [LICENSE](./LICENSE).

[mp]: https://developers.google.com/mediapipe/solutions/vision/face_landmarker
