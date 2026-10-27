---
name: uikit-chat-list
description: Build or fix a long chat or feed list in a UIKit app (iOS 18 and later, Swift or Objective-C) - live messages pinned to the bottom, older messages loading above without the view moving, jumping to a message, Dynamic Type and rotation without losing the reader's place, selectable text with your own edit menu actions. Use for UICollectionView or UITableView screens that grow at both ends (chat rooms, comment threads, logs, timelines) and when such a list stutters, jumps, drifts or loses its pin.
---

# UIKit chat list

A list that grows at both ends and never moves under the reader's eye.

## Text and menus

These rules say how to build it; they are not a list to test.

1. **Edit menu: your actions first, then the system's.** From `textView(_:editMenuForTextIn:suggestedActions:)` return `UIMenu(children: [reply] + suggestedActions)`. Appended items land on the second page on iPhone; a menu without `suggestedActions` loses Copy. No long-press recognizer over the text; the bubble's context menu returns nil over the text. Copy is the system's item: if the menu lists it, it works; never read `UIPasteboard` from a UI test, it waits on the paste prompt.
2. **Read-only message text is TextKit 1:** `UITextView(usingTextLayoutManager: false)`; a default TextKit 2 view redoes viewport layout and selection rects on every cell setup.

## The list

3. **A `UICollectionView` with your own `UICollectionViewLayout`, one section, rows keyed by message id**, `selfSizingInvalidation = .disabled`, `isPrefetchingEnabled = false`. The layout keeps row tops from saved heights, snapped to the pixel grid by rounding the running position with `traitCollection.displayScale`. An append adds frames; a changed height shifts the tops below it; never recompute every top or empty the attribute cache for one change.
4. **Measure each message once** and keep the height by message id and width, for the current text size; self-sizing rows lose the pin and drift. Measure with an `NSLayoutManager` the exact attributed string the cell shows, at the bubble's inner width, rounded up to the pixel. When older rows arrive, drop the old first row's height too (it gains a neighbour).
5. **Nothing per row that most rows do not show:** no formatters, regexes, images, paragraph styles or subviews created in configure or in the measurer; every cache capped; no `adjustsFontSizeToFitWidth` on a label updated often; no `draw(_:)` in cells (a drawn view keeps a bitmap its size).

## Updates

6. **Live changes:** queue each arrival and flush the queue on a 0.1 s timer; your own sends apply at once. The flush is a `performBatchUpdates` inside `UIView.performWithoutAnimation`, data changed in the same closure: `insertItems` for new ids, `reconfigureItems` for changed ones, never `reloadData` or a new snapshot per message. Read "pinned" before it and set the bottom offset after it.
7. **Trim in the same batch:** deletes use old positions, inserts new ones: trimming k rows from the top while appending inserts at `oldCount - k`. `reconfigureItems` asks for the cell at once, at the position given: reconfigure in a batch of its own before one that trims.
8. **Older messages above:** insert them without animation in `performBatchUpdates`, measuring only the new page; the layout's `targetContentOffset(forProposedContentOffset:)` keeps the first visible row where it was, and runs only inside batch updates, never after `reloadData` or a plain invalidation. Never add the inserted heights to the offset. The top row counts only once its visible content is below the top edge: ignore transparent margin and sub-point rounding remainders (small tolerance). Use it to capture the anchor and read the top row after restoring, so a re-layout never names the row above.
9. **Jump:** replace the window with a small one around the target (about 10 rows above, 30 below; measuring is the jump's cost), `reloadData`, set the offset from the target's saved top rounded to the pixel at once, then one `layoutIfNeeded`. No animation.

## Build and launch

10. Start making the data on a background queue in the app's first line, before `UIApplicationMain`, and build the screen while it runs (no placeholder controller swapped in later); check a guessed width against the real one.
11. The app target's Release build strips: `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`.

Check each behaviour once the screen works, then rerun only the failed tests by name (`-only-testing:`), not the whole test target. Pass `-test-timeouts-enabled YES -default-test-execution-time-allowance 60` to `xcodebuild test`. Close any simulator you started.
