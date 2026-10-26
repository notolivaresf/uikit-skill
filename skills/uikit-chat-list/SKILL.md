---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - messages streaming in while pinned to the bottom, older messages loading above without the view moving, jumping to a message, Dynamic Type and rotation without losing the reader's place, selectable message text with the system edit menu plus your own actions, and fast launch with low memory. Use for any UICollectionView or UITableView screen that grows at both ends, for chat rooms, comment threads, logs and timelines, and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

The recipe for one kind of screen: a long list that grows at the bottom while the reader is pinned there, grows at the top when older items load, and must never move under the reader's eye. UIKit only. Each rule ends with its evidence tag; `SOURCES.md` in this folder resolves the tags.

## Core rules (read these even if nothing else)

1. **Edit menu: your actions first, then the system's.** Return `UIMenu(children: [reply] + suggestedActions)` from the text view delegate. Never build a menu without `suggestedActions`, never append your action at the end (it lands on the menu's second page), and never put a long-press recognizer over selectable text. [F1.7] [F2.2] [SDK-UITextView]
2. **Never let UIKit choose the offset after rows are removed or shrink.** It animates the offset back into range and undoes your pin or anchor. Supply the final offset yourself: from the layout's `targetContentOffset(forProposedContentOffset:)`, or set it in the same no-animation pass as the change. [F1.4] [F2.2] [E1] [SDK-UICollectionViewLayout]
3. **One section, stable message ids, explicit row changes.** Insert only the new rows, reconfigure only the changed ids, never rebuild the whole list or snapshot per message, never one section per message. [F1.1] [SDK-UICollectionView] [WWDC21-10252]
4. **Measure each message once and keep the height by message id,** for the current width and text size. Do it for correctness, not speed: self-sizing rows lost the pin, drifted or crashed. [E2] [F1.5] [F2.2]
5. **Nothing per row that most rows do not show:** no formatters, regexes, images or extra subviews created in configure; no decoded bitmaps for placeholders; every cache capped. [F1.2] [F1.8] [WWDC21-10252]
6. **No shrink-to-fit text in anything updated often** (`adjustsFontSizeToFitWidth` on a label redrawn per message cost four times the whole list). Pick the font once. [E1] [F1.1]
7. **Jump by geometry, not by hope:** load a page around the target, then place it from the list's own frame for that row, without animation. [F1.3] [E2]
8. **Apply every message as it arrives.** Do not hold updates on a timer: one update per message cost the same as 50 to 80 ms batches on an iPhone 12. [F1.1] [F1.10]
9. **Time from the touch to the frame,** if you time anything: start at the event's `timestamp`, stop at the next display-link frame. [Findings intro] [TT-10855]
10. **Release builds strip:** `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`. [F1.9]

## The list: a collection view with a custom layout

Build on a `UICollectionView` with your own `UICollectionViewLayout` subclass, one section, rows keyed by message id. Its layout can keep the reader's place by itself on load older, the cheapest approach measured. [E3] [E1] [F1.1]

**The trap to know first.** After rows are removed or content shrinks (trimming, Reset, a smaller text size), the collection view clamps the offset into range and animates it back, and the pin or anchor is gone, often intermittently. Return the final, in-range offset from `targetContentOffset(forProposedContentOffset:)` (the header: "the content offset to be applied during transition or update animations"); for a large removal, `reloadData` plus that offset rather than an animated delete. This one fault cost 7 to 20 minutes of debugging in each of three builds. [F1.4] [F2.2] [SDK-UICollectionViewLayout]

The layout class, the shape of every fast collection build: [F1] [F1.1]

- Keep row tops in an array sorted by y, built from saved heights (a height index such as a Fenwick tree when prepends are frequent: S3, SC3). [F1.1 S3, SC3] [E3 spike]
- `prepare()` does the work once per invalidation; appends compute frames only for the new rows. Recomputing every frame on each invalidation was O2's stream and load-older cost. [WWDC18-225] [F1.1 O2, O3] [F1.4 O2]
- `layoutAttributesForElements(in:)` binary-searches the tops for the rect instead of filtering every row. `layoutAttributesForItem(at:)` reads the cached frame; `collectionViewContentSize` is the last row's bottom. [WWDC18-225] [E3 spike]
- `shouldInvalidateLayout(forBoundsChange:)` is true only when the width changes. [SDK-UICollectionViewLayout] [WWDC18-225]
- `selfSizingInvalidation = .disabled`: heights come from your measurer, so a cell's content change must not resize it behind the layout's back. [F1.1 S3] [WWDC22-10068] [SDK-UICollectionView]
- Read anchors from fresh frames, not attributes cached before the update: S3 lost 14 minutes to that. [F2.2 S3]
- Not a compositional layout with one section per message (the slowest UIKit shape measured: S2 and SC2 shipped it), nor a hand-rolled `UIScrollView` recycler (no cell reuse, no accessibility containers). [F1.1] [F1.11] [F §4]

**Alternative: a table view.** A `UITableView` with a plain data source takes about half the list code (61 lines against 119) and streamed for the same CPU in E1 (7.73 against 7.84 s, 1.3 % apart); that tie is being rechecked, because the table's stream took 30.7 s in the later E2/E3 runs. It has no layout-owned anchor. If you choose it, follow `references/table-view.md`. [E1] [E2]

## Heights

Keep saved heights, for correctness: self-sizing rows were not slower, they were wrong. [E2]

- Collection view, text size change mid-list: self-sizing 47 ms against 54.5 ms saved (17 against 25.5 ms of app work), but the self-sizing layout crashed inside `UICollectionView` in the text-size-at-the-bottom test in all three passes. [E2]
- Table view: self-sizing lost the bottom pin (after Reset and at the end of every stream) and drifted up to 17 pt after a text size change. Saved heights: 0 pt, always pinned. [E2]

How:

- Work out each row's height from the message, the width and the text size, and keep it in a dictionary by message id. Empty it when the width or text size changes; drop one entry when one message changes. [E1 spike, Heights.swift] [F1.5]
- When older rows arrive, drop the saved height of the old first row too: it is now threaded under a new neighbour and changes height. [E3]
- Measure with the same text settings the view draws with. See `references/measuring-text.md`. [F2.4 item 5]
- Image placeholders are a background colour on a view of the final size, never a bitmap. [F1.8]

## Live updates and pinning

- Read "pinned" before the update (the last row within a small distance of the bottom), apply the change, then restore the pin. [F1.1] [E1 spike, Pinning.swift]
- Insert with `insertItems(at:)` inside `performBatchUpdates`, inside `UIView.performWithoutAnimation`, data source changed in the same closure; then a plain `setContentOffset` to the bottom. That held all 600 streamed messages pinned. [E1] [WWDC18-225]
- Measure only the new rows. Changed messages (reactions, edits) are `reconfigureItems(at:)` for those ids, which keeps the existing cell. [F1.1] [SDK-UICollectionView] [WWDC21-10252]
- With a diffable data source, edit the current snapshot (append, `reconfigureItems`) rather than building a new one per message; `applySnapshotUsingReloadData` is for a reset such as a jump. [F1.1 S2, SC2] [F1.3 S3] [SDK-UIDiffableDataSource]

## Older messages above

Let the layout keep the anchor (approach c): 0 pt drift, 39 to 46 ms against 59 to 62 ms for the next best, about a quarter cheaper, because UIKit gets the right offset before it makes cells for a wrong place. [E3] [F1.4 O3]

1. Before the insert, the layout notes the first visible row's message id and its distance from the top of the visible area. [E3 spike]
2. Measure only the new page; drop the old first row's height; insert without animation and invalidate the layout. [F1.4 SC2, SC3] [E3]
3. `targetContentOffset(forProposedContentOffset:)` returns that row's new top minus the saved distance, once, then forgets the anchor. [E3 spike] [SDK-UICollectionViewLayout]

Simple fallback (approach a, also the table view's): record the top row id and its distance, insert, set the offset that puts it back in the same no-animation pass. 0 pt with saved heights in both views. [E3] [F1.4]

Do not add the inserted rows' total height to the offset (approach b): it drifted 42 pt, because the old first row changes height when older rows arrive and that sum misses it. With self-sizing rows it lost the place by 2 to 4 rows. [E3]

Keep the loaded window bounded (a few hundred rows) and trim with the trap rule above; the leanest build trimmed to 300. [F1.8 SC2]

## Jump to a message

- Replace the loaded window with one page around the target (about 10 rows above, 30 below), `reloadData` or `applySnapshotUsingReloadData`, then set the offset from the layout's own frame for the target, rounded to the pixel grid. No animation. Every Round 5 spike landed on the target within 0.2 pt this way. [F1.3 O1, O3, OC3] [E2]
- Recorded symptom: a jump that landed one row off ("2499 not 2500", two builds). The fix that held: the offset from the layout's frame for that row, not `scrollToItem(at:at:animated:)` alone. Unrounded, a sliver of the row above shows. [F1.3] [F2.2] [F2.2 S3]

## Text size and rotation

- On a text size change (register for `UITraitPreferredContentSizeCategory`) or a new width: capture the anchor, empty the saved heights, re-measure, invalidate, and return the anchored offset from `targetContentOffset(forProposedContentOffset:)`; S3 and SC3 did this, the fastest text-size rows measured. [F1.5 S3, SC3] [SDK-UITraitCollection]
- Rotation: capture the anchor in `viewWillTransition(to:with:)` and restore it in the coordinator's `animate(alongsideTransition:completion:)`, or when `viewDidLayoutSubviews` sees a new width. Both held drift under 0.2 pt. [F1.6] [SDK-UIViewController]
- Every height saved for the old width is wrong for the new one; key or empty by width. [F1.2 O3]

## Selectable text and menus

- Message text is a `UITextView` with `isEditable = false`, `isSelectable = true`, `isScrollEnabled = false`, zero `textContainerInset` and zero `lineFragmentPadding`. [E1 spike, RowView.swift] [F1.7]
- `textView(_:editMenuForTextIn:suggestedActions:)` alone works on every version: on iOS 26, when the ranges method is missing, the system calls it with the union range. `textView(_:editMenuForTextInRanges:suggestedActions:)` exists only from iOS 26. Adding it is a choice (for multi-range selections); if you implement both, return the same menu. [SDK-UITextView] [F1.7]

```swift
private func selectionMenu(_ suggested: [UIMenuElement]) -> UIMenu {
    UIMenu(children: [replyAction] + suggested)   // Reply first; the system's Copy stays in `suggested`
}
```

- The bubble's own context menu is a `UIContextMenuInteraction` whose `contextMenuInteraction(_:configurationForMenuAtLocation:)` returns nil over the text, or one attached only to the parts that are not text. [F1.7 O2, S2, S3] [SDK-UIContextMenuInteraction]

## Launch

- Start building the data off the main thread before the first view controller loads. [F1.2 S3, OC2]
- Create rarely used cell subviews lazily; cache strings that repeat (a timestamp per minute); compile patterns once. [F1.2 OC3, O2, S3]
- A width guessed before layout is fine only if checked against the real one; throw the work away when it differs. [F1.2 O3]

## Shortcuts that do not count

They improve a number, not the screen: redrawing streamed rows or a status label on a timer; timing part of an operation, or starting the clock inside the action; laying out the first screen from geometry saved by a previous launch; skipping work because the same action ran a moment ago. [F1.10]

## How to work

- Write the screen from the numbers you are given (sizes, colours, fonts) before studying screenshots pixel by pixel. [F2.3]
- Write checks once the behaviour works, one per behaviour. After a change, judge it only on a build you just installed. [F2.3]
- Measuring code must cost nothing: no shrink-to-fit labels, nothing redrawn per frame. [E1]

## Self-review checklist

Run this when the room first works and again before you stop (or call the `uikit-list-reviewer` agent, which checks the same list):

1. Edit menu returns `[yours] + suggestedActions`; no long press over text.
2. One section; no whole-list rebuild or new snapshot per update.
3. An insert or prepend measures only the new rows.
4. Removals and shrinks supply the final offset first, without animation.
5. No decoded bitmaps for placeholders; every cache capped.
6. Each text measured once, with the drawing view's settings.
7. Nothing created in configure; no shrink-to-fit label updated often.
8. Any timing starts at the touch's timestamp and stops at the next frame.
9. Release strip settings set.
