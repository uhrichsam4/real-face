# Face Perspective Simulator

**A single HTML file that shows how camera-to-subject distance changes the shape of a face — by re-projecting the face's real 3D depth, not by applying a filter.**

Everything runs in your browser, on your device. No upload, no server, no build step.

---

## The problem it solves

A phone selfie taken at arm's length and a portrait taken from ten feet away are not the same face. At 18 inches the nose is measurably closer to the lens than the ears are, so it projects larger; the cheeks fall away faster; the face looks wider through the middle and narrower at the edges. At 10 feet that difference has almost vanished. This is *perspective*, and it is caused by one thing only: **where the camera is standing**.

It is not caused by focal length. A 24 mm lens and a 135 mm lens produce identical geometry from the same spot — the long lens just crops. What makes a 135 mm portrait look "better" is that you had to walk backwards to fill the frame with it.

This app makes that variable adjustable. It measures the face's depth, then re-projects every pixel as if the camera had physically moved, while keeping the face the same size in frame so you are comparing shape and nothing else. Drag from 1 ft to 10,000 ft and watch what the camera position was doing to you.

**What it is not:** a beauty filter, a face-slimming tool, or a retouching app. Nothing is smoothed, whitened, reshaped by taste, or made "better". The only thing it changes is the projection geometry, and it will happily make you look *worse* if you drag the slider the other way.

---

## Quick start

There is nothing to install and nothing to build.

1. Download `face-perspective-simulator.html`.
2. Open it in a browser.

That's the whole thing — it's one self-contained file. (It fetches the face-tracking library and model from a public CDN the first time it runs, so the first load needs an internet connection. See [Privacy](#privacy).)

### The `file://` camera caveat

Browsers only allow `getUserMedia` (camera access) from a **secure context**. Opening the file straight off disk gives you a `file://` page, and browsers disagree about whether that counts:

| How you open it | Camera | Photo mode |
|---|---|---|
| **Chrome / Edge**, double-clicked from disk (`file://`) | Works — Chromium treats `file://` as a trustworthy origin | Works |
| **Safari**, double-clicked from disk (`file://`) | Blocked | Works |
| **Firefox**, from disk (`file://`) | Blocked in current versions | Works |
| Any browser over `http://localhost` or `https://` | Works | Works |

Photo mode (upload, drag-and-drop, paste) works everywhere, including Safari from disk. If the camera is unavailable the app says so and offers the photo path instead.

### Serving it locally (works in every browser)

From the folder containing the file:

```sh
python3 -m http.server 8000
```

then open <http://localhost:8000/face-perspective-simulator.html>. `localhost` is a secure context, so the camera works in Safari and Firefox too.

### GitHub Pages

Because it is a single static file with no build step, GitHub Pages serves it directly at an `https://` URL — a secure context, so the camera works there for everyone, on desktop and mobile, with no local setup at all:

**→ [uhrichsam4.github.io/real-face](https://uhrichsam4.github.io/real-face/)**

(The repository root holds a small `index.html` that forwards to `face-perspective-simulator.html`, so the clean URL works.)

### Requirements

- **WebGL** (WebGL 2 preferred; it falls back to WebGL 1). Without it the app shows an error and cannot display images.
- An internet connection **on first load**, to fetch the MediaPipe library, its WASM runtime and the landmark model.
- A camera is optional — photos work without one.

---

## The maths

### The magnification law

Model the camera as a pinhole. Put a facial point at metric depth `d` in front of the face's reference plane — `d > 0` means it protrudes *toward* the camera (the nose tip), `d < 0` means it recedes (the sides of the head, the ears).

The source photo was taken from distance `Ds` with focal length `Fs`, so the point landed at image position

```
ps = Fs * X / (Ds - d)
```

A target camera at distance `Dt` with focal length `Ft` would place the same point at

```
pt = Ft * X / (Dt - d)
```

The unknown 3D coordinate `X` cancels, leaving a pure image-space magnification — you never need to know the point's lateral position, only its depth:

```
pt / ps  =  (Ft / Fs) * (Ds - d) / (Dt - d)
```

### Focal-length compensation ("face size lock")

Left alone, moving the camera also changes how big the face is in frame, which swamps the shape change you are trying to see. **Face size lock** removes it by choosing the focal length that a photographer would have chosen — one proportional to distance:

```
Ft / Fs = Dt / Ds
```

Substituting gives the formula the whole app is built on, implemented in `calcMagnification()`:

```
m(d) = Dt (Ds - d) / (Ds (Dt - d))
```

| term | meaning |
|---|---|
| `d` | metric depth of a facial point in front of the reference plane, in metres; positive = toward the camera |
| `Ds` | distance the **source** image was actually shot from, in metres |
| `Dt` | distance of the **simulated** camera, in metres |
| `m(d)` | radial magnification to apply to that point, about the projection centre |

Its behaviour:

- **`m(0) = 1`** — the reference plane never moves. That is what makes it a shape change rather than a zoom.
- **`m < 1` for protruding features when `Dt > Ds`.** Step back and the nose shrinks relative to the rest of the face.
- **`m > 1` for receding features when `Dt > Ds`.** The sides of the head, the ears, the jaw behind the cheekbone all grow.

Together those two are exactly real perspective compression — no term for "slimming", no term for "flattering", just the projection.

Turning face size lock **off** drops the `Ft/Fs = Dt/Ds` substitution and instead scales the whole frame by `Ds/Dt`. The two operations compose to `(Ds - d) / (Dt - d)`, which is the true **fixed-focal-length** result: you walked backwards and did not change the lens, so the face got smaller as well as flatter.

### The orthographic limit

As `Dt → ∞`, the factor `Dt / (Dt - d)` tends to 1 and the law collapses to

```
m(d) → (Ds - d) / Ds  =  1 - d / Ds
```

This is the **orthographic limit**: a camera infinitely far away with an infinitely long lens, where rays are parallel and depth no longer produces any size difference at all. It is a real ceiling, not an asymptote you can pass — which is why the distance slider tops out at 10,000 ft and the app labels the last stop `ORTHOGRAPHIC LIMIT`. Everything beyond that point is visually identical.

### Guard rails in the implementation

The shader and the CPU path apply the same clamps, so nothing blows up at extreme settings:

- `Ds - d` and `Dt - d` are floored at 0.02 m (you cannot put the camera inside the nose).
- `d` is clamped to `[-0.30 m, min(3 × dRange, 0.45 × min(Ds, Dt))]`.
- `m` is clamped to `[0.45, 2.2]`.
- A uniform-scale (Procrustes-style) normalisation factor is solved each frame over the landmark set and multiplied into `m`, so face size lock holds *exactly* regardless of where the depth reference plane happens to land.
- The magnification is applied radially about the **face centre** (the mean of the two cheekbone landmarks, the forehead and the chin), not about the image centre.

### Where `Ds` comes from

`Ds` — the distance the original was shot from — is the one number the app cannot see in the pixels. It is obtained, in order of preference:

1. **EXIF.** For JPEGs the app parses `FocalLengthIn35mmFilm` (tag `0xA405`) with a small built-in reader and inverts the framing equation. The raw `FocalLength` tag is deliberately ignored — it is meaningless without the sensor's crop factor.
2. **Face size + assumed field of view.** `Ds = 0.145 m / (2 · faceWidthFraction · tan(halfFOV))`, with the assumed camera FOV adjustable from 30° to 110° (default 60°) and a portrait/landscape correction, since a camera's field of view is fixed along the sensor's long edge.
3. **You set it.** A slider and one-tap chips (1, 1.5, 2, 3, 4, 5, 6, 8, 10 ft) in the Advanced panel.

Whenever the value was estimated rather than entered, the UI prefixes it with `≈` and tells you which method produced it. The metric scale everywhere comes from one anthropometric constant: an assumed **145 mm bizygomatic (cheekbone) face width**.

### The equivalent-focal-length readout

The `Equivalent framing` figure is derived, not assumed: from the face's width as a fraction of the *visible* frame it recovers the frame's real-world width, then reports `focal = sensor_width_mm × Dt / frame_width_m`. Choose the sensor it is quoted against — full frame (36 mm), APS-C (23.6 mm), phone/generic (7.6 mm), or a custom width from 1 to 120 mm.

There is also a **focal-length control mode**, where the slider runs 14 mm → 300 mm and the app *moves the virtual camera* to whatever distance preserves the framing at that focal length. That is the honest version of "shoot it at 135 mm": the lens number is a proxy for the walk backwards.

---

## The rendering pipeline

Five GPU passes per frame, all in WebGL.

**1 · Depth field (512 × 512 offscreen).** The 478 face landmarks are converted to a metric depth `D[i] = (zRef − z[i]) · k`, where `zRef` is the mean depth of the face and `k` metres-per-unit comes from the assumed cheekbone width. Two expansion rings are generated around the 36-point face oval at 1.22× and 1.58× radius, carrying 62% and 24% of the oval's own depth, so the effect reaches into hair, temples, jaw and neck instead of stopping at a hard face boundary. The resulting 550 vertices are triangulated with a Bowyer–Watson Delaunay pass (topology cached and only recomputed when head pose changes by more than ~7°) and rasterised into an RGBA target: red = encoded depth, green = coverage mask.

**2–3 · Mask-normalised blur.** A separable 13-tap Gaussian runs horizontally then vertically over both channels. In the warp shader the blurred depth is divided by the blurred mask, which *extrapolates* the depth field smoothly outward past the face rather than fading it toward zero — that is what prevents a visible face-shaped seam. The crisp interior field is mixed back in over the face proper via a smoothstep on the unblurred mask, so detail is not lost where it matters. Blur radius scales with face size and is exposed as the "face mesh blend" control.

**4 · Per-pixel inverse warp.** The output is a single full-screen triangle. For each pixel it must answer "which source pixel lands here?", which is the *inverse* of the forward map `F(p) = C + (p − C)·g(p)` — and `g` depends on the depth at `p`, so there is no closed form. The shader solves it with **five damped fixed-point iterations** (damping 0.78, which keeps it stable in the blend annulus where `g` varies fastest). Because it is per-pixel and inverse, there are no triangle seams and no forward-scatter holes anywhere in the output. Compare mode, split position and the hold-for-original state are all uniforms in this same pass, so the comparison is exact and free.

**5 · Overlay (optional).** Wireframe and landmark points, coloured by depth, drawn only when the mesh view is on.

Supporting machinery: landmarks are smoothed with a **One Euro filter** (adaptive — heavy smoothing when still to kill jitter, light when moving to kill lag) and interpolated per render frame, while detection itself runs at 30 Hz independently of the render loop. WebGL context loss is caught and the renderer rebuilds itself.

---

## The personal 3D face scan

A single frame gives only an *estimated* depth per landmark. The scan replaces it with a measured one.

**The insight:** if you turn your head while the app watches, the same landmark is seen from several directions, and a landmark's depth `d` shows up in the accurately-measured image-x coordinate as `sin(yaw) · d`. Fusing keyframes across yaw therefore recovers real, personal facial depth instead of a generic estimate.

**Capture.** Yaw is binned into 13 bins of 7° (±42°), at most two keyframes per bin, sampled no faster than every 110 ms, and frames with more than 28° of pitch are rejected. The on-screen dots fill in as you sweep. It completes when it has the centre bin, at least one bin on each side, at least 28° of total span and at least 8 keyframes.

**Solve.** Classic alternating bundle adjustment with known correspondences, 4 iterations:

1. Align every keyframe to the current model with an **Umeyama/Horn similarity fit** (rotation recovered as the top eigenvector of the 4×4 quaternion matrix, via cyclic Jacobi).
2. Re-solve every vertex by weighted least squares, trusting each frame's x/y far more than its z — `W = diag(1, 1, 0.22)` — with a small ridge term for stability.
3. Repeat, then gauge-fix the model back onto the frontal frame's pose and scale.

**Quality.** Reported as a single 0–100% number combining x/y reprojection agreement across all keyframes (RMS scored against a 0.012 threshold) and angular coverage (scored against 55°), half each. The captured rotation span is reported alongside it.

**What it measurably changes:**

- The depth field driving the warp comes from a **multi-view solve** instead of a single-frame monocular estimate — the app fits the stored model onto each live frame with a similarity transform and substitutes its z entirely.
- Depth becomes **personal**: an actually-prominent nose stays prominent instead of being pulled toward the tracker's average face. The developer panel reports nose depth in millimetres, so the before/after is directly readable, and the **"3D scan depth" toggle flips between scanned and single-frame depth live** so you can see the difference on the same face at the same distance.
- It **carries over to photos**: scan once with the camera, then load a photo of the same person and the measured geometry is used there too.
- Exported `.obj` files record which was used in their header comment (`from personal 3D scan` vs `single-frame depth estimate`).

The model is stored in `localStorage` under the key `fps.facescan.v1` — on your device, never transmitted — and can be cleared from the Advanced panel at any time.

---

## Features

### Sources

- **Live camera**, with a device picker when more than one camera is present (tries 1920×1080, then 1280×720, then whatever it can get).
- **Photos** — file picker, drag-and-drop anywhere on the window, or paste from the clipboard. JPEG / PNG / WebP. Images are downscaled to a 2048 px long edge before processing.
- **Up to 4 faces** detected per photo; when there is more than one, tap the labelled box over any face to simulate that one.
- Automatic source-distance estimation on load, from EXIF or face size, always flagged as an estimate.
- Helpful warnings when the face is cut off at the frame edge or too small for good results.

### The main panel

- **Distance readout** in feet and metres, with a plain-language label: `EXTREME CLOSE-UP` · `PHONE SELFIE DISTANCE` · `CLOSE PORTRAIT` · `SHORT PORTRAIT` · `NATURAL PORTRAIT` · `LONG PORTRAIT` · `COMPRESSED PERSPECTIVE` · `STRONGLY COMPRESSED` · `VERY FLAT PERSPECTIVE` · `NEARLY ORTHOGRAPHIC` · `ORTHOGRAPHIC LIMIT`.
- **Distance slider**, 1 ft → 10,000 ft on a gamma-2.2 curve so roughly 60% of the travel covers the useful 1–20 ft range.
- **Presets**: 1, 2, 3, 5, 10, 20, 50 ft and ∞.
- **Live meta line**: source distance, target distance, equivalent framing in millimetres.

### Comparison modes

- **SIMULATED / ORIGINAL / SPLIT** segmented control.
- **Split view** with a draggable divider, labelled on both sides; in split mode you can drag anywhere on the image, not just the handle.
- **HOLD FOR ORIGINAL** — press and hold the button (or the spacebar) for an instant A/B; release to return.

### Advanced panel

| Control | Range / options | What it does |
|---|---|---|
| Perspective strength | 0–150% | 100% is the physically estimated simulation; below blends toward the original, above exaggerates for demonstration |
| Face size lock | on / off | Keep framing constant (on) or show the true fixed-focal-length result (off) |
| Control | Distance / Focal length | Drive the simulation by camera position or by equivalent lens |
| Source distance | slider + chips + **Estimate** | The `Ds` the original was shot from |
| Equivalent camera | Full frame / APS-C / Phone / Custom | Sensor width the focal readout is quoted against |
| Camera field of view | 30–110° | Assumed FOV used by the face-size distance estimate |
| Depth strength | 0–250% | Scales the measured depth field |
| Face mesh blend | 30–250% | Blur radius — how far the warp reaches into hair, jaw and neck |
| Landmark smoothing | 0–100% | One Euro filter aggressiveness |
| Head stabilization | on / off | Roll-corrects and re-centres on the eyes so only shape changes |
| Mirror view | on / off | Selfie mirroring (auto-off for photos) |
| Show face mesh | on / off | Depth-coloured wireframe overlay |
| 3D scan depth | on / off | Scanned geometry vs single-frame estimate |
| Scan actions | Start / Rescan · Clear · Export .obj | |
| Camera | device list | Shown when more than one camera exists |

### 3D view

Press the cube button (or **V**) to cycle **off → textured → shaded clay**. This renders the reconstructed facial surface itself — the same geometry that drives the warp — so you can inspect what the simulation is actually working from. It auto-spins, you can drag to rotate, and it resumes spinning 2.5 s after you let go. The displayed surface is clipped to the face oval for a clean silhouette and gets two light Laplacian smoothing passes on depth (display only — the warp uses the separately blurred field).

### Export

- **Save frame** — PNG of exactly what is on screen, named with the simulated distance and a timestamp.
- **Export .obj** — the reconstructed facial surface as Wavefront OBJ, in **metres**, Y up, Z toward the viewer, with UVs. Saves three files: `face.obj`, `face.mtl` and `face.jpg` (the source frame as the texture), so it opens textured in Blender, MeshLab or any 3D tool. The header comment records the vertex/triangle count and whether the geometry came from a personal scan or a single-frame estimate.

### Keyboard shortcuts

| Key | Action |
|---|---|
| `Space` (hold) | Show the original |
| `C` | Cycle compare mode: simulated → original → split |
| `M` | Toggle face mesh overlay |
| `V` | Cycle 3D view: off → textured → shaded |
| `S` | Save frame as PNG |
| `R` | Reset |
| `F` | Fullscreen |
| `D` | Toggle the developer panel (also `Cmd`/`Ctrl` + `Shift` + `D`, which works while a control has focus) |

Double-tap or double-click the image to toggle **immersive mode**, which fades the chrome away and leaves just the image and the slider. Entering fullscreen does the same.

### Developer panel

Press `D`. It reports, live: FPS, frame time, tracker rate and detection cost, landmark count, tracking state and face count, mesh triangle count, WebGL version — then source and target distance in feet and metres, the distance ratio, the focal multiplier, face width as a percentage of frame, **nose depth in millimetres**, and head yaw / pitch / roll in degrees. Three sliders at the bottom drive depth scale, warp falloff and smoothing directly.

A hidden self-test harness is exposed on `window.FPS` for verification work: `FPS.demo()` loads a synthetic analytic head (ellipsoid plus a parameterised nose bump) through the entire real pipeline with no camera and no photo, `FPS.probe(srcFt, tgtFt, lock)` returns the radial magnification factors at nose, cheek sides, chin, forehead and eye, and `FPS.step()` / `FPS.pixels()` / `FPS.diff()` render a frame and compare framebuffers numerically. It is not part of the UI.

---

## Privacy

**No image, video frame, landmark, measurement or scan ever leaves your device.** There is no server, no upload endpoint, no analytics, no telemetry, no cookies, no accounts. Camera frames go from the camera into a WebGL texture and are never read back except when *you* press Save.

The app does make exactly three kinds of outbound request, all of them to fetch code and model weights, all of them on first load only, and none of them carrying any of your data:

| Host | What is fetched | When |
|---|---|---|
| `cdn.jsdelivr.net` | `@mediapipe/tasks-vision@0.10.14` — the vision bundle and its WASM runtime | First load (primary) |
| `unpkg.com` | The same `@mediapipe/tasks-vision@0.10.14` bundle | Only if jsDelivr fails |
| `cdn.jsdelivr.net` | `@mediapipe/tasks-vision@0.10.9` | Only if both of the above fail |
| `storage.googleapis.com` | `face_landmarker.task` — the float16 landmark model | First load |

These are plain static asset downloads. Nothing is sent *to* them. Once the browser has cached them the app runs without further network activity, and if they cannot be reached the app says so plainly and still displays your image unmodified.

The only thing persisted anywhere is your 3D face scan, in `localStorage` under `fps.facescan.v1`, on that browser on that device. "Clear scan" deletes it.

If you want a fully offline build, download the three MediaPipe assets, host them alongside the HTML file, and edit the `MP_VERSIONS` and `MODEL_URLS` arrays near the top of the script to point at your local copies.

---

## Limitations

Stated honestly, because a simulation that hides its assumptions is worse than no simulation.

- **It is a 2D re-projection of measured depth, not a 3D re-render.** The warp moves pixels radially according to their depth. It cannot invent geometry that the source image never saw — nothing hidden behind the nose or beyond the silhouette can be revealed. Very large distance changes therefore stretch rather than disclose, and the app clamps magnification to `[0.45, 2.2]` to keep that from becoming absurd.
- **Accuracy is bounded by `Ds`.** If the source distance estimate is wrong, everything downstream is proportionally wrong. EXIF is trustworthy when present; the face-size estimate depends on an assumed field of view and on you having an average-width face. Set it manually when you know it.
- **The metric scale assumes a 145 mm cheekbone width.** Real adult faces vary; if yours differs, absolute distances are biased by the same ratio (relative comparisons at a fixed source distance are unaffected).
- **Single-frame depth is approximate.** Without a scan, depth comes from the tracker's monocular per-landmark z, which is relative and regularised toward an average face. The scan improves this substantially but is still recovered from a single ordinary camera under monocular assumptions — it is not a structured-light or ToF measurement.
- **The blend region is invented, not measured.** The two expansion rings that carry the warp into hair, temples and neck use fractional depths (62% and 24% of the oval's) chosen to look right. They are not reconstructed geometry.
- **Only one face is warped at a time.** In a group photo the other detected faces are left unmodified, and the transition at the frame edges of a strongly warped face can be visible.
- **Only geometry is simulated.** Moving a real camera also changes background blur, depth of field, lighting falloff, background framing and how much of the scene is behind the subject. None of that is modelled. The background stays exactly as shot.
- **No lens distortion model.** Barrel and pincushion distortion, which real wide and long lenses genuinely add, are not simulated or removed.
- **Pose limits.** Yaw/pitch/roll are approximated from landmark geometry rather than solved rigorously; the scan rejects frames beyond 28° of pitch; strongly turned or tilted heads degrade the result. The app warns you when the face is cut off at the frame edge or is small in frame.
- **Requires WebGL and a first-load internet connection.** No WebGL, no app. No network on first run, no face tracking (the image is still shown, unmodified).
- **Camera needs a secure context.** See the [`file://` caveat](#the-file-camera-caveat) above.
- **Not a medical, forensic, or surgical-planning tool.** It is a demonstration of an optical effect. Do not make decisions about your face with it.

---

## Credits

- **[MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)** by Google — provides the 478-point face mesh that everything here is built on. The `@mediapipe/tasks-vision` package is licensed under the **Apache License 2.0**.
- **`face_landmarker.task`** — the float16 face landmark model asset, distributed by Google via `storage.googleapis.com/mediapipe-models/`. See Google's model card for its intended use and its documented limitations.
- The perspective model, depth-field rendering, inverse warp, multi-view scan solver and interface in this file are original work.

## Licence

[MIT](LICENSE) — do what you like with it, keep the notice.

No third-party code is vendored in this repository: MediaPipe is fetched at runtime from a CDN, so the repo contains only URLs. Those components remain under their own licences (Apache-2.0 for `@mediapipe/tasks-vision`), and if you ever vendor them into the repo, keep their `LICENSE`/`NOTICE` files alongside and check Google's separate model terms for `face_landmarker.task`.
