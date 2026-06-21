# Animations & Rendering

Animation jank is almost always a *thread placement* problem: the animation is being
computed on a thread that's also busy. The cure is to run it where nothing blocks it —
the native/main thread — and to avoid per-frame work the GPU can't absorb.

## Where animations run (and why it matters)

- **`Animated` without the native driver** computes **every keyframe on the JS thread**.
  If the JS thread stalls (a re-render, a network callback handler), the animation
  freezes. Avoid for anything that must stay smooth under load.
- **`Animated` with `useNativeDriver: true`** ships the animation to the main thread; it
  keeps running even when JS is busy. Use this whenever possible.
- **`react-native-reanimated`** runs animation logic on the UI thread (worklets) — the
  modern default for complex/gesture-driven animation.
- **`LayoutAnimation`** uses Core Animation (iOS) / native facilities and is **unaffected
  by JS-thread or main-thread frame drops**. Great for fire-and-forget layout
  transitions. **Caveat:** it's *not interruptible* — it's for "static"
  fire-and-forget animations only. If the animation must be interruptible, use
  `Animated`.

### Native driver limits

The native driver supports non-layout properties — `transform` (translate, scale,
rotate) and `opacity`. It can't animate layout props like `width`/`height`/`top`/`left`
directly. Design animations around `transform` + `opacity`; if you must animate layout,
either use `LayoutAnimation` (fire-and-forget) or Reanimated.

## Common rendering bottlenecks → fixes

### Moving a view (scroll / translate / rotate) drops UI-thread FPS

Most common on **Android** when text with a transparent background sits on top of an
image, or any case needing **alpha compositing** redrawn each frame.

- Enable **`renderToHardwareTextureAndroid`** on the view while it's being
  animated/transformed — it promotes the view to a hardware texture so moving it is
  cheap. (On **iOS**, `shouldRasterizeIOS` does the equivalent and is on by default for
  many cases.)
- **Turn it off when the view stops moving.** Leaving hardware textures / rasterization
  on for static views blows up memory. Profile memory when using these.
- In a trace this shows up as long `DrawFrame` blocks on the UI/Render thread (GPU
  draining its command buffer) — see `profiling.md` signature #2.

### Animating an image's size drops UI-thread FPS

On iOS, changing an `Image`'s `width`/`height` **re-crops and re-scales from the
original** every frame — very expensive for large images.

- Animate **`transform: [{ scale }]`** instead of width/height (e.g. tap-to-zoom to
  fullscreen). Scale is a GPU transform; resizing is a re-render.

### Overloaded GPU / too much per-frame draw

- Avoid `needsOffscreenAlphaCompositing` (off by default; greatly increases per-frame
  GPU load). Don't enable it without a measured reason.
- Reduce overdraw, large blurs, and stacked translucent layers in animated subtrees.
- Use `renderToHardwareTextureAndroid` for complex *static* content being animated (e.g.
  navigator slide/alpha transitions).

### `Touchable` feels unresponsive on press

If `onPress` does heavy work in the same frame as the touch's opacity/highlight change,
the visual feedback is delayed until `onPress` returns (and any re-render it triggers
settles). Let the feedback paint first:

```tsx
function handleOnPress() {
  requestAnimationFrame(() => {
    doExpensiveAction();
  });
}
```

### Heavy JS work colliding with an animation

When you must animate *and* do expensive work (e.g. slide in a modal while firing
network requests and rendering its contents):

- Defer the non-critical work until the interaction finishes:

  ```tsx
  import { InteractionManager } from 'react-native';

  InteractionManager.runAfterInteractions(() => {
    // expensive setup that can wait until the animation completes
  });
  ```

- Or use `LayoutAnimation` for the transition itself so it runs natively and ignores the
  JS-thread load (remember: not interruptible).

## Navigation transitions

JS-based stack navigators animate on the JS thread and stutter when JS is busy.
**Native-stack** navigators (`@react-navigation/native-stack`) run transitions on the
main thread — prefer them for smooth, free transition animations.

## Quick rules of thumb

- Prefer `transform` + `opacity` with the native driver / Reanimated over layout
  animation on the JS thread.
- Hardware-texture / rasterize **only while moving**, then disable.
- Animate **scale**, never width/height, to resize.
- Defer heavy work with `InteractionManager`; wrap expensive `onPress` work in
  `requestAnimationFrame`.
- Native-stack navigation > JS-stack navigation for transition smoothness.
