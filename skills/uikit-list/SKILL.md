---
name: uikit-list
description: Build or fix a long, fast-scrolling list screen in a UIKit app (iOS 18 and later, Swift or Objective-C) - feeds, inboxes, catalogs, logs, timelines, comment threads, chats. Live items arriving, older or newer pages loading without the view moving, jumping to an item, variable row heights, Dynamic Type and rotation without losing the reader's place, selectable text with your own edit menu actions. Use for UICollectionView or UITableView screens that grow at one or both ends, and when such a list stutters, jumps, drifts or loses its position.
---

# UIKit list

A list that grows at its ends and never moves under the reader's eye.

## Text and menus

These rules say how to build it; they are not a list to test. Draw icons with system symbols; don't download or import image assets or an app icon.

1. **Edit menu: your actions first, then the system's.** If rows show selectable text, return `UIMenu(children: [custom] + suggestedActions)` from `textView(_:editMenuForTextIn:suggestedActions:)`. Appended items land on the second page on iPhone; a menu without `suggestedActions` loses Copy. No long-press recognizer over the text; the row's context menu returns nil over the text. Copy is the system's item: if the menu lists it, it works; never read `UIPasteboard` from a UI test, it waits on the paste prompt.
2. **Read-only row text is TextKit 1:** `UITextView(usingTextLayoutManager: false)`; a default TextKit 2 view redoes viewport layout and selection rects on every cell setup.

## The list

3. **A `UICollectionView` with your own `UICollectionViewLayout`, one section, rows keyed by the item's stable id**, `selfSizingInvalidation = .disabled`, `isPrefetchingEnabled = false`. The layout keeps row tops from saved heights, snapped to the pixel grid by rounding the running position with `traitCollection.displayScale`. An append adds frames; a changed height shifts the tops below it; never recompute every top or empty the attribute cache for one change.
4. **Measure each row once** and keep the height by item id and width, for the current text size; self-sizing rows lose the position and drift. Measure with an `NSLayoutManager` the exact attributed string the cell shows, at the content's inner width, rounded up to the pixel; a row without text measures its fixed parts the same way. When a page arrives next to existing rows, drop the neighbouring row's height too (it gains a neighbour).
5. **Nothing per row that most rows do not show:** no formatters, regexes, images, paragraph styles or subviews created in configure or in the measurer; every cache capped; no `adjustsFontSizeToFitWidth` on a label updated often; no `draw(_:)` in cells (a drawn view keeps a bitmap its size).

## Updates

6. **Live changes:** queue each arrival and flush the queue on a 0.1 s timer; changes the user makes apply at once. The flush is a `performBatchUpdates` inside `UIView.performWithoutAnimation`, data changed in the same closure: `insertItems` for new ids, `reconfigureItems` for changed ones, never `reloadData` or a new snapshot per item. If the list is anchored to an end (the bottom of a chat or log, the top of a feed), keep "pinned" as state: set it when the reader reaches that end, clear it only when the reader drags away, and never work it out again from the offset after a batch. While pinned, the layout's target offset is that end, so a flush that also trims stays there without a second offset fix.
7. **Trim rarely, in the same batch:** let the window run a page past its limit before trimming back to it, so most arrivals are plain appends. When a batch trims, deletes use old positions and inserts new ones: trimming k rows from the front while appending inserts at `oldCount - k`. `reconfigureItems` asks for the cell at once, at the position given: reconfigure in a batch of its own before one that trims.
8. **A page loaded before the first visible row:** insert it without animation in `performBatchUpdates`, measuring only the new page; the layout's `targetContentOffset(forProposedContentOffset:)` returns the pinned end while pinned and otherwise keeps the first visible row where it was, and runs only inside batch updates, never after `reloadData` or a plain invalidation. Never add the inserted heights to the offset. The top row counts only once its visible content is below the top edge: ignore transparent margin and sub-point rounding remainders (small tolerance). Use it to capture the anchor and read the top row after restoring, so a re-layout never names the row above.
9. **Jump to an item:** replace the window with a small one around the target (about 10 rows before, 30 after; measuring is the jump's cost), `reloadData`, set the offset from the target's saved top rounded to the pixel at once, then one `layoutIfNeeded`. No animation.

## Build and launch

10. Start making the data on a background queue in the app's first line, before `UIApplicationMain`, and build the screen while it runs (no placeholder controller swapped in later); check a guessed width against the real one.
11. The app target's Release build strips: `DEPLOYMENT_POSTPROCESSING`, `STRIP_INSTALLED_PRODUCT`, `STRIP_STYLE = all`, `DEAD_CODE_STRIPPING`.

Check each behaviour once the screen works, then rerun only the failed tests by name (`-only-testing:`), not the whole test target. Pass `-test-timeouts-enabled YES -default-test-execution-time-allowance 60` to `xcodebuild test`. Close any simulator you started.
