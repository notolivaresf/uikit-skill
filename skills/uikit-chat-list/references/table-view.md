# The table view alternative

Use only when the list is a `UITableView` instead of the custom collection layout in `SKILL.md`. Everything in `SKILL.md` applies except the layout class; this is what differs. A table streams for about the same cost on the app's own work but costs much more under constant accessibility queries; pick it for less code, not for cost.

- A plain data source, one section, rows are message ids, `separatorStyle = .none`, `selfSizingInvalidation = .disabled`.
- Saved heights only: return the saved value from both `tableView(_:heightForRowAt:)` and `tableView(_:estimatedHeightForRowAt:)`, so the table never guesses. `automaticDimension` loses the bottom pin and drifts after a text size change.
- Changed messages: `reconfigureRows(at:)`.
- Pinning: insert, then `scrollToRow(at:at: .bottom, animated: false)` inside the same `UIView.performWithoutAnimation`. A `contentOffset` set after the updates can be undone at the table's next layout pass.
- Older messages: no layout-owned anchor. Record the top visible row's id (`indexPathsForVisibleRows`) and its distance from the visible top (`rectForRow(at:)`), insert without animation, then set the offset that puts that row back, in the same pass.
- Removals and shrinking text: compute the final offset first and set it in the same `performWithoutAnimation`; for many rows, `reloadData` plus that offset.
- Jump: load the page around the target, `reloadData`, set the offset from `rectForRow(at:)` rounded to the pixel, without animation.
