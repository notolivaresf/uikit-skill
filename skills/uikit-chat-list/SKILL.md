---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - messages streaming in while pinned to the bottom, older messages loading above without the view moving, jumping to a message, Dynamic Type and rotation without losing the reader's place, selectable message text with the system edit menu plus your own actions, and fast launch with low memory. Use for any UITableView or UICollectionView screen that grows at both ends, for chat rooms, comment threads, logs and timelines, and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

The recipe for one kind of screen: a long list that grows at the bottom while the reader is pinned there, grows at the top when older items load, and must never move under the reader's eye. UIKit only. Each rule ends with its evidence tag; `SOURCES.md` in this folder resolves the tags. Lines marked **PENDING** are open questions still being measured; follow the interim rule they give.

## Core rules (read these even if nothing else)

1. **Edit menu: your actions first, then the system's.** Return `UIMenu(children: [reply] + suggestedActions)` from the text view delegate. Never build a menu without `suggestedActions`, never append your action at the end (it lands on the menu's second page), and never put a long-press recognizer over selectable text. [F1.7] [F2.2] [SDK-UITextView]
2. **Never let UIKit choose the offset after rows are removed or shrink.** It animates the offset back into range and undoes your pin or anchor. Set the final offset yourself, in the same no-animation pass as the change. [F1.4] [F2.2] [E1]
3. **One section, stable message ids, explicit row changes.** Insert only the new rows, reconfigure only the changed ids, never rebuild the whole list or snapshot per message, never one section per message. [F1.1] [SDK-UICollectionView] [WWDC21-10252]
4. **Measure each message once and keep the height by message id,** for the current width and text size. Never measure the same text twice. **PENDING E2** (see Heights). [F1.5] [F2.2]
5. **Nothing per row that most rows do not show:** no formatters, regexes, images or extra subviews created in configure; no decoded bitmaps for placeholders; every cache capped. [F1.2] [F1.8] [WWDC21-10252]
6. **No shrink-to-fit text in anything updated often** (`adjustsFontSizeToFitWidth` on a label redrawn per message cost four times the whole list). Pick the font once. [E1] [F1.1]
7. **Jump by geometry, not by hope:** load a page around the target, then place it from the list's own frame for that row, without animation. [F1.3]
8. **Apply every message as it arrives.** Do not hold updates on a timer: one update per message cost the same as 50 to 80 ms batches on an iPhone 12. [F1.1] [F1.10]
9. **Time from the touch to the frame,** if you time anything: start at the event's `timestamp`, stop at the next display-link frame. [Findings intro] [TT-10855]
10. **Release builds strip:** `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`. [F1.9]

## The list

Use a `UITableView` with a plain data source, or a `UICollectionView` with a custom layout: for streaming text they cost the same (7.73 s against 7.84 s of CPU for 600 streamed messages, 1.3 % apart, all rows pinned). Prefer the table view: it took about half the list code (61 lines against 119). Reach for the collection view only when the screen needs something a table cannot do; then follow `references/collection-view.md`. [E1]

- One section. Rows are message ids; the model is looked up by id, never captured by index path in a closure that runs later. [F1.1]
- Not a hand-rolled `UIScrollView` recycler: its one measured gain was mixed with a bent stream and other changes, and it loses cell reuse and accessibility containers. [F1.1] [F §4]
- A compositional layout with self-sizing cells and one section per message was the slowest UIKit shape measured; both builds that used it were rewritten for their largest gains. [F1.11]

## Heights

**PENDING E2: saved heights against self-sizing rows.** Interim rule, from every fast build so far: measure in code and save.

- Work out each row's height from the message, the width and the text size, and keep it in a dictionary by message id. Empty it when the width or text size changes; drop one entry when one message changes. [E1 spike, Heights.swift] [F1.5]
- Return the saved value from both `tableView(_:heightForRowAt:)` and `tableView(_:estimatedHeightForRowAt:)`, so the table never guesses. [E1 spike] [SDK-UITableView]
- Measure with the same text settings the view draws with. See `references/measuring-text.md`. [F2.4 item 5]
- Self-sizing invalidation (iOS 16) resizes a visible cell whenever its content changes, with animation unless wrapped in `performWithoutAnimation`. When heights come from code, set `selfSizingInvalidation = .disabled` so nothing resizes behind your back. [WWDC22-10068] [SDK-UICollectionView] [F1.1 S3]
- Placeholders for images not yet loaded are a background colour on a view of the final size, never a rendered or decoded bitmap. [F1.8]

_E2 decides: keep this section as is, or replace it with "let the system size rows" plus what self-sizing needs to land a jump on the right row._

## Live updates and pinning

- Read "pinned" before the update (the last row is within a small distance of the bottom), apply the change, then restore the pin. [F1.1] [E1 spike, Pinning.swift]
- Table view: insert the new rows and scroll to the last one with `scrollToRow(at:at: .bottom, animated: false)` inside the same `UIView.performWithoutAnimation`. Setting `contentOffset` after `endUpdates` or `performBatchUpdates` is undone at the table's next layout pass: the pin was lost after the third message. [E1]
- Collection view with a custom layout: a plain `setContentOffset` after the insert held the pin. [E1]
- Measure only the new rows. Changed messages (reactions, edits) are `reconfigureRows(at:)` / `reconfigureItems(at:)` for those ids, which keeps the existing cell. [F1.1] [SDK-UITableView] [WWDC21-10252]
- With a diffable data source, edit the current snapshot (append, `reconfigureItems`) rather than building a new one per message. [F1.1 S2, SC2] [SDK-UIDiffableDataSource]

## Older messages above

**PENDING E3: which anchor keeps the place.** Interim rule, the one every passing build used, drift at most 0.3 pt:

1. Before the insert, record the top visible row's message id and its distance from the top of the visible area (`indexPathsForVisibleRows`, `rectForRow(at:)`). [F1.4] [SDK-UITableView]
2. Measure only the new page, insert it without animation. [F1.4 SC2, SC3]
3. Find that id's new row frame and set the offset so the distance is the same, in the same no-animation pass, before the list lays out cells for the wrong place. [F1.4 O3] [E1]

_E3 decides between (a) this, (b) adding the inserted rows' total height to the offset, (c) a layout that keeps its own anchor (collection view only). Fill in the winner and name the ones that drifted._

**Removals and shrinking content** (trimming old rows, Reset, a smaller text size): UIKit animates the offset back into range and the anchor or pin is gone, often intermittently. Compute the final offset first and set it without animation; for a removal of many rows, `reloadData` plus that offset rather than an animated delete. In a collection view, return it from `targetContentOffset(forProposedContentOffset:)`. This one fault cost 7 to 20 minutes of debugging in each of three builds. [F1.4] [F2.2] [SDK-UICollectionViewLayout]

Keep the loaded window bounded (a few hundred rows) and trim by these rules; the leanest build trimmed to 300. [F1.8 SC2]

## Jump to a message

- Replace the loaded window with one page around the target (about 10 rows above, 30 below), `reloadData`, then place the target from the list's own geometry: `rectForRow(at:)` for a table, the layout's frame for a collection view. No animation. [F1.3 O1, O3, OC3]
- Landing one row off ("2499 not 2500") came from scrolling before heights were known. With saved heights the frame is exact; **PENDING E2** for self-sizing. [F1.3] [F2.2]
- Round the target offset to the screen's pixel grid, or a sliver of the row above shows. [F2.2 S3 NOTES]

## Text size and rotation

- On a text size change (register for `UITraitPreferredContentSizeCategory`) or a new width: capture the anchor, empty the saved heights, re-measure, `reloadData`, restore the anchor without animation. Measuring each text once is what separated the fast builds. [F1.5] [SDK-UITraitCollection]
- Rotation: capture the anchor in `viewWillTransition(to:with:)` and restore it in the coordinator's `animate(alongsideTransition:completion:)`, or restore it when `viewDidLayoutSubviews` sees a new width. Both held drift under 0.2 pt. [F1.6] [SDK-UIViewController]
- Every height saved for the old width is wrong for the new one; key or empty by width. [F1.2 O3]

## Selectable text and menus

- Message text is a `UITextView` with `isEditable = false`, `isSelectable = true`, `isScrollEnabled = false`, zero `textContainerInset` and zero `lineFragmentPadding`. [E1 spike, RowView.swift] [F1.7]
- Implement both delegate methods: `textView(_:editMenuForTextIn:suggestedActions:)` (iOS 16 to 25) and `textView(_:editMenuForTextInRanges:suggestedActions:)` (iOS 26; when it exists the system calls it instead). Both return the same menu: [SDK-UITextView] [F1.7]

```swift
private func selectionMenu(_ suggested: [UIMenuElement]) -> UIMenu {
    UIMenu(children: [replyAction] + suggested)   // Reply first; the system's Copy stays in `suggested`
}
```

- The bubble's own context menu is a `UIContextMenuInteraction` whose `contextMenuInteraction(_:configurationForMenuAtLocation:)` returns nil over the text, or one attached only to the parts that are not text (name line, image, file). [F1.7 O2, S2, S3] [SDK-UIContextMenuInteraction]

## Launch

- Start building the data before the first view controller loads (in `main` or `application(_:didFinishLaunchingWithOptions:)`, off the main thread). [F1.2 S3, OC2]
- Create rarely used cell subviews lazily, on first use. [F1.2 OC3, O2]
- Cache strings that repeat (a timestamp per minute); compile patterns once. [F1.2 S3]
- A width guessed before layout is fine only if checked against the real one; throw the work away when it differs. [F1.2 O3]

## Shortcuts that do not count

These make a number look better without making the screen faster. Do not use them: [F1.10]

- Redrawing streamed rows on a timer instead of as they arrive; throttling a status label.
- Timing part of an operation, or starting the clock inside the action instead of at the touch.
- Laying out the first screen from geometry saved by a previous launch.
- Skipping work because the same action ran a moment ago.

## How to work

- Write the screen from the numbers you are given (sizes, colours, fonts) before studying screenshots pixel by pixel. [F2.3]
- Write checks once the behaviour works, one per behaviour; a geometry assertion of your own is not the requirement. [F2.3]
- After a change, judge it only on a build you just installed. [F2.3]
- Measuring code must cost nothing: no shrink-to-fit labels, nothing redrawn per frame. [E1]

## Self-review checklist

Run this when the room first works and again before you stop (or call the `uikit-list-reviewer` agent, which checks the same list):

1. Edit menu returns `[yours] + suggestedActions` from both delegate methods; no long press over text.
2. One section; no whole-list rebuild or new snapshot per update.
3. An insert or prepend measures only the new rows.
4. Removals and shrinks set the final offset first, without animation.
5. No decoded bitmaps for placeholders; every cache capped.
6. Each text measured once, with the drawing view's settings.
7. Nothing created in configure; no shrink-to-fit label updated often.
8. Any timing starts at the touch's timestamp and stops at the next frame.
9. Release strip settings set.
