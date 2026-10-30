# uikit-skill

Skills that teach AI coding agents (Claude Code and Codex) to write good UIKit code for iOS 18 and later. A skill is a small folder of instructions the agent reads before it starts work. Without one, agents tend to write UIKit screens that work but stutter, jump or lose the reader's place. With one, they get the hard parts right the first time.

The newest skills focus on one job: long lists that grow while you look at them, such as chats, feeds and logs. They are short on purpose. Agents read the main file once and rarely open anything else, so each rule is in that one file.

## Which skill do I use?

| Skill | What it is for | Tag |
|---|---|---|
| `uikit-list` | Any long, fast-scrolling list in UIKit: feeds, inboxes, catalogs, logs, comment threads, chats. New items arrive, older pages load, the view does not move. Start here. | `v1.1.0-general1` |
| `uikit-chat-list` | The same rules, worded for chat screens only. Frozen: this is the version we measured. | `v1.0.0` |
| `uikit-specialist` | A big general guide to modern UIKit (15 reference files). Swift. | `v0.3-pair` |
| `uikit-whats-new-27` | What changed in the iOS 27 SDK. Swift. Pairs with `uikit-specialist`. | `v0.3-pair` |
| `uikit-specialist-objc`, `uikit-whats-new-27-objc` | The same pair with Objective-C examples. Install these instead of the Swift pair, not beside it. | `v0.4-objc-pair` |
| `uikit-specialist-slim` | One short file with only the gotchas models miss. An experiment. | `v0.2-slim` |

Not sure? Take `uikit-list`. The two list skills are 875 and 826 words, so they cost very little to load.

## Install

Each skill is a folder with a `SKILL.md` file, in the open Agent Skills format. Clone the repo, then copy the folder you want to where your agent looks for skills.

```sh
git clone https://github.com/notolivaresf/uikit-skill.git
```

To get a frozen version, check out its tag, for example `git -C uikit-skill checkout v1.1.0-general1`.

### Claude Code

For one project:

```sh
mkdir -p .claude/skills
cp -R uikit-skill/skills/uikit-list .claude/skills/
```

For every project, copy it into `~/.claude/skills/` instead.

You can also install it as a plugin. This adds every skill in the repo:

```
/plugin marketplace add notolivaresf/uikit-skill
/plugin install uikit-specialist@uikit-skill
```

### Codex

For one project:

```sh
mkdir -p .agents/skills
cp -R uikit-skill/skills/uikit-list .agents/skills/
```

For every project, use `~/.agents/skills/`.

### Check that it loaded

- Claude Code: `claude -p "List the names of the skills available to you"` should include `uikit-list`.
- Codex: `codex exec --sandbox read-only "List the names of the skills available to you"` should include it too.

## Does it help?

We built the same UIKit chat screen many times with agents, with and without the skill, and measured each build. The full write-up is in the [uikit-vs-swiftui](https://github.com/notolivaresf/uikit-vs-swiftui) repo (`docs/round5-summary.md`). The headline, for the chat-list skill at `v1.0.0`:

- With Codex's model, the skill passed all 13 checks in every build. The build without it failed one (the long-press menu).
- Those builds finished about 37 percent faster (1,597 seconds against 2,524) and used about 27 percent fewer output tokens (55,900 against 77,000).
- On a stopwatch test of the finished app, jumping to a message took 18 ms instead of 35 ms, and loading older messages took 17 ms instead of 33 ms.
- With Claude's Opus model the result was smaller: one build, 13 of 13 checks, and the fewest tokens of any Opus build we ran. One build is one sample, so treat that as a hint.

`uikit-list` is the same skill with the chat wording made general. It has not been measured yet. The older, larger skills were never put through a benchmark.

## More

`docs/` has the reasoning: `general-list-skill.md` (how the chat skill became the general one), `thinning-plan.md` (how we cut the skill down), `uikit-chat-list-sources.md` (the evidence behind each rule) and `experiment-plan.md`.

The skills contain no Apple skill text. API facts come from Apple's public documentation, WWDC sessions and the SDK headers.

## Licence

MIT, Fernando Olivares. See `LICENSE`.
