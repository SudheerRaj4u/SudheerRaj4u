# ⚡ EE654 INTERVIEW GUIDE — 3-HOUR SPRINT
## CAR Format: Context → Action → Result
**Interview format:** 10–15 minutes | Two people (Prof. Gerry Lacey + one other)
**Rule:** No generalities. Specific code + hard numbers ONLY.

---

## 🎯 YOUR OPENING STATEMENT (say this if asked "tell me about your project")

> *"My project solves a specific occlusion failure in marker-based AR. When a real person walks in front of a physical AR poster, conventional AR renders the video on top of them — which looks wrong and breaks immersion. I fixed this using ARCore's native human segmentation stencil combined with a custom URP HLSL shader that discards video pixels wherever a person is detected using HLSL's clip() function. The result: the person correctly appears in front of the virtual video. The system achieved 81% occlusion accuracy at 44 FPS on a Samsung Galaxy Tab A7 Lite, with an end-to-end latency of 28 milliseconds."*

---

## 📋 THE 10 MOST LIKELY QUESTIONS — CAR ANSWERS

---

### Q1 — What specific technical problem did you solve?

**CONTEXT:**
> Marker-based AR renders a video texture on a flat 3D plane locked to a physical poster. Unity's rendering pipeline draws that plane on top of everything in the scene — including real people who walk in front of it. This is called an occlusion error. It's a fundamental depth violation: the virtual content appears closer to the camera than real objects that are physically closer.

**ACTION:**
> I implemented a pipeline using `AROcclusionManager` from AR Foundation 6.3.4 to get ARCore's `humanStencilTexture` — a per-pixel mask where white = person, black = background. Every frame in `OcclusionManager.cs`, I call:
> ```csharp
> Texture humanStencil = _arOcclusionManager.humanStencilTexture;
> videoPlaneRenderer.material.SetTexture("_OcclusionMask", humanStencil);
> ```
> My custom `OcclusionShader.shader` then reads this mask in the fragment stage and calls `clip(0.5 - occlude)` to discard video pixels wherever the person is detected.

**RESULT:**
> - **81% occlusion accuracy** for full-body occlusion
> - **68% accuracy** for hand-only occlusion (harder because hands are small and fast-moving)
> - **44 FPS** with the full occlusion pipeline active on a Samsung Galaxy Tab A7 Lite
> - **28ms end-to-end latency** from camera frame to pixel discard

---

### Q2 — Explain exactly how clip() creates the occlusion effect

**CONTEXT:**
> Simply making pixels transparent isn't sufficient for AR occlusion. Alpha blending would still show the video through the person. What's needed is for the video pixels to be completely removed so the AR camera feed — which is already rendered in the background — can show through.

**ACTION:**
> In `OcclusionShader.shader`, the fragment shader does this:
> ```hlsl
> float maskVal = SAMPLE_TEXTURE2D(_OcclusionMask, sampler_OcclusionMask, maskUV).r;
> float edge    = smoothstep(_MaskThreshold - _EdgeSoftness,
>                            _MaskThreshold + _EdgeSoftness, maskVal);
> clip(0.5 - edge); // if edge > 0.5, this value is negative → pixel DISCARDED
> ```
> `clip(x)` in HLSL discards the fragment if `x < 0`. A discarded fragment is never written to the framebuffer. Since the AR camera background is rendered in the `Background` pass (before this shader, which uses `Queue = Geometry+1`), the camera feed is already there underneath — and it shows through the discarded pixels.
> I also use `smoothstep()` to create a soft edge at the silhouette boundary, which removes hard pixel-level aliasing.

**RESULT:**
> The person appears cleanly in front of the video. The `_EdgeSoftness = 0.08` parameter created a smooth 8% transition zone at silhouette boundaries, eliminating the blocky jagged edge that `clip()` alone would produce.

---

### Q3 — Why ARCore native stencil instead of MobileSAM / depth-based methods?

**CONTEXT:**
> I evaluated three approaches: ARCore Depth API (monocular depth estimation), MobileSAM via Unity Sentis (semantic segmentation AI), and ARCore's native human segmentation stencil.

**ACTION:**
> I chose ARCore native stencil because:
> 1. **No depth sensor required** — ARCore Depth API produces noisy estimates at depth discontinuities (object silhouette edges) on devices without dedicated depth hardware. Most phones don't have ToF sensors.
> 2. **Zero extra inference cost** — The ARCore stencil runs on ARCore's internal pipeline, not on my CPU/GPU budget. MobileSAM's two-stage ONNX pipeline adds approximately 2 frames of additional GPU latency per inference.
> 3. **Best boundary quality** — ARCore's human segmentation follows the actual visual boundary of the person, giving sharper occlusion edges than monocular depth at silhouette boundaries.
> I did implement MobileSAM as an alternative backend (`SentisInferenceRunner.cs`) using Unity Sentis 2.4 with `BackendType.GPUCompute`, but the ARCore native approach was the production solution.

**RESULT:**
> ARCore stencil: **44 FPS**, latency **~28ms**. MobileSAM backend: latency increases by ~2 frames per inference cycle due to the two-stage pipeline (Stage 1 = encoder yields 1 frame, Stage 2 = decoder yields 1 frame). ARCore was **faster and required no model download**.

---

### Q4 — Walk me through the complete technical pipeline step by step

**CONTEXT:**
> The problem requires four distinct technical stages from camera input to correct pixel output.

**ACTION (the full pipeline):**
> 1. **`ARTrackedImageManager`** detects the physical poster using the Reference Image Library. When found, `ARTrackedImageHandler.cs` calls `Instantiate()` to spawn the `ARVideoPlane` prefab at the poster's transform position and scales it to physical dimensions using `trackedImage.size.x / 10f`.
> 2. **`VideoPlayer`** on the spawned prefab renders a looping `.mp4` into a `VideoRenderTexture` (1920×1080 RenderTexture).
> 3. **`AROcclusionManager`** runs ARCore's human segmentation with `requestedHumanStencilMode = HumanSegmentationStencilMode.Best`. Each frame, `OcclusionManager.cs` reads `humanStencilTexture` and sends it to the shader via `material.SetTexture("_OcclusionMask", humanStencil)`.
> 4. **`OcclusionShader.shader`** runs in the fragment stage: samples the video texture, samples the mask in screen space, applies `smoothstep()` for edge softening, and calls `clip()` to discard pixels where the person is detected. Render Queue = `Geometry+1` ensures the AR camera feed is already rendered behind.

**RESULT:**
> End-to-end: poster detection within **1–3 seconds** of pointing the camera at it. Occlusion updates every frame. **44 FPS** sustained during full occlusion with the pipeline active.

---

### Q5 — What is BackendType.GPUCompute and why does it matter?

**CONTEXT:**
> I implemented MobileSAM as an alternative backend using Unity Sentis 2.4. The question is how to run an AI model on a mobile device without destroying frame rate.

**ACTION:**
> In `SentisInferenceRunner.cs`, I create the inference workers with:
> ```csharp
> _encoderWorker = new Worker(_encoderRuntimeModel, BackendType.GPUCompute);
> _decoderWorker = new Worker(_decoderRuntimeModel, BackendType.GPUCompute);
> ```
> `BackendType.GPUCompute` executes the ONNX model using Unity compute shaders on the mobile GPU (Adreno/Mali), rather than the CPU. GPUs are massively parallel — ideal for the matrix multiplications that dominate neural network inference. I also run each stage as a coroutine with `yield return null` after each `Schedule()` call, so the GPU runs inference during the next frame without blocking the render thread.
> I included a CPU fallback in a `try/catch` block for devices where GPU compute is unavailable.

**RESULT:**
> GPU inference avoids stalling the render thread. The 2-frame latency overhead (one yield per stage) is acceptable because people move slowly relative to the frame rate — a 1-frame-old mask is visually indistinguishable from a current one at 30+ FPS.

---

### Q6 — How does tracking work and what happens when tracking is lost?

**CONTEXT:**
> If the phone camera looks away from the poster, the video plane should not snap away abruptly. This would look jarring, especially during the occlusion demo.

**ACTION:**
> In `ARTrackedImageHandler.cs`, I implemented two mechanisms:
> 1. **Smooth pose update** using `Vector3.Lerp` and `Quaternion.Slerp` with `Time.deltaTime * 8f`, which creates a physically smooth catch-up motion instead of teleporting the plane each frame.
> 2. **Grace period**: when `trackingState == TrackingState.Limited`, I start a timer in `_lossTimers` dictionary. The video plane is only hidden after `trackingLossGracePeriod = 0.5 seconds` — so brief partial occlusion of the poster (e.g., by a hand) doesn't cause flickering.
> The AR Foundation 6.3.x API required updating from C# events (`+=`) to UnityEvents (`AddListener()`) — a breaking change from earlier AR Foundation versions.

**RESULT:**
> No visible flickering when the poster is partially covered. The 0.5s grace period covers the typical duration of a hand passing across the poster while handing over an object.

---

### Q7 — What are the limitations of your system?

**CONTEXT (be honest — Gerry will probe this):**
> No system is perfect. Acknowledging limitations shows genuine understanding.

**ACTION (what you measured and observed):**
> Three specific limitations:
> 1. **Hand occlusion accuracy drops to 68%** vs 81% for full body. Hands are small in the camera frame and move fast, causing the ARCore stencil to lag at the fingertip boundaries.
> 2. **Mask resolution**: ARCore's `humanStencilTexture` is lower resolution than the camera feed. The `smoothstep()` in the shader (`_EdgeSoftness = 0.08`) softens this but can't eliminate it at large distances from the camera.
> 3. **Tracking dependency**: the entire system requires ARCore to successfully track the poster. If the poster is heavily occluded or very poorly lit, tracking fails and the video disappears entirely — there's no graceful degradation at that point.

**RESULT:**
> The 81% → 68% drop between body and hand was measured over 100 frames of test footage on the Samsung Galaxy Tab A7 Lite. Hand tracking is the key area for future improvement — optical flow-based mask propagation between frames would address both the lag and the resolution issue.

---

### Q8 — What is TryAcquireLatestCpuImage() and why is it used? (MobileSAM path)

**CONTEXT:**
> For the MobileSAM alternative backend, I needed to feed raw camera frames to the AI model. The naive approach (taking screenshots) is unacceptably slow.

**ACTION:**
> I use `ARCameraManager.TryAcquireLatestCpuImage()` which returns an `XRCpuImage` — a struct giving direct CPU access to the YCbCr camera data without any GPU-to-CPU round-trip. Crucially, `XRCpuImage` is a native struct that MUST be disposed after use — I use a `using()` block to guarantee disposal and prevent memory leaks. I then call `TextureConverter.ToTensor()` to write the image directly into the pre-allocated `Tensor<float>` without creating any new heap objects.

**RESULT:**
> Avoids the GPU→CPU texture readback that `Texture2D.ReadPixels()` requires. Pre-allocating the tensor and reusing it every frame eliminates garbage collection pauses that would otherwise cause frame time spikes on mobile.

---

### Q9 — How did you evaluate the system?

**CONTEXT:**
> The professor wants specific test design, not just "I tested it and it worked".

**ACTION:**
> I measured three metrics:
> 1. **Occlusion Accuracy**: percentage of person pixels within the AR plane frustum correctly masked — tested over a 100-frame sequence on the Tab A7 Lite.
> 2. **FPS**: mean frame rate measured via `DemoUIController.cs`'s rolling 30-frame FPS counter, comparing baseline (no occlusion) vs occlusion active.
> 3. **Latency**: time from `AROcclusionManager.humanStencilTexture` update to `material.SetTexture()` call — measured via `Time.realtimeSinceStartup` timestamps.
> I tested in two conditions: full-body occlusion (person standing in front) and hand-only occlusion (hand reaching toward the poster).

**RESULT:**

| Metric | Value |
|--------|-------|
| Occlusion Accuracy (full body) | **81%** |
| Occlusion Accuracy (hand only) | **68%** |
| FPS — baseline (no occlusion) | **58 FPS** |
| FPS — with ARCore stencil | **44 FPS** |
| FPS drop from occlusion overhead | **-14 FPS (24% overhead)** |
| End-to-end pipeline latency | **~28 ms** |
| Test device | Samsung Galaxy Tab A7 Lite |

---

### Q10 — What would you do differently or add next?

**CONTEXT:**
> Future work shows you understand the field beyond what you built. Gerry will like specific, technically grounded directions.

**ACTION (two specific directions):**
> 1. **Optical flow mask propagation** — run the ARCore segmentation every 3rd frame, and use optical flow (Lucas-Kanade or RAFT Tiny) to warp the previous mask into the next 2 frames. This would cut stencil overhead by ~66% and address the hand-tracking lag at silhouette boundaries.
> 2. **Depth-semantic fusion on depth-capable devices** — on phones with ToF sensors (e.g., Samsung S24+), fuse the ARCore Depth API map with the semantic stencil: use depth to sharpen boundaries where the stencil is uncertain, and use the stencil to classify which depth discontinuities are foreground objects.

**RESULT:**
> Optical flow at 3-frame skip would push FPS from 44 back toward the 58 FPS baseline — recovering ~10 FPS. Depth-semantic fusion would improve hand accuracy from 68% to an estimated 85%+ on supported hardware.

---

## ⚡ RAPID-FIRE CHEAT SHEET (memorise these)

| What Gerry asks | Your number |
|----------------|-------------|
| FPS with occlusion | **44 FPS** |
| FPS without occlusion | **58 FPS** |
| Occlusion accuracy (body) | **81%** |
| Occlusion accuracy (hand) | **68%** |
| End-to-end latency | **28 ms** |
| Test device | **Samsung Galaxy Tab A7 Lite** |
| AR Foundation version | **6.3.4** |
| Unity version | **Unity 6 (6000.3.7f1)** |
| Mask threshold in shader | **0.05** |
| Edge softness in shader | **0.08** |
| Tracking grace period | **0.5 seconds** |
| Lerp speed for pose | **Time.deltaTime × 8** |
| MobileSAM encoder input | **1024 × 1024** |
| MobileSAM mask output | **256 × 256** |
| MobileSAM encoder size | **~28 MB** |
| MobileSAM decoder size | **~16.5 MB** |

---

## 🧠 KEY CODE LINES TO KNOW BY HEART

```csharp
// OcclusionManager.cs — the core pipeline every frame
_arOcclusionManager.requestedHumanStencilMode = HumanSegmentationStencilMode.Best;
Texture humanStencil = _arOcclusionManager.humanStencilTexture;
videoPlaneRenderer.material.SetTexture("_OcclusionMask", humanStencil);
```

```hlsl
// OcclusionShader.shader — the core discard logic
float maskVal = SAMPLE_TEXTURE2D(_OcclusionMask, sampler_OcclusionMask, maskUV).r;
float edge    = smoothstep(_MaskThreshold - _EdgeSoftness,
                           _MaskThreshold + _EdgeSoftness, maskVal);
clip(0.5 - edge); // discard if edge > 0.5 → person detected → camera feed shows through
```

```csharp
// SentisInferenceRunner.cs — GPU backend selection
_encoderWorker = new Worker(_encoderRuntimeModel, BackendType.GPUCompute);
_decoderWorker = new Worker(_decoderRuntimeModel, BackendType.GPUCompute);
```

```csharp
// ARTrackedImageHandler.cs — smooth pose tracking
obj.transform.SetPositionAndRotation(
    Vector3.Lerp(obj.transform.position, trackedImage.transform.position, Time.deltaTime * 8f),
    Quaternion.Slerp(obj.transform.rotation, trackedImage.transform.rotation, Time.deltaTime * 8f));
```

---

## ⏱️ 3-HOUR STUDY PLAN (it's 11:20am now)

| Time | Task |
|------|------|
| **11:20 – 11:40** | Read this entire guide once through |
| **11:40 – 12:20** | Say Q1–Q5 out loud from memory, check your answer, repeat any you got wrong |
| **12:20 – 13:00** | Say Q6–Q10 out loud from memory, check your answer, repeat any you got wrong |
| **13:00 – 13:20** | Memorise the cheat sheet numbers (cover the right column, say each number) |
| **13:20 – 13:40** | Say the 4 key code snippets from memory + explain what each line does |
| **13:40 – 14:00** | One full practice run: pretend Gerry asks — give your opening statement then answer Q1 Q3 Q5 Q7 Q9 back to back |
| **Interview** | Remember: be specific, use numbers, don't bluff |

---

## 🚨 THINGS GERRY WILL PROBE — BE READY

1. **"What does clip() actually do?"** → it discards the fragment from the render pipeline. Not transparency — complete removal.
2. **"Why Geometry+1 in the render queue?"** → the AR camera background must be rendered FIRST so it's already there underneath when clip() removes the video pixel.
3. **"What is the stencil texture exactly?"** → a per-pixel grayscale image from ARCore where white = human detected, black = background.
4. **"Why not just use alpha blending?"** → alpha blending would still show some video through the person. clip() makes the pixel fully transparent to the layer below (the camera feed).
5. **"Your accuracy is only 81% — what causes the other 19%?"** → mask resolution mismatch between the ARCore stencil and the video texture, plus stencil lag on fast-moving edges.
6. **"What's the difference between your two backends?"** → ARCore native = free, fast, 44 FPS, no AI model needed. MobileSAM = more flexible, works without ARCore human segmentation support, but adds ~2 frame latency per inference.
7. **"What's smoothstep?"** → a HLSL built-in that interpolates smoothly between 0 and 1 across a threshold range, giving a soft edge transition instead of a hard binary cutoff.
8. **"Why yield return null?"** → suspends the coroutine for one frame, allowing the GPU to execute the scheduled AI inference in parallel while Unity renders the current frame normally.

---

## 💬 CLOSING STATEMENT (if asked "anything to add?")

> *"The key insight of this project is that correct AR occlusion requires the virtual layer to be rendered after the real-world layer — and then holes punched through it using clip() wherever real foreground objects are detected. clip() is the only correct mechanism because it truly removes the fragment from the pipeline, exposing the already-rendered camera feed underneath. Alpha blending cannot achieve this. The 44 FPS result shows this approach is viable for consumer AR on mid-range hardware today."*

---

*Good luck — you know this project. Be specific, cite your numbers, and don't generalise. 🎯*
