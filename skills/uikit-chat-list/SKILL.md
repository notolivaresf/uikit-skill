---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - messages streaming in while pinned to the bottom, older messages loading above without the view moving, jumping to a message, Dynamic Type and rotation without losing the reader's place, selectable message text with the system edit menu plus your own actions, and fast launch with low memory. Use for any UICollectionView or UITableView screen that grows at both ends, for chat rooms, comment threads, logs and timelines, and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

The recipe for one kind of screen: a long list that grows at the bottom while the reader is pinned there, grows at the top when older items load, and must never move under the reader's eye. UIKit only.

## Core rules (read these even if nothing else)

1. **Edit menu: your actions first, then the system's.** Return `UIMenu(children: [reply] + suggestedActions)` from the text view delegate. Never build a menu without `suggestedActions`, never append your action at the end (it lands on the menu's second page), and never put a long-press recognizer over selectable text.
2. **Never let UIKit choose the offset after rows are removed or shrink.** It animates the offset back into range and undoes your pin or anchor. Supply the final offset yourself: from the layout's `targetContentOffset(forProposedContentOffset:)`, or set it in the same no-animation pass as the change.
3. **One section, stable message ids, explicit row changes.** Insert only the new rows, reconfigure only the changed ids, never rebuild the whole list or snapshot per message, never one section per message.
4. **Measure each message once and keep the height by message id,** for the current width and text size. Do it for correctness, not speed: self-sizing rows lose the pin, drift or crash.
5. **Nothing per row that most rows do not show:** no formatters, regexes, images or extra subviews created in configure; no decoded bitmaps for placeholders; every cache capped.
6. **No shrink-to-fit text in anything updated often.** `adjustsFontSizeToFitWidth` on a label redrawn per message can cost several times the whole list. Pick the font once.
7. **Jump by geometry, not by hope:** load a page around the target, then place it from the list's own frame for that row, without animation.
8. **Apply every message as it arrives.** Do not hold updates on a timer: batching gains nothing on a recent phone and makes the stream visibly bend.
9. **Time from the touch to the frame,** if you time anything: start at the event's `timestamp`, stop at the next display-link frame.
10. **Release builds strip:** `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`.
11. **Never force layout from inside the list's own passes.** No `layoutIfNeeded`, and nothing that reads frames through it, from `scrollViewDidScroll`, cell provisioning (`cellForItemAt`, `cellForRowAt`) or a layout invalidation callback. The offset can move during the collection view's own layout pass, so these fire inside it, and a flush there crashes it ("Avoid flushing layout"). Read the layout's cached geometry instead.

## The list: a collection view with a custom layout

Build on a `UICollectionView` with your own `UICollectionViewLayout` subclass, one section, rows keyed by message id. Its layout can keep the reader's place by itself on load older, the cheapest way to do it.

**The trap to know first.** After rows are removed or content shrinks (trimming, Reset, a smaller text size), the collection view clamps the offset into range and animates it back, and the pin or anchor is gone, often intermittently. Return the final, in-range offset from `targetContentOffset(forProposedContentOffset:)` (the header: "the content offset to be applied during transition or update animations"); for a large removal, `reloadData` plus that offset rather than an animated delete. This one fault is a common, slow-to-find bug.

The layout class:

- Keep row tops in an array sorted by y, built from saved heights (a height index such as a Fenwick tree when prepends are frequent).
- `prepare()` does the work once per invalidation; appends compute frames only for the new rows. Recomputing every frame on each invalidation makes streaming and load older slow.
- `layoutAttributesForElements(in:)` binary-searches the tops for the rect instead of filtering every row. `layoutAttributesForItem(at:)` reads the cached frame; `collectionViewContentSize` is the last row's bottom.
- `shouldInvalidateLayout(forBoundsChange:)` is true only when the width changes.
- `selfSizingInvalidation = .disabled`: heights come from your measurer, so a cell's content change must not resize it behind the layout's back.
- Read anchors from fresh frames, not attributes cached before the update.
- Not a compositional layout with one section per message (the slowest UIKit shape), nor a hand-rolled `UIScrollView` recycler (no cell reuse, no accessibility containers).

**Alternative: a table view.** A `UITableView` with a plain data source takes about half the list code. On the app's own work it streams for about the same cost, but under constant accessibility queries (UI tests, assistive technology polling the screen) it costs several times more, while the collection view does not: one more reason for the default. It has no layout-owned anchor. If you choose it, follow `references/table-view.md`.

## Heights

Keep saved heights, for correctness: self-sizing rows are not slower, they are wrong.

- Collection view: self-sizing rows are slightly faster on a text size change, but a self-sizing layout can crash inside `UICollectionView` when the text size changes at the bottom of the list (core rule 11), and even without the crash it loses the bottom pin on some text size changes.
- Table view: self-sizing rows lose the bottom pin (after Reset and at the end of a stream) and drift after a text size change. Saved heights keep it exact and pinned.

How:

- Work out each row's height from the message, the width and the text size, and keep it in a dictionary by message id. Empty it when the width or text size changes; drop one entry when one message changes.
- When older rows arrive, drop the saved height of the old first row too: it is now threaded under a new neighbour and changes height.
- Measure with the same text settings the view draws with. See `references/measuring-text.md`.
- Image placeholders are a background colour on a view of the final size, never a bitmap.

## Live updates and pinning

- Read "pinned" before the update (the last row within a small distance of the bottom), apply the change, then restore the pin.
- Insert with `insertItems(at:)` inside `performBatchUpdates`, inside `UIView.performWithoutAnimation`, data source changed in the same closure; then a plain `setContentOffset` to the bottom. That keeps a long stream pinned.
- Measure only the new rows. Changed messages (reactions, edits) are `reconfigureItems(at:)` for those ids, which keeps the existing cell.
- With a diffable data source, edit the current snapshot (append, `reconfigureItems`) rather than building a new one per message; `applySnapshotUsingReloadData` is for a reset such as a jump.

## Older messages above

Let the layout keep the anchor (approach c): no drift, and about a quarter cheaper than the next best, because UIKit gets the right offset before it makes cells for a wrong place.

1. Before the insert, the layout notes the first visible row's message id and its distance from the top of the visible area.
2. Measure only the new page; drop the old first row's height; insert without animation and invalidate the layout.
3. `targetContentOffset(forProposedContentOffset:)` returns that row's new top minus the saved distance, once, then forgets the anchor.

Simple fallback (approach a, also the table view's): record the top row id and its distance, insert, set the offset that puts it back in the same no-animation pass. No drift with saved heights in either view.

Do not add the inserted rows' total height to the offset (approach b): it drifts, because the old first row changes height when older rows arrive and that sum misses it. With self-sizing rows it loses the place by several rows.

Keep the loaded window bounded (a few hundred rows) and trim with the trap rule above.

## Jump to a message

- Replace the loaded window with one page around the target (about 10 rows above, 30 below), `reloadData` or `applySnapshotUsingReloadData`, then set the offset from the layout's own frame for the target, rounded to the pixel grid. No animation. This lands on the target exactly.
- A known symptom: a jump that lands one row off. The fix is the offset from the layout's frame for that row, not `scrollToItem(at:at:animated:)` alone. Unrounded, a sliver of the row above shows.

## Text size and rotation

- On a text size change (register for `UITraitPreferredContentSizeCategory`) or a new width: capture the anchor, empty the saved heights, re-measure, invalidate, and return the anchored offset from `targetContentOffset(forProposedContentOffset:)`. This is the fastest way to handle a text size change.
- Rotation: capture the anchor in `viewWillTransition(to:with:)` and restore it in the coordinator's `animate(alongsideTransition:completion:)`, or when `viewDidLayoutSubviews` sees a new width. Both keep the reader's place.
- Every height saved for the old width is wrong for the new one; key or empty by width.

## Selectable text and menus

- Message text is a `UITextView` with `isEditable = false`, `isSelectable = true`, `isScrollEnabled = false`, zero `textContainerInset` and zero `lineFragmentPadding`.
- `textView(_:editMenuForTextIn:suggestedActions:)` alone works on every version: on iOS 26, when the ranges method is missing, the system calls it with the union range. `textView(_:editMenuForTextInRanges:suggestedActions:)` exists only from iOS 26. Adding it is a choice (for multi-range selections); if you implement both, return the same menu.

```swift
private func selectionMenu(_ suggested: [UIMenuElement]) -> UIMenu {
    UIMenu(children: [replyAction] + suggested)   // Reply first; the system's Copy stays in `suggested`
}
```

- The bubble's own context menu is a `UIContextMenuInteraction` whose `contextMenuInteraction(_:configurationForMenuAtLocation:)` returns nil over the text, or one attached only to the parts that are not text.

## Launch

- Start building the data off the main thread before the first view controller loads.
- Create rarely used cell subviews lazily; cache strings that repeat (a timestamp per minute); compile patterns once.
- A width guessed before layout is fine only if checked against the real one; throw the work away when it differs.

## Shortcuts that do not count

They improve a number, not the screen: redrawing streamed rows or a status label on a timer; timing part of an operation, or starting the clock inside the action; laying out the first screen from geometry saved by a previous launch; skipping work because the same action ran a moment ago.

## How to work

- Write the screen from the numbers you are given (sizes, colours, fonts) before studying screenshots pixel by pixel.
- Write checks once the behaviour works, one per behaviour. After a change, judge it only on a build you just installed.
- Measuring code must cost nothing: no shrink-to-fit labels, nothing redrawn per frame.

## Self-review checklist

Run this when the room first works and again before you stop (or call the `uikit-list-reviewer` agent, which checks the same list):

1. Edit menu returns `[yours] + suggestedActions`; no long press over text.
2. One section; no whole-list rebuild or new snapshot per update.
3. An insert or prepend measures only the new rows.
4. Removals and shrinks supply the final offset first, without animation.
5. No decoded bitmaps for placeholders; every cache capped.
6. Each text measured once, with the drawing view's settings.
7. Nothing created in configure; no layout flush in scroll, cell or invalidation callbacks; no shrink-to-fit label updated often.
8. Any timing starts at the touch's timestamp and stops at the next frame.
9. Release strip settings set.
