# The table view alternative

Use when you choose a `UITableView` instead of the custom collection layout in `SKILL.md`: it takes about half the list code. Everything in `SKILL.md` still applies except the layout class; this file lists what differs.

**Streaming cost.** On the app's own work the table streams for about the same CPU as the custom-layout collection view. Under constant accessibility queries (assistive technology or automation polling the screen) the table costs several times more, while the collection view does not. That is one more reason the collection view is the default; pick the table for less code, not for cost. When you compare the two, measure with nothing polling the app.

## Shape

- A plain `UITableViewDataSource`, one section, rows are message ids, `separatorStyle = .none`.
- Saved heights only. Return the saved value from both `tableView(_:heightForRowAt:)` and `tableView(_:estimatedHeightForRowAt:)`, so the table never guesses. A self-sizing table (`automaticDimension`) loses the bottom pin after a full reload and at the end of a stream, and drifts by many points after a text size change mid-list.
- When heights come from code, set `selfSizingInvalidation = .disabled`.
- Changed messages: `reconfigureRows(at:)` for those ids, which keeps the existing cell.

## Pinning

Insert the new rows and scroll to the last one with `scrollToRow(at:at: .bottom, animated: false)` inside the same `UIView.performWithoutAnimation` as the insert. Setting `contentOffset` after `endUpdates` or `performBatchUpdates` has been seen undone at the table's next layout pass (the pin lost within a few messages); the scroll call inside the same pass holds.

## Older messages and removals

- No layout-owned anchor. Use approach (a): before the insert, record the top visible row's message id (`indexPathsForVisibleRows`) and its distance from the top of the visible area (`rectForRow(at:)`); measure only the new page and drop the old first row's saved height; insert without animation; then set the offset that puts that row back, in the same no-animation pass. No drift with saved heights; about a point with self-sizing rows.
- Not the sum of the inserted rows' heights: it drifts, because the old first row changes height. The change in `contentSize.height` holds exactly with saved heights.
- Removals, full reloads and shrinking text: the table animates its offset back into range just like the collection view. Compute the final offset first and set it in the same `performWithoutAnimation`; for many rows, `reloadData` plus that offset.

## Jump

Load the page around the target, `reloadData`, then set the offset from `rectForRow(at:)` for the target, rounded to pixels, without animation. Saved-height and self-sizing tables both land on the target this way.
