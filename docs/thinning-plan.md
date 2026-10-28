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

## thin2 rule use

27 Oct 2026, read from the generated code (nothing built or run for this section). Subjects: the two thin2 builds (E20 Opus, E21 Sol), the thin and thin-timer builds for corroboration (E16 Opus, E17 Sol, E19 Sol; the E18 Opus tree is no longer on disk), and the controls: the two no-skill Round 1 UIKit builds and the two Round 2 builds with the general UIKit skill pair (the Round 3 control folders are the Round 2 trees). The Apple-skill builds are SwiftUI and say nothing about these rules. Method: grep for the API each rule names, then a read of the update path in E20 and E21. "Yes" means the code does what the rule says; "ctl" is how many of the four controls already do it.

### Rules of thin2 SKILL.md

| thin2 rule | E20 Opus | E21 Sol | E16 / E17 / E19 | Controls | Verdict |
|---|---|---|---|---|---|
| 1 Edit menu: your actions + `suggestedActions`, no long press | Yes | Yes | Yes, Yes, Yes | 2 of 4 (Opus no-skill uses a long-press recognizer, Sol no-skill and the Sol pair lack the system actions) | Keep |
| 2 TextKit 1 view | Yes | Yes | Yes, Yes, Yes | 1 of 4 (Sol no-skill) | Keep the TextKit 1 call. Cut the list "not editable, selectable, no scrolling, zero inset, zero padding": the inset and padding lines are in all 10 builds, controls included |
| 2 pointer to `measuring-text.md` | not opened | not opened | not opened | n/a | Cut file (136 words): opened in 0 of 6 thin-family transcripts |
| 3 collection view, own layout, one section, keyed by id | Yes | Yes | Yes | 4 of 4 use a custom layout | Keep as the frame (one clause) |
| 3 `selfSizingInvalidation = .disabled` | Yes | Yes | Yes | 0 of 4 | Keep |
| 3 pixel snap with `displayScale` | Yes (13 uses) | Yes | Yes | 1 of 4 (Opus no-skill: none) | Keep (measured fix for the text-size rows) |
| 3 binary search; invalidate on width change only | Yes | Yes | Yes | 4 of 4 (every control binary-searches; all have the width-only check except the Sol pair) | Cut |
| 3 append adds frames, never recompute every top | Yes | Yes (`update` re-measures only from the first changed row) | Yes | not isolated | Keep |
| 3 pointer to `table-view.md` | n/a | n/a | n/a | 0 table views in any build | Cut file (221 words), cut pointer |
| 4 measure once, saved heights by id and width | Yes | Yes | Yes | 4 of 4 | Keep one sentence (the self-sizing warning carries the measured loss) |
| 4 `NSLayoutManager` on the exact string, inner width, rounded up | Yes | Yes | Yes | 3 of 4 | Keep one clause; cut "zero inset and padding" and "hand that string to the cell" (controls do both) |
| 4 one layout pass, header by `size()`, fonts once per size | no `size()` call | `size()` 3 uses | 2 of 3 | 1 of 4 | Cut: Opus E20 passes without it; the one measurement was -10 % of measure time |
| 4 drop the old first row's height on older load | Yes (prepend recomputes the row) | Yes (`measurer.remove(first.id)`) | Yes | not isolated | Keep |
| 5 no formatters, regexes, images, subviews per row; capped caches | Yes (one `DateFormatter` in a static helper) | Yes (formatters built once) | Yes | 0 to 2 of 4 build regexes per call | Keep |
| 5 no `adjustsFontSizeToFitWidth` | Yes (0) | Yes (0) | Yes (0) | 3 of 4 use it | Keep |
| 5 no `draw(_:)` in cells | Yes (0) | Yes (0) | E17 1, E19 4 (before the rule) | 3 of 4 draw | Keep (E21 memory went from +50 % to -4 %) |
| 5 plain colour, not bitmaps, for placeholders | not visible in code | not visible | not visible | not visible | Cut: no evidence either way |
| 6 0.1 s timer flush | Yes | Yes | E16, E17 next-turn (rule then said so); E19 Yes | 0 of 4 (one has an `asyncAfter` for the scripted stream only) | Keep |
| 6 `performBatchUpdates` in `performWithoutAnimation`, insert and reconfigure, no `reloadData` per message | Yes | Yes | Yes | 1 of 4 batches at all | Keep |
| 6 "much cheaper than one per message" | n/a | n/a | n/a | n/a | Cut (reason, not instruction) |
| 7 trim in the same batch, inserts at `oldCount - k` | Yes (`oldCount - trim`) | Yes | Yes | 0 of 4 | Keep |
| 7 reconfigure in its own batch before a trim | Yes (`refreshRows`) | Yes (`changeRows`) | E19 Yes | 0 of 4 | Keep |
| 7 "keep a few hundred rows" | Yes | Yes | Yes | 4 of 4 (the kit's window constant is in the prompt) | Cut |
| 8 you set the offset, never UIKit | Yes (`setOffset` after every batch) | Yes | Yes | 4 of 4 set `contentOffset` by hand | Cut as its own rule |
| 8 `targetContentOffset` only inside batch updates | Yes (layout, pending anchor) | Yes (layout, `restore`) | Yes | 0 of 4 use it | Keep, folded into the older-messages rule |
| 9 older: insert without animation, new page only, layout keeps the row | Yes | Yes | Yes | 0 of 4 use the layout hook | Keep (load older 17 to 23 ms against 38 to 43 in the controls) |
| 10 jump: window 10 above / 30 below, `reloadData`, offset at once, one `layoutIfNeeded` | Yes (10 / 30) | Yes (10 / 29) | Yes | 0 of 4 (window of about 80 rows) | Keep |
| 11 no layout flush from scroll, provisioning or invalidation | Yes | Yes | Yes | 4 of 4 (no `layoutIfNeeded` in any of those callbacks in any of the 10 builds) | Cut |
| 12 data started before `UIApplicationMain`, no placeholder | Yes | Yes (`main.swift`) | E17, E19 no | 0 of 4 | Keep |
| 12 check a guessed width against the real one | Yes (width compared in the measure path) | Yes (`widthChanged`) | not isolated | not isolated | Keep (one clause) |
| 13 strip settings | Yes | Yes | Yes | 0 of 4 | Keep |
| Closing line: close the simulator you started | no extra simulator | none | none | n/a | Keep (machine hygiene) |

### Reference files and the agent

| File | Opened in the 6 thin-family transcripts (E16 to E21) | Did the code follow it? | Verdict |
|---|---|---|---|
| `references/more-surfaces.md` (347 words) | Every build, once, at turn 4 to 28 | The task has no keyboard, input bar, VoiceOver, right-to-left or image rows. Sol builds still added custom accessibility actions and a flip for right-to-left (E17, E19, E21) and, in E17, a keyboard guide: none of the Sol controls did, and none of it is scored. The text size and rotation bullets: Opus E20 hooks the size category and `viewWillTransition`; the Opus no-skill build hooks the size category too and does not handle rotation | Cut. Extras cost Sol agent time, which is the one bar Sol misses. One measured clause moves into rule 3: prefetching off (all six thin-family builds turn it off, no control does; the only place the skill said anything about it was this file) |
| `references/measuring-text.md` (136 words) | 0 of 6 | n/a, a fallback for a TextKit 2 message view; no build needed it | Cut |
| `references/table-view.md` (221 words) | 0 of 6 | n/a, no build used a table view | Cut |
| `agents/uikit-list-reviewer.md` (216 words) | 0 of 6 (never installed in a build) | n/a | Cut: it quotes rule numbers that this change renumbers |

### The cut group (branch `thin3`)

SKILL.md 829 to 621 words; folder 1,533 to 621 words (the three references gone); with the agent, 1,749 to 621.

Cuts, all in one commit: rule 11; rule 8's offset half (folded: its `targetContentOffset` sentence now ends the older-messages rule); rule 2's attribute list; rule 3's binary search, width-only invalidation and table-view pointer; rule 4's single-pass/`size()`/fonts sentence and two clauses; rule 5's placeholder clause; rule 6's reason; rule 7's "few hundred rows"; the pointer line to `more-surfaces.md`; the three references; the agent. Added: `isPrefetchingEnabled = false` in rule 3. Rules renumbered 1 to 11.

If a build fails, the map back: a fault in the stream or the pin rows maps to rule 6 or 7 (nothing of theirs was cut except a reason and a sizing hint); load older or jump rows map to rule 8 (offset half cut); text size or rotation rows map to the removed `more-surfaces.md` bullets or the `size()` and fonts sentence in rule 4; memory rows map to the prefetch clause or rule 5; a menu or selection failure maps to the rule 2 attribute list.

## Agent time

27 Oct 2026, read from the six build transcripts (E20, E22 Opus with thin2 and thin3; E21, E23 Sol with thin2 and thin3; E24 Opus and E25 Sol with no skill). Method: every tool call in order, each charged with the model time before it plus its own run time, by what the call was. "Own UI tests" is writing, building and running the agent's own XCUITest target plus reading its output; "reference study" is pixel measuring of the given screenshots. Seconds; columns add up to the wall time.

| Phase (s; the hung row is part of the row above) | E20 Opus thin2 | E22 Opus thin3 | E24 Opus none | E21 Sol thin2 | E23 Sol thin3 | E25 Sol none |
|---|---|---|---|---|---|---|
| Reading (prompt, skill, kit, logs) | 158 | 39 | 187 | 156 | 421 | 197 |
| Reference study (screenshot pixels) | 55 | 590 | 409 | 5 | 299 | 78 |
| Writing app code | 818 | 454 | 455 | 188 | 672 | 306 |
| Builds | 155 | 91 | 105 | 116 | 77 | 51 |
| Failed builds | 0 | 0 | 0 | 93 | 0 | 43 |
| Simulator by hand (install, launch, screenshot) | 30 | 33 | 282 | 102 | 117 | 93 |
| Looking at own screenshots | 10 | 76 | 11 | 0 | 0 | 0 |
| Own UI tests (write, build, run, read) | 1,151 | 1,423 | 634 | 898 | 759 | 1,174 |
| of which one hung test run | 600 | 601 | 0 | 0 | 0 | 0 (two runs of 613 and 672 s, but Sol kept working beside them) |
| Waiting (sleep) | 0 | 17 | 0 | 0 | 0 | 0 |
| Final report | 12 | 13 | 19 | 34 | 20 | 12 |
| **Wall** | **2,389** | **2,737** | **2,102** | **1,593** | **2,366** | **1,954** |
| Wall without the hung run | 1,789 | 2,136 | 2,102 | 1,593 | 2,366 | 1,954 |
| `xcodebuild test` runs | 9 | 15 | 9 | 12 | 11 | 8 |

What the table says:

1. **The biggest single detour is one hung `xcodebuild test` call of about 600 s**, in both Opus skill builds. The Sol no-skill build also had two test runs that never ended, but it ran them beside other commands, so they cost it little idle time. The Opus agent ran its own UI test in the foreground with a 10-minute tool timeout; the test never ended, so the agent sat out the whole timeout. Without it, E22 is 2,136 s (inside the 2,454 s bar) and E20 1,789 s. No-skill Opus E24 had no hang, which is most of its 635 s lead over E22.
2. **In E22 the hang comes from rule 1.** The rule says a menu without `suggestedActions` loses Copy; the agent checked Copy by long-pressing a text view in a UI test, tapping Copy and reading `UIPasteboard.general.string` from the test. Reading another app's pasteboard puts up the paste prompt, which nobody answers: the log stops right after the menu item list (`Reply, Copy, Look Up, Translate`). The menu checks in E22 took 626 s of its own-test time against 139 s in E24 (which checked only that Reply shows, never read the pasteboard, and fails text selection). In E20 the hung call was a full-suite run (no `-only-testing`) that did not exit after a failing test; its cause is not in the log, but the cure is the same: a per-test time limit.
3. **Every build, skill or not, spends 634 to 1,423 s on its own UI tests**, 9 to 15 runs; the prompt never asks for tests. That is model behaviour, not a rule, and the skill cannot remove it without becoming a testing guide.
4. **Sol thin3's extra 773 s over thin2 is visual matching**: a rendering test that compares its rows to the given screenshots, pixel study, and fetching the original icons from the web. None of it maps to a skill rule (the skill says nothing about looks); Sol no-skill did much less of it. Treat it as the spread between Sol builds until a second build says otherwise.
5. No rule made an agent re-measure or redo the list code: builds 6 to 8 per run, failed builds only in E21 and E25, writing time within the usual spread.

### The thin4 edit (branch `thin4`, tag `v0.11.4-thin4`)

The smallest edit that removes the biggest detour, two sentences, no rule changed:

- Rule 1 ends: "Copy is the system's item: if the menu lists it, it works; never read `UIPasteboard` from a UI test, it waits on the paste prompt."
- Before the closing line: run `xcodebuild test` for the test you changed (`-only-testing:`) with `-test-timeouts-enabled YES -default-test-execution-time-allowance 60`, so a test stuck on a system prompt fails in a minute instead of holding the run.

SKILL.md 621 to 677 words. Expected effect: Opus wall time down by the hung run (about 600 s) with no change to the app code; the screen rows should not move. E26 (Opus) tests it; E27 (Sol) follows.

### The thin5 edit (branch `thin5`, tag `v0.11.5-thin5`)

Why: Sol's top-row pick had no tolerance, so after the offset was restored the row above (a float remainder of pixel rounding) was named the top row, and text-size anchoring failed. Every build that passed had some margin or tolerance.

Rule 8 gains two sentences: the top row counts only once its visible content is below the top edge (ignore transparent margin and sub-point rounding remainders, small tolerance), and the same rule is used to capture the anchor and to read the top row after restoring. No new rule. The timing/stopwatch candidate is not added.

SKILL.md 677 to 721 words.

### The thin6 edit (branch `thin6`, tag `v0.11.6-thin6`)

Why: the round 5 agent-time note found that Sol writes one test per rule and reruns the whole test target, which costs 600 to 1,250 s of model time per build.

Two edits. A line before the numbered rules: "These rules say how to build it; they are not a list to test." The closing sentence now says to check each behaviour once the screen works and rerun only the failed tests by name (`-only-testing:`), keeping the time-limit flags and "Close any simulator you started". Rule 1's pasteboard sentence and rule 8's tolerance wording are kept. No rule added or removed.

SKILL.md 721 to 730 words.

### The thin7 edit (branch `thin7`, tag `v0.11.7-thin7`)

Why: the round 5 thin6 time breakdown (docs/round5-thin6-time-breakdown.md in the benchmark repo) proposed four small edits.

Four edits. Rule 6 keeps "pinned to the bottom" as state (set at the bottom, cleared only on a drag away, never worked out again from the offset) and says a pinned flush that also trims needs no second offset fix. Rule 7 trims rarely: the window runs a page past its limit before trimming back, so most arrivals are plain appends. Rule 8's target offset returns the bottom while pinned and otherwise keeps the first visible row. A line after "not a list to test" says to draw icons with system symbols and not download or import image assets or an app icon. The pasteboard sentence in rule 1 and rule 8's tolerance wording are kept. No rule added or removed; the optional faster-arrivals sentence is not added.

SKILL.md 730 to 826 words.
