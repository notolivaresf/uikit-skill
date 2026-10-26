# Sources for uikit-chat-list

Every rule in `SKILL.md` and `references/` carries one or more tags. This file resolves them. It is for reviewers of the skill; an agent building an app does not need to read it.

Draft v0.1 (26 Oct 2026), written before Round 5 experiments E2 and E3 finished. Lines marked **PENDING E2** or **PENDING E3** get their final wording from those results.

## Our measurements

Paths are in the benchmark repo `uikit-vs-swiftui` (private).

| Tag | Source |
|---|---|
| `F1` to `F1.11`, `F2.1` to `F2.4`, `F §3.2`, `F §4` | `docs/round5-dataset-findings.md`, the numbered section. A build name after the tag (`F1.4 O3`) points to that build's row in the section, which gives the file and line in `builds/`. |
| `Findings intro` | the same doc, "A finding about the measurements themselves" (the two timing clocks) |
| `E1` | `docs/round5-results.md`, E1 (commit ba78558): table view 7.734 s, custom collection layout 7.837 s median stream CPU, 600 messages, both pinned; table chosen on 61 against 119 lines of list code; the table pin rule (`scrollToRow` inside `performWithoutAnimation`); the shrink-to-fit label cost (7.8 to 31 s) |
| `E1 spike, <file>` | `spikes/round5/SpikeKit/Sources/SpikeKit/<file>` or `spikes/round5/Apps/E1Table/TableAdapter.swift`, the code E1 measured |
| `E2` | `docs/round5-results.md`, E2. **Not yet run.** |
| `E3` | `docs/round5-results.md`, E3. **Not yet run.** |

Short build names: O1/O2/O3 are `round{1,2,3}-uikit-swift-opus`, S1/S2/S3 `round{1,2,3}-uikit-swift-sol`, OC2/OC3 and SC2/SC3 the Objective-C builds by Opus and Sol.

## Apple, checked 26 Oct 2026

| Tag | Source | What it supports |
|---|---|---|
| `WWDC18-225` | WWDC18 "A Tour of UICollectionView" (session 225) | custom `UICollectionViewLayout`; caching attributes in `prepare()`; binary search in `layoutAttributesForElements(in:)`; data source changes inside `performBatchUpdates` |
| `WWDC21-10252` | WWDC21 "Make blazing fast lists and collection views" (10252) | `reconfigureItems` reuses the existing cell; registrations created once; prefetching; image preparation and thumbnails |
| `WWDC22-10068` | WWDC22 "What's new in UIKit" (10068) | self-resizing cells on by default in iOS 16, the `selfSizingInvalidation` property, resizing animates unless wrapped in `performWithoutAnimation` |
| `TT-10855` | Apple tech talk "Explore UI animation hitches and the render loop" (10855) | the render loop from touch event to frame on screen; hitch and hitch time ratio |
| `SDK-UITextView` | iPhoneOS 26.4 SDK, `UITextView.h` | `textView:editMenuForTextInRange:suggestedActions:` (iOS 16, to be deprecated); `textView:editMenuForTextInRanges:suggestedActions:` (iOS 26), called in its place when implemented; nil returns the system menu; `textViewUsingTextLayoutManager:` |
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
- Whether the simulator order holds on a phone (E5, optional, not run).
