# Sources for uikit-chat-list

The evidence trail for `skills/uikit-chat-list/` and `agents/uikit-list-reviewer.md`. It lives outside the installed skill folder on purpose (v0.6.1, 26 Oct 2026): the validation builds copy that folder in, so the skill names no build, round, repo or measured number; every rule is plain guidance, and this file keeps the tags and numbers behind each one. The tag map below lists, rule by rule, the tags each rule carried in v0.2; the tables after it resolve the tags. For reviewers of the skill only.

## Tag map (v0.6.1 rule -> v0.2 tags)

`SKILL.md`, core rules:

| Rule | Tags | Number behind the plain wording |
|---|---|---|
| 1 Edit menu | F1.7, F2.2, SDK-UITextView | |
| 2 Offset after removals | F1.4, F2.2, E1, SDK-UICollectionViewLayout | |
| 3 One section, explicit changes | F1.1, SDK-UICollectionView, WWDC21-10252 | |
| 4 Measure once, saved heights | E2, F1.5, F2.2 | |
| 5 Nothing per row | F1.2, F1.8, WWDC21-10252 | |
| 6 No shrink-to-fit | E1, F1.1 | "several times the whole list": the label cost four times the list (7.8 to 31 s) |
| 7 Jump by geometry | F1.3, E2 | |
| 8 Apply as they arrive | F1.1, F1.10 | "gains nothing on a recent phone": per-message cost equal to 50 to 80 ms batches on an iPhone 12; "bends": the 100 ms ruling (F1.10) |
| 9 Touch to frame | Findings intro, TT-10855 | |
| 10 Release strip | F1.9 | |
| 11 No layout flush in callbacks (v0.6.2) | E2-followup | the self-sizing collection layout (`E2CollectionSelf`) crashed in all three passes of text size at the bottom; exception "Expected dequeued view to be returned to the collection view ... Avoid flushing layout before returning the view" |

`SKILL.md`, sections:

| Section, line | Tags | Number behind the plain wording |
|---|---|---|
| The list, opening ("cheapest way") | E3, E1, F1.1 | |
| The trap ("common, slow-to-find bug") | F1.4, F2.2, SDK-UICollectionViewLayout | 7 to 20 minutes of debugging in each of three builds |
| Layout class, heading | F1, F1.1 | "the shape of every fast collection build" |
| Sorted tops, Fenwick tree | F1.1 S3, SC3; E3 spike | |
| `prepare()` once | WWDC18-225; F1.1 O2, O3; F1.4 O2 | O2's stream and load-older cost |
| Binary search | WWDC18-225, E3 spike | |
| `shouldInvalidateLayout` | SDK-UICollectionViewLayout, WWDC18-225 | |
| `selfSizingInvalidation` | F1.1 S3, WWDC22-10068, SDK-UICollectionView | |
| Fresh frames for anchors | F2.2 S3 | S3 lost 14 minutes |
| Not compositional / not a recycler | F1.1, F1.11, F §4 | S2 and SC2 shipped the compositional shape |
| Table alternative (v0.6.2: tie on own work, costlier under accessibility queries) | E1, E2, E2-followup | 61 against 119 lines; unpolled stream CPU table 6.92, 6.75, 6.87 s against collection 6.53, 6.63, 6.57 s (within 4 %); under XCUITest polling table 30.7 s against collection 8.25 s |
| Heights, opening | E2, E2-followup | |
| Heights, collection bullet | E2, E2-followup | self-sizing 47 ms against 54.5 ms saved (17 against 25.5 ms app work); crash in all three passes; after the crash fix pinned after only 4 of 6 text size steps at the bottom |
| Heights, table bullet | E2 | drift up to 17 pt; saved 0 pt |
| Heights, how: dictionary | E1 spike Heights.swift, F1.5 | |
| Heights, old first row | E3 | |
| Heights, same settings | F2.4 item 5 | |
| Heights, placeholders | F1.8 | |
| Pinning: read pinned | F1.1, E1 spike Pinning.swift | |
| Pinning: insert recipe | E1, WWDC18-225 | all 600 streamed messages pinned |
| Pinning: measure new rows | F1.1, SDK-UICollectionView, WWDC21-10252 | |
| Pinning: diffable | F1.1 S2, SC2; F1.3 S3; SDK-UIDiffableDataSource | |
| Older: approach (c) | E3, F1.4 O3 | 0 pt; 39 to 46 ms against 59 to 62 ms ("about a quarter cheaper") |
| Older: steps 1 to 3 | E3 spike; F1.4 SC2, SC3; E3; SDK-UICollectionViewLayout | |
| Older: fallback (a) | E3, F1.4 | 0 pt both views |
| Older: not (b) | E3 | 42 pt; 2 to 4 rows with self-sizing |
| Older: bounded window | F1.8 SC2 | the leanest build trimmed to 300 |
| Jump: recipe | F1.3 O1, O3, OC3; E2 | within 0.2 pt in every spike |
| Jump: one row off | F1.3, F2.2, F2.2 S3 | "2499 not 2500", two builds |
| Text size | F1.5 S3, SC3; SDK-UITraitCollection | S3 and SC3, the fastest text-size rows |
| Rotation | F1.6, SDK-UIViewController | drift under 0.2 pt |
| Width key | F1.2 O3 | |
| Text view settings | E1 spike RowView.swift, F1.7 | |
| Edit menu methods | SDK-UITextView, F1.7 | |
| Context menu | F1.7 O2, S2, S3; SDK-UIContextMenuInteraction | |
| Launch bullets | F1.2 S3, OC2; F1.2 OC3, O2, S3; F1.2 O3 | |
| Shortcuts | F1.10 | |
| How to work | F2.3, F2.3, E1 | |

`references/table-view.md`:

| Line | Tags | Number behind the plain wording |
|---|---|---|
| Opening ("half the list code") | E1 | 61 against 119 lines |
| Streaming cost (v0.6.2) | E2-followup | unpolled 6.8 against 6.6 s median (table 4 % above); under XCUITest `waitForExistence` polling 30.7 s against 8.25 s; a height cache by id in the table changed nothing (35.1 s); "measure with nothing polling" from the same rerun |
| Shape: data source | E1 spike E1Table/TableAdapter.swift, F1.1 | |
| Saved heights only | E1 spike, E2, SDK-UITableView | drift up to 17 pt |
| `selfSizingInvalidation` | WWDC22-10068, SDK-UITableView | |
| `reconfigureRows` | SDK-UITableView, WWDC21-10252 | |
| Pinning | E1 | pin lost after the third message, 0 of 600 followed |
| Older, approach (a) | E3, F1.4, SDK-UITableView | 0 pt saved; 1 pt self-sizing |
| Not the sum | E3 | 42 pt; content-height change 0 pt |
| Removals | F1.4, F2.2 | |
| Jump | F1.3, E2, SDK-UITableView | message 2,500 within 0.2 pt |

`references/measuring-text.md`:

| Line | Tags | Number behind the plain wording |
|---|---|---|
| Opening | E2, F §4 | |
| Pairings | F2.4 item 5, F §4 | drift under 0.2 pt; TextKit 1 in every Opus build, TextKit 2 in every Sol build and the Round 5 spikes |
| Settings bullets | E1 spike RowView.swift, RowGeometry.swift; F1.2 O3; F1.6; F2.2 S3 | |
| Measure twice | F1.3 O2, F2.2, F1.5 O3 | O2, an estimated 15 to 40 ms |
| Key by id, revision | F §3.2 outline, F1.2 SC2/SC3 | |
| Width or text size change | F1.5 | every build re-measured the window; measuring once separated the fast ones |
| Insert only new rows | F1.4 SC2 | SC2 re-measured up to 300 rows to add 40, slowest load older |
| Off the main thread | F1.2 S3, OC3, O3 | |
| Parallel measuring | F1.5 O3, F §4 | O3 49 to 44 ms at the bottom, 67 to 42 ms mid-list, with other changes; OC3 moved nothing |
| Custom drawing | F1.5 OC3 | about 130 lines |
| Jump | F1.3, E2, SDK-UITableView | |
| Older rows | E3 | approach (b), 42 pt |

`agents/uikit-list-reviewer.md` carries the same nine checks as the `SKILL.md` checklist; its sources are those of core rules 1 to 6, 9, 10 and 11 above (v0.6.2 merged rule 11 into checklist item 7, so the list stays at nine).

v0.6.2 (26 Oct 2026). From the Round 5 follow-up lane (commits 3da9ff2, dd8a2ce): core rule 11 and checklist item 7 (no layout flush in scroll, cell or invalidation callbacks); the table streaming cost is settled (tie on the app's own work, several times more under accessibility polling), stated in `SKILL.md` and `references/table-view.md` as a further reason for the collection-view default, without the unproven cause; the fixed self-sizing collection layout still lost the bottom pin, added to the heights collection bullet.

v0.2 (26 Oct 2026). E1, E2 and E3 are in. Changes from v0.1: the custom-layout collection view is the default (Fernando chose it as the base once E1 tied on cost) and the table view moved to `references/table-view.md`; the E2 and E3 rules replace the PENDING lines; the edit-menu rule now follows the `UITextView.h` fallback; the jump, pixel-rounding and parallel-layout lines were reworded to say only what the sources record.

## Our measurements

Paths are in the benchmark repo `uikit-vs-swiftui` (private).

| Tag | Source |
|---|---|
| `F1` to `F1.11`, `F2.1` to `F2.4`, `F §3.2`, `F §4` | `docs/round5-dataset-findings.md`, the numbered section. A build name after the tag (`F1.4 O3`) points to that build's row in the section, which gives the file and line in `builds/`. |
| `Findings intro` | the same doc, "A finding about the measurements themselves" (the two timing clocks) |
| `E1` | `docs/round5-results.md`, E1 (commit ba78558): table view 7.734 s, custom collection layout 7.837 s median stream CPU, 600 messages, both pinned; table chosen on 61 against 119 lines of list code (Fernando then chose the collection view as the base, recorded in `docs/round5-results-e2e3.md`); the table pin rule (`scrollToRow` inside `performWithoutAnimation`); the shrink-to-fit label cost (7.8 to 31 s) |
| `E1 spike, <file>` | `spikes/round5/SpikeKit/Sources/SpikeKit/<file>` or `spikes/round5/Apps/E1Table/TableAdapter.swift`, the code E1 measured |
| `E2` | `docs/round5-results-e2e3.md`, E2 (commit 4d173b7), iPhone 17 simulator, three passes. Text size mid-list, frame / work ms: collection saved 54.5 / 25.5, self-sizing 47 / 17; table saved 64 / 22.5, self-sizing 80 / 17. Self-sizing collection layout crashed in `UICollectionView` in text size at the bottom, all three passes. Self-sizing table lost the pin (after Reset, end of both streams) and drifted up to 17 pt after the text size change; saved heights 0 pt. Jump to 2,500 within 0.2 pt in all four apps. Stream CPU: collection 8.25 s (agrees with E1), table 30.7 s against 7.73 s in E1, unexplained and being rerun. |
| `E3` | `docs/round5-results-e2e3.md`, E3 (commit 4d173b7), load older from the bottom, first run of each pass, median of three. Collection with saved heights: (a) top row id and offset 0 pt, 59 to 62 ms; (b) inserted rows' heights 42 pt (the old first row changes height); (b') change in content height 0 pt, 66 to 83 ms; (c) layout keeps the anchor in `targetContentOffset(forProposedContentOffset:)` 0 pt, 39 to 46 ms. Table with saved heights: (a) 0 pt, (b) 42 pt. Self-sizing: (b) and (b') lost 2 to 4 rows; (a) 0 pt collection, 1 pt table. |
| `E2-followup` | `docs/round5-results-e2e3.md` (updated) and `docs/round5-handoff-followup.md`, commits 3da9ff2 and dd8a2ce. Crash cause: the adapter's `scrollViewDidScroll` refreshed a debug label that read row frames through `collection.layoutIfNeeded()`; with self-sizing rows the real cell height moves the offset (`contentOffsetAdjustment`) inside the collection view's own layout pass, so `scrollViewDidScroll` fired there and the flush ran before the dequeued cell was returned. Fix: no flush inside `scrollViewDidScroll`. After the fix: 6 text size steps at the bottom, median 40 ms, max 47 ms, no crash, pinned after 4 of 6. Stream CPU with no test attached (`simctl launch -uvsAutoStream`, `ps` CPU, three alternating passes, includes launch): table 6.92, 6.75, 6.87 s; collection 6.53, 6.63, 6.57 s. Through XCUITest (`testLiveStream` waits with `waitForExistence`, which snapshots accessibility for 30 s): table 30.7 s (E1 code too: 30.0 s), collection 8.25 s. Suspected cause, not proven: the table's accessibility support builds cells for off-screen rows. The E1 table column (7.73 s) ran under a harness fault (`--only` without `launch` measured the previously installed app) and is not reproducible. |
| `E3 spike` | `spikes/round5/Apps/R5CollectionAnchor/CollectionAdapter.swift`, the (c) layout: sorted row tops, binary search, `beginPrepend` and `targetContentOffset(forProposedContentOffset:)` |

Short build names: O1/O2/O3 are `round{1,2,3}-uikit-swift-opus`, S1/S2/S3 `round{1,2,3}-uikit-swift-sol`, OC2/OC3 and SC2/SC3 the Objective-C builds by Opus and Sol.

## Apple, checked 26 Oct 2026

| Tag | Source | What it supports |
|---|---|---|
| `WWDC18-225` | WWDC18 "A Tour of UICollectionView" (session 225) | custom `UICollectionViewLayout`; caching attributes in `prepare()`; binary search in `layoutAttributesForElements(in:)`; data source changes inside `performBatchUpdates` |
| `WWDC21-10252` | WWDC21 "Make blazing fast lists and collection views" (10252) | `reconfigureItems` reuses the existing cell; registrations created once; prefetching; image preparation and thumbnails |
| `WWDC22-10068` | WWDC22 "What's new in UIKit" (10068) | self-resizing cells on by default in iOS 16, the `selfSizingInvalidation` property, resizing animates unless wrapped in `performWithoutAnimation` |
| `TT-10855` | Apple tech talk "Explore UI animation hitches and the render loop" (10855) | the render loop from touch event to frame on screen; hitch and hitch time ratio |
| `SDK-UITextView` | iPhoneOS 26.4 SDK, `UITextView.h` | `textView:editMenuForTextInRange:suggestedActions:` (iOS 16, marked to be deprecated, so it works on every supported version); `textView:editMenuForTextInRanges:suggestedActions:` (iOS 26 only); the header: "If the delegate does not implement this method then the `textView:editMenuForTextInRange:suggestedActions:` method will be called and passed the union range instead"; nil returns the system menu; `textViewUsingTextLayoutManager:` |
| `SDK-UICollectionView` | same SDK, `UICollectionView.h` | `selfSizingInvalidation` and its three cases; `reconfigureItemsAtIndexPaths:` "more efficient than reloading" |
| `SDK-UICollectionViewLayout` | same SDK, `UICollectionViewLayout.h` | `targetContentOffsetForProposedContentOffset:` "the content offset to be applied during transition or update animations"; `shouldInvalidateLayoutForBoundsChange:` |
| `SDK-UIDiffableDataSource` | same SDK, `UIDiffableDataSource.h` | `reconfigureItemsWithIdentifiers:`; `applySnapshotUsingReloadData:` skips diffing, for resets and very large changes |
| `SDK-UITableView` | same SDK, `UITableView.h` | `heightForRowAtIndexPath`, `estimatedHeightForRowAtIndexPath`, `reconfigureRowsAtIndexPaths:`, `rectForRowAtIndexPath:`, `indexPathsForVisibleRows`, `selfSizingInvalidation` |
| `SDK-UITraitCollection` | same SDK, `UITraitCollection.h` | `registerForTraitChanges` |
| `SDK-UIViewController` | same SDK, `UIViewController.h` | `viewWillTransitionToSize:withTransitionCoordinator:` |
| `SDK-UIContextMenuInteraction` | same SDK, `UIContextMenuInteraction.h` | `contextMenuInteraction:configurationForMenuAtLocation:` (nil means no menu at that point) |

Apple documentation pages under developer.apple.com/documentation could not be read as text on 26 Oct (they render in the browser only), so header comments stand in for them.

## What the sources do not give

- A rule on how far updates may be coalesced. We have one measurement (no gain on an iPhone 12, F1.1) and one ruling that 100 ms bends the stream (F1.10). The skill says "apply as they arrive".
- A measured cost for TextKit 1 against TextKit 2 text views. Both pairings held drift under 0.2 pt; neither was timed against the other.
- Whether the simulator order holds on a phone (E5, optional, not run). E2 and E3 also ran on the simulator only.
- Why a table view costs so much more under accessibility polling. The off-screen-cells explanation is a guess (no Time Profiler run yet), so the skill states only the effect.
- A third E2 stream pass for the four E2 apps; the self-sizing collection's pin loss after the crash fix was seen in one run set only.
- Approach (c) with self-sizing rows, and load older from the middle of the list (not run in E3).
- What the pixel-grid rounding rule rests on: only S3's notes (`F2.2 S3`). The spike's `RowGeometry.swift` only `ceil`s sizes; it does not round positions.
- Whether parallel row layout helps: O3's gain came with other changes, OC3's moved no number (`F §4`).
