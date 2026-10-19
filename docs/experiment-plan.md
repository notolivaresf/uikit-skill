# Experiment plan

19 Oct 2026. How we check that the skill loads, how we will test whether it helps, and how we change it. Part (a) is done; parts (b) and (c) are plans only. No benchmark builds are started from this repo: Fernando starts those himself.

## (a) Does it load? Done, 19 Oct

Set-up: a scratch folder with `git init`, the skill copied to `.claude/skills/uikit-specialist/` and `.agents/skills/uikit-specialist/`, and nothing installed globally (`~/.claude/skills`, `~/.agents/skills` and `~/.codex/skills` have no copy).

Claude Code 2.1.299:

- `claude -p "List the names of the skills available to you..." --model haiku --setting-sources project` listed `uikit-specialist` first.
- A UIKit question ("keep the scroll position when I prepend older messages", `--allowedTools "Skill,Read"`) made the agent call `Skill` with `uikit-specialist`, then `Read` only `references/updates-and-scrolling.md`, and answer with the anchor method from that file (4 turns, 7 s). So the description triggers the skill and the reference table loads one file on demand, as intended.
- `claude plugin validate .` passes for `.claude-plugin/marketplace.json` and `plugin.json`.

Codex CLI 0.162.1:

- `codex features list` shows skill support as stable (`skill_search`, `skill_mcp_dependency_install`).
- One call, `codex exec --sandbox read-only "List only the names of the skills available to you..."`, listed `uikit-specialist` first. The session log shows Codex read our `description` and the path to our `SKILL.md` from the project's `.agents/skills`.

Finding that matters for (b): Claude Code always lists its bundled skills too (`deep-research`, `dataviz`, `code-review`, `simplify`, `run`, `verify`, `debug` and others). The benchmark's current Opus command uses `--disable-slash-commands`, which disables all skills, ours included. A permission `deny` on `Skill(<name>)` does not remove a skill from the list (checked: the init event still listed them). See "Isolation" below.

## (b) Benchmark arms (plan only)

The benchmark is [notolivaresf/uikit-vs-swiftui](https://github.com/notolivaresf/uikit-vs-swiftui): one chat screen (Campfire) built from `PROMPT.md` by an agent, built for Release, installed on the iPhone 12 and measured by the harness (`Harness/record.sh`). Level 1 (18 Oct) is the no-skills arm.

### Arms

| Arm | UIKit builds | SwiftUI builds |
|---|---|---|
| A. No skills (done, level 1) | none | none |
| B. Apple's skills | `uikit-app-modernization` | `swiftui-specialist` (+ `swiftui-whats-new-27`) |
| C. Matched skills | this skill | `swiftui-specialist` |

Arm C is the fair pair: a general skill on each side. Arm B shows what Apple's own pairing does, with a lopsided UIKit side, and needs Xcode 27 installed to export the Apple skills (Fernando's go-ahead; not installed). Arm C can run on the UIKit side without Xcode 27 if only UIKit builds are compared against arm A. Each arm runs both models (Opus and Sol), so 4 builds per arm, one at a time, as in level 1.

### Size arm (added 19 Oct)

Fernando's hypothesis: UIKit is mature and models know it, so a skill needs only a fraction of the size; Apple's SwiftUI skill is large because SwiftUI changes a lot. The size arm tests that on the UIKit side only, with the same models and the same clean clone:

| Arm | UIKit skill | Size |
|---|---|---|
| S0. No skill | none (this is arm A, level 1) | 0 |
| S1. Slim | `skills/uikit-specialist-slim` (tag `v0.2-slim`) | 1,669 words, 12 KB |
| S2. Full | `skills/uikit-specialist` (tag `v0.1`) | 8,918 words, 66 KB |

Install exactly one skill folder per build, identically for Opus and Sol, with the isolation rules below. Metrics, in the same order as "What counts as success": (1) behaviours that pass, (2) agent cost (time, turns, builds and failed builds, tokens), (3) the gap between models, (4) only then fps, hitches, launch and memory. Also record whether the skill was invoked and which rules the agent followed. Read the result as: slim is enough if S1 matches S2 on 1 to 3; the skill is worth having at all only if S1 or S2 beats S0 by more than the level 1 spread. Level 1 failure the slim skill targets: opus-uikit could not select message text. Plan only; no builds run, and none start without Fernando.

Apple's exported text is never committed to either repo. Run records hold the export command, the Xcode build number and file hashes instead.

### Isolation

Each build runs in a fresh clone with an empty config, as in level 1 (`docs/harness.md`). For a skill arm:

- Copy the arm's skill folders into the clone's `.claude/skills/` and `.agents/skills/`, identically for both models, before the agent starts.
- Claude Code: drop `--disable-slash-commands` for skill arms (it hides our skill too). The bundled Claude Code skills then appear in the list as well. Two options, to settle before the first skill run: (1) also deny the bundled skills with `permissions.deny` `Skill(<name>)` rules so they cannot run, accepting that their names stay listed, and give arm A a rerun under the same setting if we want a perfectly matched control; or (2) find a setting that hides bundled skills only (not found on 19 Oct; check the Claude Code docs and `claude --help` for the version in use). Record which one was used.
- Codex: `CODEX_HOME=<empty dir>` as before; Codex may add its own system skills to that home on first run. Record the skill list Codex reports at start.
- Keep the level 1 isolation check: store the init event's `skills` list (Claude) and the session's skill list (Codex) in each run's `meta.json`, and check that it holds exactly the arm's skills plus any known bundled ones.

### What counts as success

Level 1 UIKit builds are already near the speed ceiling: about 58 fps while flinging on the 60 Hz iPhone 12, 2 hitches, about 600 ms cold launch, 40 to 44 MB peak memory (`docs/results-l1.md` in the benchmark repo). A skill cannot buy much there. So we judge the skill, in this order, on:

1. **Behaviours that pass.** The PROMPT.md checks: position kept within 2 pt, pinned to the bottom, stream of 600 messages, first content under 1 s, text selection, context menu, plus the stack rule. In level 1, opus-uikit failed text selection; sol-uikit passed everything.
2. **Agent cost.** Wall time, turns, builds (and failed builds), tokens. Level 1: opus-uikit 2633 s, 75 turns, 22 builds (2 failed); sol-uikit 1200 s, 102 turns, 15 builds (3 failed). A skill that saves turns and failed builds is worth having even if the app is no faster.
3. **The gap between models.** Does the skill bring the weaker run up to the stronger one (for UIKit, Opus up to Sol)?
4. **Only then fps, hitches, launch time and memory.** Watch for regressions more than gains.

Also record: whether the agent invoked the skill (a `Skill` call for Claude, a read of `SKILL.md` for Codex), which reference files it read and when, and whether it followed or ignored the rule that mattered (for example, a non-editable text view for selectable text).

One round per build is noisy. If an arm C result differs from arm A by less than the spread seen in level 1, call it no difference.

### Guarding against teaching to the test

The skill must stay general: no Campfire names, no PROMPT.md numbers, no harness labels. Before each arm, grep the skill for `Campfire`, `PROMPT`, `harness` and the benchmark's message counts. A rule added because a benchmark build failed (for example, text selection) is allowed only if it is general UIKit advice that any UIKit app would want, and the change log says which run prompted it.

## (c) Iterating the skill

- **Versions.** `version` in `.claude-plugin/plugin.json` plus a git tag per release (`v0.1` is this draft). Benchmark runs use a tag, never an untagged working copy.
- **Hash.** The run record stores the tag, the commit and a hash of the skill folder:

  ```sh
  find skills/<variant folder> -type f | sort | xargs shasum -a 256 | shasum -a 256
  ```

  v0.1 (first commit): `a5ccc2dd73ee530aa28f5f82399edf72d5e95d4cd89a5e6ee93efe26a3949e84`.
- **Change log.** `CHANGELOG.md` (added with the first change after v0.1): what changed, why, and which run or review prompted it.
- **Size budget.** Keep the total within about 10 percent of the SwiftUI skill used on the other side of the comparison, and SKILL.md near 1,000 words. Record `wc -w` per file with each release.
- **Cheap checks between benchmark runs.** Before spending a full build: (1) the load check from (a); (2) a handful of short prompts (prepend without jumping, selectable text in a cell, Dynamic Type in a custom cell, image decoding in a grid) answered with and without the skill on a small model, judged by reading the answers; (3) a review prompt on a known-bad UIKit file to see whether the checklist catches its faults. These cost minutes, not builds.
- **API currency.** Each release, check the API names in the skill against Apple's developer documentation for the newest SDK (deprecations such as `textLabel` in the iOS 27 SDK) and note anything new worth adding.
