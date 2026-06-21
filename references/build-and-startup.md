# Build, Bundle & Startup Performance

This covers two related things users conflate: **runtime startup time** (how fast the
app launches and becomes interactive) and **developer build time** (how fast you can
iterate). Both are addressable; keep them distinct in your diagnosis.

## Always: production build, Hermes, no console

### Strip `console.*` in production

`console.log` (and logger middleware like `redux-logger`) is a genuine JS-thread
bottleneck in a bundled app. Remove all `console.*` calls from release builds with
`babel-plugin-transform-remove-console`:

```bash
npm i babel-plugin-transform-remove-console --save-dev
```

```json
{
  "env": {
    "production": {
      "plugins": ["transform-remove-console"]
    }
  }
}
```

Keep this even if your own code never logs — a third-party dependency might log on every
render or event.

### Use Hermes

Hermes is the default JS engine on current React Native and is purpose-built for fast
startup and lower memory on device. It precompiles JS to bytecode at build time, so the
app skips parsing/compiling JS at launch. Confirm it's enabled (it is by default on
recent versions) and that you're shipping the Hermes bytecode bundle. If a project
disabled it, re-enabling is usually a large, cheap startup win.

### Measure in release on a real device

As everywhere: dev-mode startup numbers are meaningless. Measure cold start on a real,
mid-range device with a release build.

## Reducing startup / time-to-interactive

- **Shrink the bundle / defer work off the critical path.** The less JS that must be
  parsed and executed before first render, the faster startup. Move non-essential
  modules out of the initial path.
- **Inline / lazy requires:** defer evaluating expensive modules until first use rather
  than at bundle load. Metro's `inlineRequires` and lazily `require()`-ing heavy modules
  inside the function that needs them keeps them out of startup.
- **Avoid heavy synchronous work at module top-level** (big computations, large JSON
  parses, eager singletons). Anything at module scope runs during startup.
- **Defer post-launch work** with `InteractionManager.runAfterInteractions` or after the
  first frame, so the first screen paints before secondary setup runs.
- **Trim native startup:** minimize work in app/Activity startup and in eagerly-loaded
  native modules.

## Reducing developer build time

These speed up *your* iteration, not the shipped app:

- **Don't build for every architecture in development.** Restrict to the ABI of the
  device/emulator you actually run (e.g. a single `abiFilters` / arch) instead of all of
  them — building all architectures multiplies native compile time.
- **Build only the variant you need** (debug for the running device).
- **Enable native build caching** where available so unchanged native code isn't
  recompiled.
- **Keep dependencies lean** — each native dependency adds compile time.

(Specifics like Gradle/Xcode flags and cache setup change across RN versions; verify
against the current "Speeding up your Build phase" doc for the user's version.)

## Checklist

- [ ] Profiling/measuring in a **release** build on a real mid-range device?
- [ ] Hermes enabled and bytecode shipped?
- [ ] `babel-plugin-transform-remove-console` in the production Babel env?
- [ ] No heavy synchronous work at module top-level?
- [ ] Expensive modules lazy/inline-required, not loaded at startup?
- [ ] Post-launch setup deferred until after first interaction/frame?
- [ ] (Dev) building only the single needed architecture/variant with caching on?
