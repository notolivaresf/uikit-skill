# uikit-skill

A pair of agent skills for writing, reviewing and speeding up modern UIKit code (iOS 18 and later), for Claude Code and Codex.

**Status: v0.4-objc-pair (19 Oct 2026).** Not yet tested against a benchmark. See `docs/experiment-plan.md`.

## Variants

| Variant | Folder | Files | Size | Words |
|---|---|---|---|---|
| Specialist (v0.3) | `skills/uikit-specialist` | SKILL.md + 15 references | 130,946 bytes (128 KB) | 17,195 |
| What's new in iOS 27 (v0.3) | `skills/uikit-whats-new-27` | SKILL.md + 10 references | 59,868 bytes (58 KB) | 7,485 |
| Specialist, Objective-C (v0.4) | `skills/uikit-specialist-objc` | SKILL.md + 15 references | 170,049 bytes (166 KB) | 20,602 |
| What's new in iOS 27, Objective-C (v0.4) | `skills/uikit-whats-new-27-objc` | SKILL.md + 10 references | 63,747 bytes (62 KB) | 8,105 |
| Slim (v0.2-slim) | `skills/uikit-specialist-slim` | one SKILL.md | 12,410 bytes (12 KB) | 1,669 |
| Full single skill (v0.1) | tag `v0.1` only (the folder now holds v0.3) | SKILL.md + 8 references | 66 KB | 8,918 |
| Chat list (branch `thin`, proposal from v0.10.0, 27 Oct) | `skills/uikit-chat-list` + `agents/uikit-list-reviewer.md` | SKILL.md + 3 references; one optional read-only reviewer agent; sources in `docs/uikit-chat-list-sources.md`, cuts in `docs/thinning-plan.md` | 11714 bytes | 819 (SKILL.md), 704 (references), 216 (agent) |

### Chat list skill and reviewer (Round 5 draft)

`uikit-chat-list` is one focused skill for a long chat or feed list: streaming rows pinned to the bottom, older rows above without moving, jump to a message, text size and rotation, selectable text with the system edit menu. The whole recipe is in SKILL.md as 13 numbered rules, because agents read the main file once and rarely open references. The default list is a collection view with a custom layout; `references/measuring-text.md` covers a message view that must stay TextKit 2, `references/table-view.md` the table-view alternative, and `references/more-surfaces.md` keyboard, text size traits and rotation, VoiceOver, right-to-left, image rows and streaming replies, read only when the screen has them. The installed folder names nothing about the benchmark (no build names, rounds or measured numbers), because validation builds copy it in; the evidence for every rule (the Round 5 dataset findings, Round 5 experiment results, WWDC sessions and SDK headers) is in `docs/uikit-chat-list-sources.md`, keyed rule by rule.

`agents/uikit-list-reviewer.md` is an optional Claude Code subagent with only Read, Grep and Glob that reviews a list against SKILL.md's rule numbers and returns file:line findings. Copy it to `.claude/agents/` (or install the plugin, which picks up `agents/`); skill-only installs do not include it.

Draft status: v0.2 has the E1, E2 and E3 results (saved heights kept for correctness; the layout keeps the anchor on load older). One open check: the table view's streaming CPU disagreed between E1 and the E2/E3 runs and is being rerun, so the skill calls the streaming tie "tied in E1, being rechecked".

Counts with `cat <files> | wc -c` and `wc -w` over every `.md` in the folder, 19 Oct 2026.

The Objective-C pair (`uikit-specialist-objc` and `uikit-whats-new-27-objc`) has the same rules and the same topics as the Swift pair, with every example in modern Objective-C (nullability, lightweight generics, `weakSelf`/`strongSelf` blocks, `UICollectionViewCellRegistration`, `NSDiffableDataSourceSnapshot<Section, Item>`). Install it instead of the Swift pair in an Objective-C project, not beside it. It is about 22 percent larger than the Swift pair. Selectors were checked against the iOS 26.4 SDK headers; iOS 27 symbols were taken from the Objective-C declarations on Apple's documentation pages, and the skill says "Swift only" where Apple documents no Objective-C form (`navigationBarMinimization`, the `UIArrangementViewController` placement methods, reserved regions, `LayoutRegion.bar`, observation tracking in section providers).

The pair is the default: install `uikit-specialist` and `uikit-whats-new-27` together. It matches the shape of Apple's Xcode 27 SwiftUI pair (a specialist skill for practice and performance, and a separate skill for what changed in the newest SDK), so a UIKit and SwiftUI comparison gives both sides the same kind of help. The slim and v0.1 variants stay for later experiments on skill size.

## Why

Apple's Xcode 27 ships agent skills, including a SwiftUI specialist and a SwiftUI "what's new" skill. There is no general UIKit equivalent: the only UIKit skill Apple ships is about modernizing an app for the scene life cycle and resizing. Agents writing UIKit get no guidance of the same kind, which matters when comparing UIKit and SwiftUI results built by agents.

This pair aims to be the UIKit counterpart, written from scratch. It contains no Apple skill text. API facts come from Apple's public documentation, the iOS 27 release notes, WWDC26 session 278 and the SDK headers; each `uikit-whats-new-27` reference lists its sources in a closing comment.

## What it covers

### `uikit-specialist` (iOS 18 and later)

SKILL.md holds how to work, the core rules, one "use when" line per reference and a review checklist. References: `collection-layout`, `diffable-data`, `cells`, `updates-and-scrolling`, `list-performance`, `layout`, `navigation-and-bars`, `dataflow`, `traits-and-environment`, `text`, `animation`, `interactions`, `accessibility`, `concurrency`, `deprecated-apis`.

### `uikit-whats-new-27` (iOS 27.0 and 27.1 SDK)

| Reference | Topic |
|---|---|
| `app-lifecycle.md` | scene life cycle required, launch screen required, state restoration into foreground, scene closure confirmation |
| `resizable-apps.md` | resizable iPhone apps, `UIScreen.main`, `displayScale`, effective geometry, idiom and orientation, `UIRequiresFullScreen` discrete resizing, presented view controller traits |
| `tabs-and-sidebar.md` | `prominentTabIdentifier`, iPhone sidebars, `sidebar.isAvailable` |
| `navigation-bars.md` | `navigationBarMinimization`, scroll edge effects, edge element containers, centered search scope bars |
| `menus-and-drag.md` | `preferredImageVisibility`, Ask Siri, drag sessions without a gesture, `allowsPointerDragBeforeLiftDelay` |
| `scene-accessories.md` | `registerSceneAccessory(_:)`, external display content, camera capture accessory (27.1) |
| `text.md` | viewport layout in `UITextView` subclasses, attachment view reuse, rendering surfaces, text tables |
| `observation-and-sensors.md` | observation in section providers, `UIView` as motion and heading body, Catalyst refresh control and stepper |
| `foldables-27-1.md` | `UIArrangementViewController`, reserved regions, `UIHingeInteraction`, vertical bars (27.1) |
| `deprecations-and-fixes.md` | `canOpenURL` and other changes, behavior fixes, a grep list |

## Install

The skill is a folder in the open Agent Skills format (`SKILL.md` with `name` and `description`, plus `references/`). Copy or symlink `skills/uikit-specialist` and `skills/uikit-whats-new-27` (or, for Objective-C code, `skills/uikit-specialist-objc` and `skills/uikit-whats-new-27-objc`) into the place your agent reads skills from.

### Claude Code

Per project:

```sh
mkdir -p .claude/skills
cp -R /path/to/uikit-skill/skills/uikit-specialist /path/to/uikit-skill/skills/uikit-whats-new-27 .claude/skills/
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
cp -R /path/to/uikit-skill/skills/uikit-specialist /path/to/uikit-skill/skills/uikit-whats-new-27 .agents/skills/
```

For every project, use `~/.agents/skills/`.

### Check that it loaded

- Claude Code: `claude -p "List the names of the skills available to you"` in the project should include `uikit-specialist` and `uikit-whats-new-27`.
- Codex: `codex exec --sandbox read-only "List the names of the skills available to you"` should include both.

## Versions

Each change to the skill bumps `version` in `.claude-plugin/plugin.json` and gets a git tag (`v0.1`, `v0.2`, ...). Benchmark runs record the tag, the commit and a hash of the skill folder; see `docs/experiment-plan.md`.

## Licence

MIT, Fernando Olivares. See `LICENSE`.
