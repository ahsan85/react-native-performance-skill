---
name: react-native-performance
description: >-
  Use this skill whenever you are building, reviewing, refactoring, debugging, or
  optimizing a React Native app — especially anything touching frame rate / jank,
  slow or choppy lists, slow navigation transitions, slow startup / bundle load,
  animation stutter, or "why is my app laggy / janky" type questions. Trigger it
  for any React Native architecture review, code review, or request to act as a
  senior React Native / mobile performance engineer, even if the user does not say
  the word "performance" (e.g. "review this screen", "why is scrolling choppy",
  "make this faster", "optimize my FlatList", "app feels laggy on Android",
  "navigation is slow", "animations freeze"). Also trigger for FlatList /
  FlashList / VirtualizedList tuning, Animated vs native-driver / Reanimated
  questions, Hermes / bundle / startup-time questions, and profiling with Android
  Studio Profiler, Perfetto, or Xcode Instruments. Grounded in the official React
  Native performance docs (reactnative.dev/docs/performance).
---

# React Native Performance Architect

Act as a world-class Senior React Native Architect and Mobile Performance Engineer.
The target is a steady **60 FPS** (or 120 on ProMotion/high-refresh devices), which
means every frame must be produced within its budget: **~16.67ms at 60 FPS**, ~8.3ms
at 120 FPS. Missing that budget on any thread = a dropped ("janked") frame the user
feels.

## The mental model: two frame rates, two threads

React Native reports **two** FPS counters (visible via Dev Menu → "Show Perf Monitor").
Diagnosing performance starts with deciding which one is dropping, because the fix is
completely different.

- **JS thread FPS** — your React app, business logic, API calls, touch handling, and
  JS-driven animations all run here. If a state update triggers an expensive re-render
  that takes, say, 200ms, you drop ~12 frames and any JS-driven animation freezes.
  Symptom: `TouchableOpacity` feels unresponsive, navigation transitions stutter, JS
  animations hitch.
- **UI / main thread FPS** — native measure/layout/draw and the Render thread (GPU
  command generation) run here. Native-stack navigation transitions and `ScrollView`
  scrolling live here, which is why they stay smooth even when JS is busy. Symptom:
  scrolling/transforming a heavy view drops frames even though JS is idle.

**Key consequence:** prefer pushing work off the JS thread. Native-stack navigators
(`@react-navigation/native-stack`), `useNativeDriver: true` animations,
`react-native-reanimated`, and `LayoutAnimation` all run on the main thread and
survive a busy JS thread.

## Workflow: always measure before optimizing

Do not guess. Follow this order:

1. **Reproduce in a release build.** Dev mode (`__DEV__ === true`) cripples JS-thread
   perf by design (extra validation, warnings, redboxes). Numbers from dev builds are
   meaningless. Always profile a release/production build on a **real, mid-range
   device** — not a flagship, not a simulator.
2. **Watch the Perf Monitor** and identify which FPS counter drops (JS vs UI).
3. **Capture a trace** to find where the 16ms goes. Android: Android Studio Profiler
   "Capture System Activities" (the standalone `systrace` tool is gone) → inspect in
   Studio or export to Perfetto. iOS: Xcode Instruments. See
   `references/profiling.md` for how to read a trace and the three telltale jank
   signatures (busy JS thread, overloaded GPU, view creation mid-interaction).
4. **Fix the identified bottleneck**, then re-measure. Re-trace to confirm the frames
   now fit inside the boundaries.

When reviewing code without a device in hand, reason explicitly about *which thread*
each suspect line loads, and flag the likely-but-unconfirmed culprits rather than
asserting certainty — then tell the user exactly what to capture to confirm.

## Quick-win checklist (apply to almost every app)

These are cheap, high-leverage, and rarely have downsides:

- **Strip `console.*` in production.** `console.log` in a bundled app is a real
  JS-thread bottleneck (worse with logger middleware like `redux-logger`). Add
  `babel-plugin-transform-remove-console` to the production Babel env — keep it even
  if *you* never log, because a dependency might. See `references/build-and-startup.md`.
- **Confirm Hermes is on** and ship bytecode; it cuts startup time and memory. See
  `references/build-and-startup.md`.
- **Use a native-stack navigator** instead of a JS-based stack so transitions run on
  the main thread.
- **Drive animations natively:** `useNativeDriver: true` for `Animated`, or use
  Reanimated. Reserve JS-driven animation only for properties the native driver can't
  handle (e.g. layout). See `references/animations-and-rendering.md`.
- **Virtualize long lists** (`FlatList`/`FlashList`, never `.map()` into a
  `ScrollView` for large data). See `references/flatlist-optimization.md`.

## Decision guide: pick the right reference

Match the symptom to the deep-dive file and load it:

| Symptom / task | Read |
| --- | --- |
| List is slow, scroll is choppy, blank areas while scrolling, large data sets, `FlatList`/`FlashList`/`VirtualizedList` tuning | `references/flatlist-optimization.md` |
| Need to find *where* time goes; reading a system trace; Android Studio Profiler / Perfetto / Instruments; "is it JS or UI?" | `references/profiling.md` |
| Janky/freezing animations, `Animated` vs native driver vs Reanimated, `LayoutAnimation`, transforms, scrolling/rotating a view drops UI FPS, animating image size, unresponsive `Touchable`, alpha compositing on Android | `references/animations-and-rendering.md` |
| Slow startup, slow first load, bundle size, Hermes, build times, `console.*` removal, lazy/inline requires | `references/build-and-startup.md` |
| Re-render storms, `memo`/`useCallback`/`useMemo`, Context fan-out, expensive renders blocking the JS thread | `references/react-render-optimization.md` |

For a broad "review my app / screen" request, skim the relevant references and produce
a prioritized report (below). For a narrow request ("optimize this FlatList"), go
straight to the matching reference.

## Common bottlenecks → fixes (fast reference)

- **Too much JS work at once (slow navigator transitions, frozen JS animations):**
  defer non-critical work with `InteractionManager.runAfterInteractions`, or use
  `LayoutAnimation` (Core Animation / native, immune to JS jank) for fire-and-forget
  transitions. Note `LayoutAnimation` is *not* interruptible — use `Animated` if it
  must be.
- **Unresponsive `Touchable` after press:** if `onPress` kicks off a heavy re-render in
  the same frame as the opacity/highlight change, wrap the heavy work in
  `requestAnimationFrame(() => { ... })` so the touch feedback paints first.
- **Moving/scrolling/rotating a view drops UI FPS** (common on Android with text over
  an image — alpha compositing per frame): set `renderToHardwareTextureAndroid` while
  animating (iOS rasterizes by default via `shouldRasterizeIOS`). Turn it **off** when
  the view is static again — leaving it on bloats memory.
- **Animating an `Image`'s width/height drops UI FPS** (iOS re-crops/re-scales from the
  original each frame): animate `transform: [{ scale }]` instead.
- **Slow `FlatList`:** implement `getItemLayout` (skips async measurement), `memo()`
  item components, stable `keyExtractor`, hoist `renderItem` out + `useCallback`, and
  tune `windowSize` / `maxToRenderPerBatch` / `initialNumToRender`. Consider FlashList
  or Legend List. Full detail in `references/flatlist-optimization.md`.

## Output format for reviews

When asked to review/refactor/optimize, structure the response so the user can act:

```
## Summary
One-paragraph verdict: is this JS-bound, UI-bound, or a render-storm? Biggest win first.

## Findings (prioritized)
1. [HIGH] <issue> — which thread it loads, why it costs frames, the fix (with code).
2. [MED]  ...
3. [LOW / nit] ...

## How to confirm
Exactly what to capture (Perf Monitor counter, which trace, which interaction) to
verify each hypothesis and measure the improvement.
```

Order by impact, not by file order. Tie every finding back to the frame budget and the
specific thread it threatens. Give concrete code, not vague advice. If a claim depends
on profiling you can't run, say so plainly instead of overstating certainty.

## Guardrails

- Never recommend an optimization without naming its trade-off (memory, complexity,
  interruptibility). E.g. bigger `windowSize` = fewer blanks but more memory.
- Don't micro-optimize before measuring; a single trace usually reveals the one fix
  that matters more than ten speculative tweaks.
- Verify version-specific API claims against current React Native docs when the user
  pins a version — defaults and APIs (New Architecture, Hermes, list components) shift
  between releases.
