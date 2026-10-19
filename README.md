# uikit-specialist

An agent skill for writing, reviewing and speeding up modern UIKit code (iOS 18 and later), for Claude Code and Codex.

**Status: draft (v0.1 full and v0.2-slim, 19 Oct 2026).** Not yet tested against a benchmark. See `docs/experiment-plan.md`.

## Two variants

| Variant | Folder | Size |
|---|---|---|
| Full (v0.1) | `skills/uikit-specialist` | 8,918 words, 66 KB: SKILL.md plus 8 references |
| Slim (v0.2-slim) | `skills/uikit-specialist-slim` | 1,669 words, 12 KB: one SKILL.md, no references |

Why a slim one: UIKit is mature and models already know it (both UIKit benchmark builds were fast with no skill). A skill should hold only what an agent gets wrong or forgets. The slim variant keeps recent API changes and deprecations, a few correctness rules (selectable text, scroll position on prepend, staying pinned, self-sizing invalidation, diffable reconfigure vs reload, main-actor rules) and a checklist, and drops general UIKit teaching. The benchmark's size arm (no skill / slim / full) will show whether the extra 7,000 words pay for themselves; see `docs/experiment-plan.md`. Install the slim one the same way as below, with the folder name `uikit-specialist-slim`.

## Why

Apple's Xcode 27 ships agent skills, including `swiftui-specialist` for idiomatic, fast SwiftUI. There is no general UIKit equivalent: the only UIKit skill Apple ships is about migrating to the scene and multi-window lifecycle, and the community skill directories list none. Agents writing UIKit get no guidance of the same kind, which matters when comparing UIKit and SwiftUI results built by agents.

This skill aims to be the UIKit counterpart: the same size and shape as a SwiftUI specialist skill, general purpose, written from scratch. It contains no Apple text.

## What it covers

`skills/uikit-specialist/SKILL.md` holds the always-on rules (always, never, review checklist) and a table that points to one reference file per topic. The agent reads a reference only when the task needs it.

| File | Topic |
|---|---|
| `references/collection-views.md` | compositional layout, diffable data sources, cell registration, content configurations, self-sizing |
| `references/updates-and-scrolling.md` | reconfigure vs reload, keeping the position when prepending, sticking to the bottom, restoring position |
| `references/performance.md` | image decoding off main, prefetching, text and layout cost, rendering, Instruments, XCTest metrics |
| `references/layout.md` | Auto Layout hygiene, safe area, keyboard layout guide, size classes, rotation, lifecycle, navigation |
| `references/text-and-traits.md` | Dynamic Type, `UIFontMetrics`, trait change registration, automatic trait tracking, colors, accessibility |
| `references/interactions.md` | context menus, text selection, edit menus, swipe actions, drag and drop, pointer and keyboard |
| `references/concurrency.md` | `@MainActor`, tasks and view controller lifetime, moving work off main, streams |
| `references/avoid.md` | old APIs and their replacements, performance and bug patterns |

Length: 8,918 words in total (SKILL.md 985). For comparison, Paul Hudson's SwiftUI Pro (MIT) is 9,581 words (SKILL.md 809 plus 12 references); Thomas Ricouard's `swiftui-ui-patterns` SKILL.md is 1,029 words. Counts with `wc -w`, 19 Oct 2026.

## Install

The skill is a folder in the open Agent Skills format (`SKILL.md` with `name` and `description`, plus `references/`). Copy or symlink `skills/uikit-specialist` into the place your agent reads skills from.

### Claude Code

Per project:

```sh
mkdir -p .claude/skills
cp -R /path/to/uikit-skill/skills/uikit-specialist .claude/skills/
```

For every project, copy it to `~/.claude/skills/` instead.

As a plugin (this repo is also a plugin marketplace):

```
/plugin marketplace add notolivaresf/uikit-skill
/plugin install uikit-specialist@uikit-skill
```

The repo is private, so the marketplace add needs GitHub access to it.

### Codex

Per project:

```sh
mkdir -p .agents/skills
cp -R /path/to/uikit-skill/skills/uikit-specialist .agents/skills/
```

For every project, use `~/.agents/skills/`.

### Check that it loaded

- Claude Code: `claude -p "List the names of the skills available to you"` in the project should include `uikit-specialist`.
- Codex: `codex exec --sandbox read-only "List the names of the skills available to you"` should include it.

## Versions

Each change to the skill bumps `version` in `.claude-plugin/plugin.json` and gets a git tag (`v0.1`, `v0.2`, ...). Benchmark runs record the tag, the commit and a hash of the skill folder; see `docs/experiment-plan.md`.

## Licence

MIT, Fernando Olivares. See `LICENSE`.
