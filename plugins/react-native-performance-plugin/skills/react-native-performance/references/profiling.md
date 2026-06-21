# Profiling: Finding Where the 16ms Goes

Profiling is how you replace guessing with evidence. The goal is to answer one question
for the janky interaction: **within each ~16ms frame, which thread runs past the frame
boundary?** That tells you whether the problem is JS, GPU/draw, or native view creation.

## Before anything: turn dev mode OFF

Profile a **release build** on a **real device** (ideally mid-range, not a flagship).
Dev mode adds heavy per-frame validation that distorts every number. This is the most
common profiling mistake.

## Tools

- **Android:** Android Studio Profiler → **"Capture System Activities"** (System Trace).
  The standalone `systrace` CLI has been removed from platform-tools; use the Profiler.
  Export the trace and open it in **Perfetto** for a richer UI if desired.
- **iOS:** Xcode **Instruments** (Time Profiler, Core Animation).
- **In-app:** Dev Menu → **Show Perf Monitor** for a live JS-FPS / UI-FPS readout — use
  it first to decide which thread to investigate.
- **Native CPU hotspots (Android):** Android Studio Profiler →
  **"Find CPU Hotspots (Java/Kotlin Method Recording)"** (not the Callstack Sample
  variant). Slower and less precise on timing, but shows *which native methods* dominate
  a frame. Keep the recording short — it's resource-intensive. Export to the Firefox
  Profiler if you want an external viewer.

## Capturing an Android System Trace

1. Connect the device via USB. Open the project's `android/` folder in Android Studio.
2. Build/run the app as **profileable** and select the device.
3. Navigate to *just before* the janky animation/interaction.
4. Start "Capture System Activities", perform the interaction, then "Stop recording".
5. Inspect in Studio or export → open in Perfetto. Enable **VSync highlighting** to draw
   the 16ms frame boundaries (zebra stripes). If stripes don't appear, try another
   device — some Samsungs don't surface vsyncs well; Nexus/Pixel are reliable.
   (Navigate traces with W/A/S/D to zoom/strafe.)

## Threads that matter

On the left of the trace, find your process (it may be truncated, e.g.
`book.adsmanager` for `com.facebook.adsmanager`). Watch these rows:

- **UI thread** (named after your package or "UI Thread") — native measure/layout/draw;
  look for `Choreographer`, `traversals`, `DispatchUI`.
- **`mqt_js`** — JavaScript execution; look for `JSCall`, `Bridge.executeJSCall`.
- **`mqt_native_modules`** — native module calls (e.g. `UIManager`); look for
  `NativeCall`, `callJavaModuleMethod`, `onBatchComplete`.
- **Render thread** (Android 5+; `RenderThread`) — generates the GPU/OpenGL commands;
  look for `DrawFrame`, `queueBuffer`.

## Reading the trace: three jank signatures

A healthy 60 FPS trace shows each frame's work finishing well inside its 16ms slot —
**no thread touches the frame boundary**.

1. **JS thread runs across frame boundaries** → **the problem is in JS.** It's executing
   almost continuously and overrunning frames. Drill into the JS row to see what's being
   called. Classic finding: an event/handler (e.g. `RCTEventEmitter`) firing many times
   per frame, or an expensive re-render. Investigate why — usually product code and
   re-render control (see `react-render-optimization.md`). → fixes in
   `references/react-render-optimization.md`.

2. **UI thread / Render thread run across boundaries with long `DrawFrame`** →
   **too much GPU work.** Time is spent waiting for the GPU to drain the previous
   frame's command buffer. Mitigate by:
   - using `renderToHardwareTextureAndroid` for complex *static* content being
     animated/transformed (turn off when static);
   - making sure `needsOffscreenAlphaCompositing` is **not** enabled (it's off by
     default and greatly raises per-frame GPU load).
   → more in `references/animations-and-rendering.md`.

3. **JS thinks → native modules work → expensive UI-thread traversal, mid-interaction**
   → **you're creating new native views during the animation/scroll** (e.g. loading new
   content as the user scrolls). Mitigation: postpone creating that UI until after the
   interaction (`InteractionManager.runAfterInteractions`), or simplify the UI being
   created. There's no cheap fix — the work is real; move it out of the frame.

## From signature to next step

| Trace shows | Bottleneck | Go to |
| --- | --- | --- |
| `mqt_js` overruns frames | JavaScript | `react-render-optimization.md`, then check for heavy synchronous work / over-firing events |
| `DrawFrame` / Render thread overruns | GPU / draw | `animations-and-rendering.md` (hardware texture, alpha compositing) |
| View creation mid-interaction | Native view construction | defer with `InteractionManager`; simplify the new UI |
| Native methods dominate (hotspot recording) | Native code | inspect the specific methods; consider native-side optimization |

## Workflow reminder

Trace → identify the offending thread/signature → apply the matching fix → **re-trace**
to confirm frames now fit inside the boundaries. One good trace beats a dozen
speculative tweaks.
