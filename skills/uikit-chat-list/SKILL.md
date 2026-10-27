---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - live messages pinned to the bottom, older messages loading above without the view moving, jumping to a message, Dynamic Type and rotation without losing the reader's place, selectable text with your own edit menu actions. Use for UICollectionView or UITableView screens that grow at both ends (chat rooms, comment threads, logs, timelines) and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

A list that grows at both ends and never moves under the reader's eye.

## Text and menus

1. **Edit menu: your actions first, then the system's.** From `textView(_:editMenuForTextIn:suggestedActions:)` return `UIMenu(children: [reply] + suggestedActions)`. Appended items land on the second page on iPhone; a menu without `suggestedActions` loses Copy. No long-press recognizer over the text; the bubble's context menu returns nil over the text.
2. **Read-only message text is TextKit 1:** `UITextView(usingTextLayoutManager: false)`, not editable, selectable, no scrolling, `textContainerInset = .zero`, `lineFragmentPadding = 0`; a default TextKit 2 view redoes viewport layout and selection rects on every cell setup. A message view that must stay TextKit 2: `references/measuring-text.md`.

## The list

3. **A `UICollectionView` with your own `UICollectionViewLayout`, one section, rows keyed by message id**, `selfSizingInvalidation = .disabled`. The layout keeps row tops from saved heights, snapped to the pixel grid by rounding the running position with `traitCollection.displayScale`. `layoutAttributesForElements(in:)` binary-searches the tops; `shouldInvalidateLayout(forBoundsChange:)` is true only when the width changes. An append adds frames; a changed height shifts the tops below it; never recompute every top or empty the attribute cache for one change. For a `UITableView` instead, read `references/table-view.md`.
4. **Measure each message once** and keep the height by message id and width, for the current text size; self-sizing rows lose the pin and drift. Measure with an `NSLayoutManager` the exact attributed string the cell shows, at the bubble's inner width, with zero inset and padding, rounded up to the pixel; hand that string to the cell. One layout pass per text: wrap at the maximum width (the used width is the natural width); a one-line header takes the string's own `size()`. Fonts and paragraph styles are made once per text size. When older rows arrive, drop the old first row's height too (it gains a neighbour).
5. **Nothing per row that most rows do not show:** no formatters, regexes, images, paragraph styles or subviews created in configure or in the measurer; plain colour, not bitmaps, for placeholders; every cache capped; no `adjustsFontSizeToFitWidth` on a label updated often; no `draw(_:)` in cells (a drawn view keeps a bitmap its size).

## Updates

6. **Live changes:** queue each arrival and flush the queue on a 0.1 s timer (one batch per tenth of a second is much cheaper than one per message); your own sends apply at once. The flush is a `performBatchUpdates` inside `UIView.performWithoutAnimation`, data changed in the same closure: `insertItems` for new ids, `reconfigureItems` for changed ones, never `reloadData` or a new snapshot per message. Read "pinned" before it and set the bottom offset after it.
7. **Trim in the same batch:** keep a few hundred rows. Deletes use old positions, inserts new ones: trimming k rows from the top while appending inserts at `oldCount - k`. `reconfigureItems` asks for the cell at once, at the position given: reconfigure in a batch of its own before one that trims.
8. **You set the offset, never UIKit.** After rows are removed or shrink, or the text size changes (rebuild the strings, empty the heights), compute the final offset and set it in the same no-animation pass, or UIKit animates it back into range and undoes the pin or anchor. `targetContentOffset(forProposedContentOffset:)` runs only inside batch updates, never after `reloadData` or a plain invalidation.
9. **Older messages above:** insert them without animation in `performBatchUpdates`, measuring only the new page; the layout's `targetContentOffset(forProposedContentOffset:)` keeps the first visible row where it was. Never add the inserted heights to the offset.
10. **Jump:** replace the window with a small one around the target (about 10 rows above, 30 below; measuring is the jump's cost), `reloadData`, set the offset from the target's saved top rounded to the pixel at once, then one `layoutIfNeeded`. No animation.
11. **Never force layout from inside the list's own passes:** no `layoutIfNeeded` (or a frame read through it) from `scrollViewDidScroll`, cell provisioning or an invalidation callback; read the layout's cached geometry.

## Build and launch

12. Start making the data on a background queue in the app's first line, before `UIApplicationMain`, and build the screen while it runs (no placeholder controller swapped in later); check a guessed width against the real one.
13. The app target's Release build strips: `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`.

Close any simulator you started when you stop.

Keyboard and input bar, rotation, VoiceOver, right-to-left, image decoding and replies that grow while streaming: `references/more-surfaces.md`, only when the screen has them.
