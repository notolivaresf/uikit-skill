---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - live messages pinned to the bottom, older messages loading above without the view moving, jumping to a message, keyboard, Dynamic Type, rotation, right-to-left and VoiceOver without losing the reader's place, selectable text with your own edit menu actions, image rows, low memory. Use for UICollectionView or UITableView screens that grow at both ends (chat rooms, comment threads, logs, timelines) and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

A list that grows at both ends and never moves under the reader's eye. UIKit only.

## Core rules (read these even if nothing else)

1. **Edit menu: your actions first, then the system's.** Return `UIMenu(children: [reply] + suggestedActions)` from the text view delegate. Never drop `suggestedActions` or append yours at the end (on iPhone it lands on the second page); no long-press recognizer over selectable text.
2. **Never let UIKit choose the offset after rows are removed or shrink.** It animates the offset back into range and undoes your pin or anchor. Set the final offset yourself in the same no-animation pass. `targetContentOffset(forProposedContentOffset:)` runs only inside batch updates, not after `reloadData` or a plain invalidation.
3. **One section, stable message ids, explicit row changes.** Insert only new rows, reconfigure only changed ids; never rebuild the list or snapshot per message, never a section per message (one per day is fine).
4. **Measure each message once and keep the height by message id** and width, for the current text size. Self-sizing rows are as fast but lose the pin and drift.
5. **Nothing per row that most rows do not show:** no formatters, regexes, images, paragraph styles or subviews created in configure or in the measurer; no bitmaps for placeholders; every cache capped; no `adjustsFontSizeToFitWidth` on anything updated often.
6. **Keep each row's measure cheap; it is the cost of every jump, page and text size change.** One layout pass per text, none for a one-line header (use the string's own size), fonts and styles made once per text size, and only as many rows as the window needs.
7. **Jump by geometry:** replace the window with a small one around the target (about 10 rows above, 30 below; measuring is the jump's main cost), `reloadData`, set the offset from its saved-height top rounded to the pixel, then one `layoutIfNeeded`. No animation; keep image caches.
8. **Apply live changes on the next main-queue turn, no timer.** Queue each arrival and schedule one flush if none is pending; it applies everything queued as one batch. A fixed timer delays every message by most of its interval; a display-link flush groups no more at normal rates. Your own sends apply at once.
9. **Each update costs only its rows:** append frames for new rows, shift the tops below a changed height, keep every other cached frame. Never recompute all tops or empty the attribute cache for an append.
10. **The app target's Release build strips:** `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`.
11. **Never force layout from inside the list's own passes.** No `layoutIfNeeded` (or anything that reads frames through it) from `scrollViewDidScroll`, cell provisioning or an invalidation callback: it crashes the collection view. Read the layout's cached geometry.
12. **Batch indexes:** in one `performBatchUpdates`, deletes use old positions, inserts new ones. Trimming k rows from the top while appending inserts at `oldCount - k`, not `oldCount`. Do both in that one batch: `reloadData` plus a second batch reconfigures every visible cell.

## The list: a collection view with a custom layout

A `UICollectionView` with your own `UICollectionViewLayout` subclass, one section, rows keyed by message id:

- Row tops sorted by y from saved heights, snapped to the pixel grid (`traitCollection.displayScale`) by rounding the running position: off-grid edges leave a sliver of the row above for the next anchor to pick.
- `prepare()` works once per invalidation; `layoutAttributesForElements(in:)` binary-searches the tops.
- `shouldInvalidateLayout(forBoundsChange:)` is true only when the width changes.
- `flipsHorizontallyInOppositeLayoutDirection` returns true (right-to-left mirrors).
- Read anchors from fresh frames, not attributes cached before the update.

On the collection view: `selfSizingInvalidation = .disabled`. Prefetching makes little difference; turning it off also disables `prefetchDataSource`, so keep it if you prefetch images.

**A table view** is half the code but costlier under accessibility queries; see `references/table-view.md`.

## Heights

- Empty the store on a width or text size change; drop one entry when a message changes, and the old first row's when older rows arrive (it gains a new neighbour).
- One measurer, fonts and styles made once, the drawing view's settings; hand the measured string to the cell. See `references/measuring-text.md`.
- A reply that grows while it streams changes only its own height: reconfigure it and shift the rows below; keep the pin.
- Image rows take their height from the attachment's stored size, never the loaded image. Decode thumbnails off the main thread at display size (`preparingThumbnail(of:)`) into a capped cache.

## Live updates and pinning

- Read "pinned" before the update, apply the change, restore the pin.
- `insertItems(at:)` in `performBatchUpdates` in `UIView.performWithoutAnimation`, data changed in the same closure; then set the bottom offset.
- Changed messages (reactions, edits) are `reconfigureItems(at:)`. Deletes may animate when the reader is not pinned.
- Diffable: edit the current snapshot; `applySnapshotUsingReloadData` only for a full replace such as a jump.
- Keep the window to a few hundred rows; trim in the append's batch (rule 12).

## Keyboard and input bar

- Pin the input bar to `view.keyboardLayoutGuide`; the list's bottom inset is the bar's height plus the keyboard overlap; re-pin when it changes while pinned.
- `keyboardDismissMode = .interactive`; during an interactive dismiss update the inset only, never re-pin per frame.

## Older messages above

Let the layout keep the anchor:

1. Note the first visible row's id and its distance from the visible top.
2. Measure only the new page, drop the old first row's height, insert without animation in `performBatchUpdates`.
3. `targetContentOffset(forProposedContentOffset:)` returns that row's new top minus the distance, once.

Fallback (the table view's): set the offset that puts the anchor row back in the same pass. Never add the inserted rows' heights to the offset: the old first row's change drifts it.

## Text size, rotation, right-to-left

- Register for `UITraitPreferredContentSizeCategory`, `UITraitLegibilityWeight` and `UITraitDisplayScale` (and width changes). Cell labels use `adjustsFontForContentSizeCategory = false`: fonts come from the measurer's strings so drawn and measured text agree.
- On a change: capture the anchor, rebuild the strings, empty the heights, invalidate, `layoutIfNeeded`, set the anchored offset yourself (rule 2). The cost is the measuring, so keep the window small.
- Rotation: capture the anchor in `viewWillTransition(to:with:)`, restore it alongside the transition.
- Right-to-left: `.natural` alignment, leading and trailing constraints.

## Selectable text and menus

- Read-only message text is a TextKit 1 `UITextView(usingTextLayoutManager: false)` measured with an `NSLayoutManager`: not editable, selectable, no scrolling, zero `textContainerInset` and `lineFragmentPadding`. TextKit 2 redoes viewport layout on every cell setup. The input bar keeps TextKit 2; never read its `.layoutManager` (that switches it to TextKit 1).
- `textView(_:editMenuForTextIn:suggestedActions:)` alone works everywhere; if you add the iOS 26 ranges method, return the same menu.

```swift
private func selectionMenu(_ suggested: [UIMenuElement]) -> UIMenu {
    UIMenu(children: [replyAction] + suggested)   // Reply first; the system's Copy stays in `suggested`
}
```

- The bubble's context menu returns nil over the text, or is attached only to the parts that are not text.

## VoiceOver

- One element per row (sender, text, time); reply and react as `accessibilityCustomActions`.
- Announce arrivals (`.announcement`) only while pinned, grouped; never move focus on a live insert. After a jump, post `.layoutChanged` with the target row.

## Launch

- Build the data off the main thread before the first view controller loads; check a guessed width against the real one.
- Cache strings that repeat (a timestamp per minute); compile patterns once; make rarely used cell parts lazily.

## How to work

- If you time anything, start at the event's `timestamp` and stop at the next display-link frame; timing code must cost nothing. A change that improves a timing but not the screen (holding rows back, reusing a previous launch's geometry) is a fault.
- Judge a change only on a build you just installed. Close any simulator you started when you stop.

## Self-review checklist

Run once before you stop, or call the `uikit-list-reviewer` agent:

1. Edit menu returns `[yours] + suggestedActions`; no long press over text.
2. One section; live changes applied on the next turn, no timer; no whole-list rebuild.
3. Inserts, prepends and text size changes measure only the rows needed; trims and appends share one batch with correct indexes.
4. Removals, shrinks and text size changes set the final offset without animation; row edges on the pixel grid.
5. Jumps load a small window; image heights from metadata, thumbnails at display size; every cache capped.
6. Each text measured once, with the drawing view's settings; TextKit 1 for read-only message text.
7. Nothing created in configure; no layout flush in scroll, cell or invalidation callbacks; no shrink-to-fit label updated often.
8. Keyboard inset re-pins; right-to-left mirrors; VoiceOver actions, pinned-only announcements.
9. Release strip settings set.
