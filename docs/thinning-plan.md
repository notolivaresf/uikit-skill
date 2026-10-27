# Thinning uikit-chat-list (branch `thin`, from v0.10.0)

27 Oct 2026. Fernando: "make sure the skill is as thin as possible ... trim and remove as many things as we can, based on data." This branch is a proposal: not tagged, not merged, not yet built. The ablation plan at the end says how to test it.

## Words

Counted with `wc -w` over each file.

| File | v0.10.0 | `thin` |
|---|---|---|
| `skills/uikit-chat-list/SKILL.md` | 1,548 | 819 |
| `references/measuring-text.md` | 788 | 136 |
| `references/table-view.md` | 429 | 221 |
| `references/more-surfaces.md` (new) | 0 | 347 |
| `agents/uikit-list-reviewer.md` | 1,163 | 216 |
| **Total** | **3,928** | **1,739** (-56 %) |
| What a build actually receives (skill folder only, no agent) | 2,765 | 1,523 (-45 %) |
| What a build reads at turn 1 to 2 (SKILL.md) | 1,548 | 819 (-47 %) |

## Data used

- Results docs in the public benchmark repo: `docs/round5-results.md` (E1), E2/E3 follow-ups, `docs/round5-e7` to `-e12` results, `docs/round5-v07-diagnosis.md`, `docs/round5-v08`/`v09`/`v10-spikes.md` (E8, E9, E13), `docs/round5-dataset-findings.md` (F1 to F3), the preliminary E14 pass for Sol v0.10.
- Transcripts of the eight usable chat-list builds: Opus v0.6.2, v0.7, v0.8, v0.9, v0.9b; Sol v0.6.2, v0.9r, v0.10 (Sol v0.7, v0.8, v0.9 are empty: the Codex login failed).
- Code of 14 builds, read rule by rule (nothing built or run): the two true no-skill builds (`opus-uikit`, `sol-uikit`), four general-UIKit-skill builds (`*-ours-pair`, `round3-uikit-swift-*`) and the eight chat-list builds.
- Evidence keys per rule: `docs/uikit-chat-list-sources.md`.

## Five findings that drive the cuts

1. **The reviewer agent never ran.** The harness copies only skill folders into `.claude/skills` and `.agents/skills`; no build had `.claude/agents`. 0 calls in 8 transcripts, and the closing 9-item checklist (144 words) was never worked through by hand. Cut the checklist; keep the agent as an optional, short file.
2. **SKILL.md is read once, at turn 1 or 2, and never re-read; references are opened 0 to 2 times.** `measuring-text.md` was read once in every run (turns 2 to 8). `table-view.md` was read in none of the transcripts checked (the E10 and E12 write-ups say once; either way no build used a table view). Whatever must change the code has to be in SKILL.md, and short.
3. **Extra surfaces cost agent time and buy nothing in this task.** v0.10 added keyboard, right-to-left, VoiceOver and image material. Only Sol v0.10 implemented them (keyboard guide, flips, three traits, custom actions, announcements), none of which the task exercises. Sol v0.10: 96 turns, 22 builds (8 failed), 9.0 M input tokens, against Sol v0.9r: 74 turns, 11 builds (4 failed), 3.6 M. The preliminary E14 pass shows no gain on screen (jump 45, older 45, text size 43/48 ms, peak 76 MB, stream 7.49 s; v0.9 Sol 49/40/40/42 ms, 71.2 MB, 7.14 s). Moved to a reference read "only when the screen has them".
4. **Controls already do a large part of the recipe.** Both no-skill builds do: offset set by hand (C2), saved heights (C4), collection view with custom layout (L1), `prepare()` once plus binary search (L3), fresh anchors (L6), height-store resets (H1), a TextKit 1 view with an `NSLayoutManager` measurer (S1/H2/M1/M2), the pin read and restore (P1, P2), a row window (P5), older messages anchored without adding heights (O1, O2), text size and rotation re-anchoring (T2, T3), the delegate edit-menu method (S2). Those lines are cut or reduced to a clause. TextKit 1 stays as a rule because earlier skill versions steered builds away from it (v0.6 to v0.8 measured with `boundingRect` against a TextKit 2 view) and E9 measured -25 % stream CPU when it came back.
5. **The "How to work" lines were ignored and fight the prompt.** The timing line and "a change that improves a timing but not the screen is a fault" left no trace in any build; the jump still landed one row off in Opus v0.9b and Sol v0.10. Cut; only "close any simulator you started" stays.

## Evidence per rule

Columns: (a) measured effect, with source, or "none"; (b) do the no-skill controls (`opus-uikit`, `sol-uikit`) already do it; (c) applied in the eight chat-list builds, from the code (only builds whose skill version carried the rule count); (d) words in v0.10.0. Class: KEEP (stays as its own rule), MERGE (folded into another rule as a clause), MOVE (to a reference), CUT. "Thin" gives the rule number on the branch.

Rule ids: C1 to C12 are v0.10's numbered core rules; the letters after are its sections (L list, H heights, P pinning, K keyboard, O older, T text size/rotation/RTL, S selectable text, V VoiceOver, N launch, W how to work, R checklist); M are `measuring-text.md` parts.

| Id | Rule | (a) Measured effect | (b) Controls | (c) Applied | (d) Words | Class | Thin |
|---|---|---|---|---|---|---|---|
| C1 | Edit menu `[yours] + suggestedActions`, no long press over text | 3 of 4 Round 2 failures; about 25 min of debugging in Round 3 (F1.7, F2.2) | No (both) | 8/8 | 42 | KEEP | 1 |
| C2 | Set the final offset yourself; `targetContentOffset` only in batch updates | 7 to 20 min of debugging in each of three builds (F1.4); E13 callback probe: 0 calls outside batch updates | Yes | 8/8 | 49 | KEEP (the E13 trap is not obvious; ablation candidate) | 8 |
| C3 | One section, insert new ids, reconfigure changed ones | S2/SC2 stream 9.7 to 7.4 s and 15.5 to 9.6 s | No (Sol rebuilds every row) | 8/8 | 35 | MERGE | 3, 6 |
| C4 | Measure once, saved heights, not self-sizing | E2: self-sizing loses the pin, drifts 17 pt | Yes | 8/8 | 30 | KEEP (frame of the recipe) | 4 |
| C5 | Nothing per row; capped caches; no shrink-to-fit on busy labels | E1 spike: shrink-to-fit label 7.8 to 31 s | No (both) | 8/8 | 38 | KEEP | 5 |
| C6 | Cheap measure: one pass, header by `size()`, fonts once per size | E13: header -10 % of measure time | No (Sol) | 6/8 | 50 | MERGE | 4 |
| C7 | Jump by geometry, small window | E13: 81 to 41 rows, jump 49 to 33 ms; E8: offset set right after `reloadData` 65 to 33-44 ms | No (both use 81 rows) | 6/8 | 46 | KEEP | 10 |
| C8 | Flush on next main-queue turn, no timer | Mixed: E13 delay 66 to 7.5 ms but +27 % CPU; E8 timer -24 % CPU against per-message | No (both) | 1/1 (v0.10 only; v0.8 to v0.9 said timer) | 55 | KEEP, ablate | 6 |
| C9 | Incremental layout | none isolated | No (both rebuild every frame) | 7/8 | 36 | MERGE | 3 |
| C10 | Release strip settings | app 1.34 to 0.84 MB | No (both) | 8/8 | 13 | KEEP | 13 |
| C11 | No layout flush from scroll, provisioning or invalidation callbacks | E2 follow-up crash (3 of 3 passes) | No (both) | 8/8 | 37 | KEEP | 11 |
| C12 | Trim and append in one batch, insert at `oldCount - k` | E13: -16 % CPU, cells configured 678 against 1,550 | No (Opus) | 2/2 (v0.7 to v0.9 said the opposite) | 43 | KEEP | 7 |
| L1 | Collection view + custom layout | none (E1 tie with table) | Yes | 8/8 | in 124 | KEEP (one clause) | 3 |
| L2 | Pixel-snapped tops | E9: fixed the text-size check | No (Opus) | 4/4 (v0.9 on) | in 124 | KEEP | 3 |
| L3 | `prepare()` once, binary search | none | Yes | 8/8 | in 124 | MERGE (half sentence) | 3 |
| L4 | Invalidate on width change only | none | No (both always false) | 7/8 | in 124 | MERGE | 3 |
| L5 | Flip for right-to-left | none; not in task | No | 1/1 | in 124 | MOVE | more-surfaces |
| L6 | Anchors from fresh frames | none | Yes | 8/8 | in 124 | CUT | - |
| L7 | `selfSizingInvalidation = .disabled`; prefetching note | prefetch: no effect (E13 fling, mixed memory) | No (both) | 8/8 | in 124 | MERGE (flag); prefetch note MOVE | 3 |
| H1 | Empty heights on width or size change, drop old first row | none | Yes | 8/8 | 105 | MERGE | 4, 8 |
| H2 | One measurer with the drawing view's settings | with S1, E9 | Yes | 4/4 | in 105 | MERGE | 4 |
| H3 | Streaming reply | none; not in task | Yes | 8/8 | in 105 | MOVE | more-surfaces |
| H4 | Image height from stored size; thumbnails | none; task has no images to decode | n/a | n/a | in 105 | MERGE (height clause) / MOVE (thumbnails) | 4, more-surfaces |
| P1-P3 | Pin read and restore; batch inside no-animation; reconfigure | covered by C3, C4 | Yes / Yes / No (Sol) | 8/8 | 77 | MERGE | 6 |
| P4 | Diffable: edit the current snapshot | none; no chat-list build used diffable | n/a | n/a | in 77 | CUT (one clause "never a new snapshot per message" stays) | 6 |
| P5 | Window of a few hundred rows | none | Yes | 8/8 | in 77 | MERGE | 7 |
| K1-K2 | Input bar on the keyboard guide, interactive dismiss | none (review H5) | n/a (composer is drawn only) | 1 build, unasked | 41 | MOVE | more-surfaces |
| O1-O2 | Older messages anchored by `targetContentOffset`, never add heights | E3: 39-46 against 59-62 ms; adding heights drifts 42 pt | Yes | 8/8 | 81 | MERGE (shortened; ablation candidate to cut) | 9 |
| T1 | Three traits, `adjustsFontForContentSizeCategory = false` | none (review H6) | No (content size only) | 1/1 | 80 | MOVE | more-surfaces |
| T2-T3 | Text size and rotation re-anchor | via L2 (E9) | Yes | 8/8 | in 80 | MERGE (clause in 8) / MOVE | 8, more-surfaces |
| T4 | Right-to-left `.natural` | none; not in task | n/a | 1 build | in 80 | MOVE | more-surfaces |
| S1 | Read-only text in TextKit 1 | E9: -25 % stream CPU; memory -10 % with prefetch off | Yes (but skill v0.6-0.8 had pushed builds off it) | 4/4 | 110 | KEEP | 2 |
| S2 | `editMenuForTextIn` alone works | none | Yes | 8/8 | in 110 | MERGE | 1 |
| S3 | Bubble context menu nil over text | part of C1 | No (Opus) | 8/8 | in 110 | MERGE | 1 |
| S1b | Never read `.layoutManager` on the TextKit 2 input bar | none; no editable input bar in task | n/a | n/a | in 110 | MOVE | more-surfaces |
| V1-V2 | VoiceOver rows, announcements | none (review M5) | n/a | 1 build, unasked | 37 | MOVE | more-surfaces |
| N1 | First page built off main | none isolated | No (Sol) | 8/8 | 40 | KEEP (one line) | 12 |
| N2 | Cache repeating strings, compile patterns once | none isolated | No (both) | 8/8 | in 40 | MERGE | 5 |
| W1 | How to time; gaming is a fault | none; ignored, jump still one row off in 2 runs | n/a | 0 | 62 | CUT | - |
| W2 | Close simulators you started | none (machine hygiene) | n/a | n/a | in 62 | KEEP (one line) | closing line |
| R0 | 9-item self-review checklist | none; never run | n/a | 0/8 | 144 | CUT | - |
| RV | Reviewer agent | none; never installed or called | n/a | 0/8 | 1,163 | MOVE (optional, rewritten to 216 words, points at SKILL.md's numbers) | agent |
| M1-M2 | TextKit 1 pairing; settings that must match | with S1 | Yes | 4/4 | about 300 | MERGE; TextKit 2 fallback KEEP in reference | 4, measuring-text |
| M3 | Measure once, two widths, revision number | none | Yes | 8/8 | about 120 | MERGE (two widths MOVE) | 4, more-surfaces |
| M4 | Measuring on other threads | E13: no parallel gain | n/a | n/a | about 90 | MOVE | more-surfaces |
| M5 | No custom drawing | none | Yes | 8/8 | about 40 | KEEP in reference (one line) | measuring-text |
| M6 | Old first row on prepend | none | Yes | 8/8 | about 40 | MERGE | 4 |
| M7-M8 | Growing message; image rows | none; not in task | n/a | n/a | about 150 | MOVE | more-surfaces |
| TV | Table view alternative | none; 0 builds used a table | n/a | 0 | 429 | KEEP as reference, rewritten to bullets (221) | table-view |

## Structure

One skill, one main file, three small references, one optional agent:

- `SKILL.md` (819 words): 13 numbered rules in four groups (text and menus, the list, updates, build and launch). It holds everything the measured fixes need, because it is the only file every build read.
- `references/measuring-text.md` (136): only the TextKit 2 fallback for a message view that cannot be TextKit 1. The common path is now inside rule 4.
- `references/more-surfaces.md` (347): keyboard and input bar, text size traits and rotation, VoiceOver, right-to-left, image rows, a reply that grows while streaming, measuring on several threads. SKILL.md points to it "only when the screen has them".
- `references/table-view.md` (221): the table-view alternative.
- `agents/uikit-list-reviewer.md` (216): optional; reviews against SKILL.md's rule numbers instead of restating them.

Why not split into several skills: each split adds a description the agent must match and a file it must decide to open, and the data says agents open the main file once and references 0 to 2 times. The measured fixes come from rules 1 to 13 together (a build with half of them still stutters), so splitting them would let a build load one part and miss the rest. The one split that the data supports is the opposite direction: moving surfaces the task lacks out of the main file (finding 3). An agent split is not supported either: the reviewer agent was never installed or called, so a second agent would add words without a measured use; it stays optional.

## Ablation plan

The smallest set that tells whether the thin skill holds: two builds, run one at a time on a quiet machine, each scored three passes against a same-sitting control.

| Order | Arm | Model | Compare against | Pass if |
|---|---|---|---|---|
| 1 | `thin` as on this branch | Opus | Opus v0.9 (the stable baseline: all 13 checks, 62 turns, $6.90) | all 13 checks; jump, older, text size and stream within the pass-to-pass noise of v0.9; agent turns and cost no higher |
| 2 | `thin` | Sol | Sol v0.9r and Sol v0.10 | all 13 checks; timings no worse than v0.9r; turns, builds and tokens below v0.10 (96, 22, 9.0 M) |

Only if an arm fails, bisect by putting back cut groups in order of measured value, one build each: first the v0.10 "Older messages" and "Text size, rotation" sections in full (about 160 words), then the measuring-text recipe in full, then the rest.

Optional single-variable arms, after 1 and 2, Opus only, one build each:

3. **Flush timing (rule 6):** `thin` with a 0.1 s timer in place of the next main-queue turn. E13 and E8 disagree (delay against CPU), and no build has run v0.10's wording on Opus yet.
4. **Thinner:** `thin` minus the rules the no-skill controls already do (rule 9 older messages, and rule 8's text size clause), about 110 words. If it scores the same, cut them for good.

Record for every arm: skill commit and folder hash, turns, wall time, builds and failed builds, cost or tokens, which skill files were opened and at what turn, and the 13 checks with the three-pass medians.
