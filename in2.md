# ⚡ EE654 INTERVIEW GUIDE — CAR FORMAT
**Interview:** 10–15 min | Prof. Gerry Lacey + one other | Rule: specific code + hard numbers only.

---

## 🎯 OPENING STATEMENT

> *"My project fixes the AR occlusion error — when a real person walks in front of an AR video poster, conventional AR incorrectly renders the video on top of them, so I used ARCore's native human segmentation stencil and a custom HLSL shader that calls clip() to discard video pixels wherever a person is detected, achieving 81% occlusion accuracy at 44 FPS with 28ms latency on a Samsung Galaxy Tab A7 Lite."*

---

## Q1 — What specific technical problem did you solve?

**C:** Unity's rendering pipeline draws the AR video plane on top of everything in the scene, including real people who physically stand in front of the poster — a depth violation called an occlusion error.

**A:** I used `AROcclusionManager` from AR Foundation 6.3.4 to read ARCore's `humanStencilTexture` every frame in `OcclusionManager.cs` and send it to `OcclusionShader.shader`, which calls `clip(0.5 - occlude)` to discard video pixels wherever a person is detected.

**R:** 81% occlusion accuracy for full-body, 68% for hand-only, at 44 FPS with 28ms end-to-end latency on a Samsung Galaxy Tab A7 Lite.

---

## Q2 — Explain exactly how clip() creates the occlusion effect

**C:** Alpha blending can't achieve true AR occlusion because it still composites some video over the person — you need the pixel completely removed so the camera feed underneath shows through.

**A:** In `OcclusionShader.shader`, the fragment stage reads the mask value, applies `smoothstep()` for soft edges, then calls `clip(0.5 - edge)` — since `clip(x)` discards the fragment when `x < 0`, any pixel where `edge > 0.5` (person detected) is thrown away entirely, revealing the AR camera background already rendered in the `Background` pass behind it.

**R:** The `_EdgeSoftness = 0.08` parameter creates a smooth 8% transition zone at silhouette boundaries, eliminating the hard pixel-level aliasing that clip() alone would produce.

---

## Q3 — Why ARCore native stencil instead of MobileSAM or depth-based occlusion?

**C:** I evaluated three approaches — ARCore Depth API, MobileSAM via Unity Sentis, and ARCore's native human segmentation stencil — and needed the best balance of accuracy, latency, and hardware compatibility.

**A:** I chose ARCore native stencil because it runs on ARCore's internal pipeline at zero extra GPU cost, requires no depth sensor, and produces sharper silhouette boundaries than monocular depth estimation — I still implemented MobileSAM as an alternative backend in `SentisInferenceRunner.cs` using `BackendType.GPUCompute`, but used ARCore as the production solution.

**R:** ARCore stencil achieved 44 FPS and 28ms latency; MobileSAM adds approximately 2 frames of extra GPU latency per inference due to its two-stage encoder-decoder pipeline.

---

## Q4 — Walk me through the complete pipeline step by step

**C:** The system needs four distinct stages to go from raw camera input to correctly occluded pixel output.

**A:** `ARTrackedImageManager` detects the poster and `ARTrackedImageHandler.cs` spawns the `ARVideoPlane` prefab scaled to `trackedImage.size.x / 10f` metres; `VideoPlayer` renders a looping MP4 into a 1920×1080 `VideoRenderTexture`; `OcclusionManager.cs` reads `humanStencilTexture` each frame and calls `material.SetTexture("_OcclusionMask", humanStencil)`; `OcclusionShader.shader` discards person pixels via `clip()` at `Queue = Geometry+1`, revealing the camera feed underneath.

**R:** Poster detection takes 1–3 seconds, occlusion updates every frame, sustained at 44 FPS.

---

## Q5 — What is BackendType.GPUCompute and why does it matter?

**C:** Running a 28MB + 16.5MB AI model on a mobile device every frame risks destroying frame rate if done on the CPU.

**A:** In `SentisInferenceRunner.cs`, I create both inference workers with `new Worker(model, BackendType.GPUCompute)`, which executes the ONNX model on the mobile GPU via Unity compute shaders, and I use `yield return null` after each `Schedule()` call so the GPU runs inference during the next render frame without blocking the main thread — with a CPU fallback in a `try/catch` block.

**R:** GPU inference keeps the render thread unblocked; the 2-frame coroutine overhead is imperceptible because people move slowly relative to a 44 FPS frame rate.

---

## Q6 — How does tracking work and what happens when tracking is lost?

**C:** If the phone looks away from the poster, snapping the video off instantly would look jarring and break the demo.

**A:** In `ARTrackedImageHandler.cs`, I update the plane pose every frame using `Vector3.Lerp` and `Quaternion.Slerp` at `Time.deltaTime * 8f` for smooth catch-up, and when `trackingState == TrackingState.Limited` I start a timer in `_lossTimers` — only hiding the video plane via `SetActive(false)` after `trackingLossGracePeriod = 0.5 seconds`.

**R:** The 0.5s grace period eliminates flickering when a hand briefly covers part of the poster, which is the exact scenario that occurs during the occlusion demo.

---

## Q7 — What are the limitations of your system?

**C:** No system is perfect — understanding limitations is what separates a real implementation from a superficial one.

**A:** The three main limitations are: hand occlusion accuracy drops to 68% vs 81% for full body because hands occupy fewer pixels and move faster than the stencil update rate; the ARCore stencil resolution is lower than the camera feed, softened but not eliminated by `_EdgeSoftness = 0.08` in the shader; and if the poster itself is occluded or poorly lit, ARCore tracking fails entirely with no graceful degradation.

**R:** The 81% → 68% accuracy drop between body and hand was measured over 100 test frames on the Samsung Galaxy Tab A7 Lite.

---

## Q8 — What is TryAcquireLatestCpuImage() and why use it? (MobileSAM path)

**C:** Feeding live camera frames to an AI model requires getting the raw image data fast — a naive screenshot approach requires a slow GPU-to-CPU round-trip that would stall the render thread.

**A:** `ARCameraManager.TryAcquireLatestCpuImage()` returns an `XRCpuImage` struct giving direct CPU access to the YCbCr camera data with no GPU round-trip, and I dispose it immediately using a `using()` block to prevent memory leaks, then write it into a pre-allocated `Tensor<float>` via `TextureConverter.ToTensor()` with no new heap allocation.

**R:** Pre-allocating the tensor and reusing it every frame eliminates garbage collection pauses that would otherwise cause visible frame-time spikes on mobile.

---

## Q9 — How did you evaluate the system?

**C:** I needed three distinct metrics to fully characterise system performance: visual accuracy, rendering cost, and pipeline latency.

**A:** FPS was read from the rolling 30-frame counter in `DemoUIController.cs` with the toggle button used to compare occlusion ON vs OFF; latency was measured using `Time.realtimeSinceStartup` timestamps around the `Update()` pipeline over 500 frames; occlusion accuracy was calculated over 100 test frames by comparing the ARCore stencil against a manually reviewed reference ground truth, counting correctly discarded person pixels within the AR plane area.

**R:** Baseline 58 FPS → 44 FPS with occlusion (24% overhead); 28ms latency; 81% accuracy (full body), 68% (hands).

---

## Q10 — What would you improve next?

**C:** The two biggest remaining weaknesses are GPU overhead from the stencil pipeline and the drop in accuracy for fast-moving hands.

**A:** I would implement optical flow mask propagation — run the ARCore segmentation every 3rd frame and use Lucas-Kanade or RAFT Tiny to warp the previous mask into the next 2 frames — and on depth-sensor devices I would fuse the ARCore Depth API map with the semantic stencil, using depth to sharpen boundaries where the stencil is uncertain.

**R:** Optical flow at 3-frame skip would recover approximately 10 FPS (from 44 back toward the 58 FPS baseline), and depth-semantic fusion would push hand accuracy from 68% toward an estimated 85%+ on ToF-capable devices.

---

## ⚡ NUMBERS TO KNOW COLD

| Metric | Value |
|--------|-------|
| FPS — occlusion ON | **44 FPS** |
| FPS — occlusion OFF | **58 FPS** |
| FPS overhead | **-14 FPS (24%)** |
| Latency | **28 ms** |
| 1 frame at 44 FPS | **22.7 ms** |
| Accuracy — full body | **81%** |
| Accuracy — hand only | **68%** |
| Test device | **Samsung Galaxy Tab A7 Lite** |
| AR Foundation version | **6.3.4** |
| Unity version | **6000.3.7f1** |
| Mask threshold | **0.05** |
| Edge softness | **0.08** |
| Grace period | **0.5 seconds** |
| Lerp speed | **Time.deltaTime × 8** |
| MobileSAM encoder input | **1024 × 1024** |
| MobileSAM mask output | **256 × 256** |
| MobileSAM encoder size | **~28 MB** |
| MobileSAM decoder size | **~16.5 MB** |
| Poster width in library | **0.21 m (A4)** |
| Max tracking distance | **~1.5 m (ARCore limit for A4)** |

---

## 🧠 KEY CODE LINES

```csharp
// OcclusionManager.cs — the core every frame
Texture humanStencil = _arOcclusionManager.humanStencilTexture;
videoPlaneRenderer.material.SetTexture("_OcclusionMask", humanStencil);
```
```hlsl
// OcclusionShader.shader — the core discard
float edge = smoothstep(_MaskThreshold - _EdgeSoftness, _MaskThreshold + _EdgeSoftness, maskVal);
clip(0.5 - edge);
```
```csharp
// SentisInferenceRunner.cs — GPU backend
_encoderWorker = new Worker(_encoderRuntimeModel, BackendType.GPUCompute);
```
```csharp
// ARTrackedImageHandler.cs — smooth tracking
Vector3.Lerp(obj.transform.position, trackedImage.transform.position, Time.deltaTime * 8f)
```

---

## 🚨 GERRY'S LIKELY PROBES — ONE-LINE ANSWERS

**"What does clip() actually do?"** → It discards the fragment from the render pipeline entirely — not transparency, complete removal.

**"Why Geometry+1 in the render queue?"** → The AR camera background must render first so it's already behind the video plane when clip() punches holes through it.

**"Why not just alpha blending?"** → Alpha blending still composites some video over the person; clip() removes the pixel completely so the camera feed behind shows through.

**"Your accuracy is only 81% — why not higher?"** → Mask resolution mismatch between the ARCore stencil and the camera feed, plus stencil lag on fast-moving silhouette edges.

**"What's smoothstep?"** → A HLSL built-in that interpolates smoothly between 0 and 1 across a threshold range, giving a soft-edge transition instead of a hard binary cutoff.

**"What's yield return null?"** → It suspends the coroutine for one frame, letting the GPU execute the scheduled AI inference in parallel while Unity renders normally.

**"What's the difference between your two backends?"** → ARCore native: free, 44 FPS, no model needed; MobileSAM Sentis: hardware-agnostic but adds ~2 frames of GPU latency per inference.

**"Max distance the video plays?"** → No hard limit in code — ARCore tracking for a 0.21m A4 poster works reliably to about 1.5m, then drops to Limited state and hides after the 0.5s grace period.

---

## 💬 CLOSING STATEMENT

> *"The key insight is that correct AR occlusion requires the virtual layer to render after the camera feed, then punch holes through it using clip() wherever real foreground objects are detected — alpha blending cannot achieve this — and the 44 FPS result proves this approach is viable on consumer mid-range hardware today."*
