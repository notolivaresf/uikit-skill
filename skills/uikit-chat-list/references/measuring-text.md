# Measuring message text

Use when writing the height measurer, when rows drift by a fraction of a point, or when a text size change is slow. Tags resolve in `../SOURCES.md`.

**PENDING E2.** If E2 finds self-sizing rows as fast as saved heights, this file shrinks to "Measuring for a jump" below. Until then it is the measurer every fast build used. [F §4]

## Pick one pairing and keep it

Two pairings were measured, and both held drift under 0.2 pt through load older, text size and rotation: [F2.4 item 5] [F §4]

| Text view | Measurer | Used by |
|---|---|---|
| TextKit 1: `UITextView(usingTextLayoutManager: false)` | an `NSLayoutManager` with its own text storage and container | every Opus build |
| Default (TextKit 2) `UITextView` | `NSAttributedString.boundingRect(with:options: [.usesLineFragmentOrigin, .usesFontLeading], context: nil)` | every Sol build; the Round 5 spikes |

What breaks the pairing is measuring with one text engine and settings and drawing with another. [F2.4 item 5] [SDK-UITextView]

## Settings that must match

- The text view draws with `textContainerInset = .zero` and `textContainer.lineFragmentPadding = 0`, so the measured size is the drawn size. [E1 spike, RowView.swift]
- Measure the exact attributed string the view will show: same fonts, paragraph style, links and emoji. Build it once per message and keep it with the height, then hand that same string to the cell. [E1 spike, RowGeometry.swift]
- The width is the bubble's maximum inner width for the current container width, never the screen's. [F1.2 O3] [F1.6]
- Round up: `ceil` the measured width and height. Round cumulative row positions to the pixel grid when you place an offset, or a sliver of the neighbouring row shows. [E1 spike, RowGeometry.swift] [F2.2 S3]

## Measure once

- One measurement per message per (width, text size). O2 measured every text twice, once for its natural width and once wrapped; that cost an estimated 15 to 40 ms on load older and text size. Measure wrapped at the maximum width; the result's width is the natural width when it is narrower. [F1.3 O2] [F2.2] [F1.5 O3]
- Key the saved value by message id, and drop it when that message's content changes (an edit, a new reaction that changes the row's height). A revision number on the message makes that check cheap. [F §3.2 outline] [F1.2 SC2/SC3]
- Width or text size change: empty the whole store, then measure only what the list asks for. Every build re-measured the whole loaded window; measuring each text once is what separated the fast ones. [F1.5]
- On insert or prepend, measure only the new rows. SC2 re-measured up to 300 rows to add 40 and had the slowest load older. [F1.4 SC2]

## Off the main thread

- Building the first page's attributed strings and heights off the main thread, before the first view controller loads, is safe when the width is checked against the real one afterwards. [F1.2 S3, OC3, O3]
- If you measure on more than one thread, give each worker its own measurer (O3). Parallel row layout moved no reported number in the one build that measured it separately, so do not add it for speed. [F1.5 O3, OC3] [F §4]

## Custom drawing

Drawing bodies from a pre-measured layout and swapping in a real `UITextView` only when touched (OC3, about 130 lines) removed most of the app's own text work and moved the measured text size time by nothing. Do not do it. [F1.5 OC3]

## Measuring for a jump

Whatever E2 decides, a jump needs the target row's frame before the list scrolls. With saved heights, `rectForRow(at:)` (table) or the layout's frame (collection view) is exact. With estimated heights, the row lands one off until the rows above it are measured. [F1.3] [SDK-UITableView]
