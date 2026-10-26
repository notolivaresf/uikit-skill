# The table view alternative

Use when you choose a `UITableView` instead of the custom collection layout in `SKILL.md`: it takes about half the list code (61 lines of adapter against 119). Everything in `SKILL.md` still applies except the layout class; this file lists what differs. Tags resolve in `../SOURCES.md`.

**Streaming cost: tied in E1, being rechecked.** In E1 the table streamed 600 messages for 7.73 s of CPU against 7.84 s for the collection view. In the later E2/E3 runs the same table adapter took 30.7 s while the collection view stayed at 8.25 s; that table number is being rerun. Until it is, do not pick the table expecting it to stream cheaper or equal. [E1] [E2]

## Shape

- A plain `UITableViewDataSource`, one section, rows are message ids, `separatorStyle = .none`. [E1 spike, E1Table/TableAdapter.swift] [F1.1]
- Saved heights only. Return the saved value from both `tableView(_:heightForRowAt:)` and `tableView(_:estimatedHeightForRowAt:)`, so the table never guesses. A self-sizing table (`automaticDimension`) lost the bottom pin after Reset and at the end of every stream, and drifted up to 17 pt after a text size change mid-list. [E1 spike] [E2] [SDK-UITableView]
- When heights come from code, set `selfSizingInvalidation = .disabled`. [WWDC22-10068] [SDK-UITableView]
- Changed messages: `reconfigureRows(at:)` for those ids, which keeps the existing cell. [SDK-UITableView] [WWDC21-10252]

## Pinning

Insert the new rows and scroll to the last one with `scrollToRow(at:at: .bottom, animated: false)` inside the same `UIView.performWithoutAnimation` as the insert. Setting `contentOffset` after `endUpdates` or `performBatchUpdates` is undone at the table's next layout pass: the pin was lost after the third message and never came back (0 of 600 followed). [E1]

## Older messages and removals

- No layout-owned anchor. Use approach (a): before the insert, record the top visible row's message id (`indexPathsForVisibleRows`) and its distance from the top of the visible area (`rectForRow(at:)`); measure only the new page and drop the old first row's saved height; insert without animation; then set the offset that puts that row back, in the same no-animation pass. 0 pt with saved heights; 1 pt with self-sizing rows. [E3] [F1.4] [SDK-UITableView]
- Not the sum of the inserted rows' heights: 42 pt drift, because the old first row changes height. The change in `contentSize.height` held at 0 pt with saved heights. [E3]
- Removals, Reset and shrinking text: the table animates its offset back into range just like the collection view. Compute the final offset first and set it in the same `performWithoutAnimation`; for many rows, `reloadData` plus that offset. [F1.4] [F2.2]

## Jump

Load the page around the target, `reloadData`, then set the offset from `rectForRow(at:)` for the target, rounded to pixels, without animation. Both the saved-height and the self-sizing table landed on message 2,500 within 0.2 pt. [F1.3] [E2] [SDK-UITableView]
