# React Native Performance — a Claude Skill

A Claude [Agent Skill](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
that turns Claude into a senior React Native architect and mobile performance engineer.
It diagnoses and fixes frame drops / jank, slow lists, choppy navigation, animation
stutter, slow startup, and re-render storms — grounded in the official React Native
performance docs.

## What it does

When you ask Claude to review, refactor, or speed up a React Native app, this skill
gives it a structured methodology instead of generic advice:

- **Two-thread mental model** — decides whether a problem is on the JS thread or the
  UI/main thread (different threads need different fixes) against the ~16.67ms (60 FPS)
  frame budget.
- **Measure-before-optimize workflow** — release builds, the Perf Monitor, and reading a
  system trace (Android Studio Profiler / Perfetto / Xcode Instruments).
- **Targeted deep-dives** loaded on demand:
  - `flatlist-optimization.md` — `getItemLayout`, `memo`, `keyExtractor`, windowSize /
    batch tuning, FlashList.
  - `profiling.md` — capturing & reading traces; the three jank signatures.
  - `animations-and-rendering.md` — native driver vs JS-driven, `LayoutAnimation`,
    hardware textures, animating `scale` not size.
  - `build-and-startup.md` — Hermes, `console.*` removal, lazy/inline requires.
  - `react-render-optimization.md` — `memo` / `useCallback` / `useMemo`, Context fan-out.
- **A prioritized review output format** (HIGH/MED/LOW findings + how to confirm each).

## Install (Claude.ai — Free, Pro, Max, Team, Enterprise)

1. Enable the runtime: **Settings → Capabilities → Code execution and file creation**
   (on Team/Enterprise this lives under **Organization settings → Skills** and an owner
   must enable it).
2. Download **`react-native-performance.skill`** from this repo (see
   [Releases](../../releases), or the file in the repo root).
3. In Claude, go to **Customize → Skills → `+` → Upload skill**, and select the
   `.skill` file.
4. Make sure the skill is **toggled on**. Done — Claude will load it automatically when
   you ask about React Native performance. You can also hit **"Try in chat"**.

> Security note from Anthropic: before enabling any skill from someone else, open the
> files and read them. This repo is plain Markdown — no scripts, no network calls, no
> dependencies — so it's safe to review in a couple of minutes.

## Install (Claude Code — one-line marketplace commands)

This repo is also a **Claude Code plugin marketplace**, so no cloning or copying files
is needed. Just run, inside Claude Code:

```
/plugin marketplace add ahsan85/react-native-performance-skill
/plugin install react-native-performance-plugin@rn-performance-marketplace
```

That's it — the skill is installed and active in the current and future sessions.

To check it's there: `/plugin list`. To update later, after you push changes:
`/plugin marketplace update rn-performance-marketplace` then reinstall.

To remove it: `/plugin uninstall react-native-performance-plugin@rn-performance-marketplace`.

### Alternative: copy the skill folder by hand (no marketplace)

If you'd rather not register a marketplace, you can still just drop the skill folder
into your skills directory:

```bash
# user-level (all projects)
mkdir -p ~/.claude/skills
cp -r plugins/react-native-performance-plugin/skills/react-native-performance ~/.claude/skills/

# or project-level (committed with the repo)
mkdir -p .claude/skills
cp -r plugins/react-native-performance-plugin/skills/react-native-performance .claude/skills/
```

## How to verify it's working

Ask Claude: **"When would you use the react-native-performance skill?"** It should
describe diagnosing jank, list/animation/startup issues, etc. If it can't, the
description needs tightening or the skill isn't enabled.

## Repo structure

```
.
├── .claude-plugin/
│   └── marketplace.json             # marketplace catalog (for /plugin marketplace add)
├── plugins/
│   └── react-native-performance-plugin/
│       ├── .claude-plugin/
│       │   └── plugin.json          # plugin manifest
│       └── skills/
│           └── react-native-performance/   # the skill source (review these files)
│               ├── SKILL.md
│               └── references/
│                   ├── flatlist-optimization.md
│                   ├── profiling.md
│                   ├── animations-and-rendering.md
│                   ├── build-and-startup.md
│                   └── react-render-optimization.md
└── react-native-performance.skill   # packaged, ready to upload to Claude.ai directly
```

## Repackaging after edits

The `.skill` file is just a zip with the skill folder at its root. After editing the
Markdown, regenerate it:

```bash
cd react-native-performance && zip -r ../react-native-performance.skill . -x '*.DS_Store' && cd ..
```

(or re-run Anthropic's `package_skill.py` if you use the skill-creator tooling).

## Credits

Built from the official React Native performance documentation
(<https://reactnative.dev/docs/performance> and its sub-pages). Not affiliated with Meta
or Anthropic.

## License

MIT — see [LICENSE](./LICENSE).
