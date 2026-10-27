# Measuring message text

Use when writing the height measurer, when rows drift by a fraction of a point, or when a text size change is slow.

Keep this measurer even though self-sizing rows are about as fast: self-sizing rows lose the pin and drift, so saved heights stay for correctness.

## Use the TextKit 1 pairing

| Text view | Measurer |
|---|---|
| TextKit 1: `UITextView(usingTextLayoutManager: false)`, or `init(frame:textContainer:)` with a container that has its own `NSLayoutManager` and text storage | an `NSLayoutManager` with its own text storage and container, `ensureLayout(for:)` then `usedRect(for:)` |

This applies to read-only text in reused cells. A default (TextKit 2) `UITextView` there is much more expensive: every frame or text assignment in cell setup reruns its viewport layout, selection rects and content size, so streaming costs clearly more CPU and memory. Skipping unchanged text assignments does not recover it; switching the text view to TextKit 1 does. If you keep TextKit 2 anyway, measure with `NSAttributedString.boundingRect(with:options: [.usesLineFragmentOrigin, .usesFontLeading], context: nil)`. The input bar stays on TextKit 2: never read `.layoutManager` on a TextKit 2 text view, which silently switches it to TextKit 1.

What breaks a pairing is measuring with one text engine and settings and drawing with another.

## Settings that must match

- The text view draws with `textContainerInset = .zero` and `textContainer.lineFragmentPadding = 0`, so the measured size is the drawn size.
- Measure the exact attributed string the view will show: same fonts, paragraph style, links and emoji. Build it once per message and keep it with the height, then hand that same string to the cell.
- The width is the bubble's maximum inner width for the current container width, never the screen's.
- Round up to the pixel: `ceil(h * scale) / scale`, with `scale` from `traitCollection.displayScale` (not `UIScreen.main`).
- Snap row tops and bottoms to the pixel grid in the layout, rounding the running position, not each height. Offsets are rounded too, so edges off the grid leave a sliver of the neighbouring row, and an anchor read from the top row then picks that neighbour.

## Measure once

- Make fonts, paragraph styles and the measurer's text storage once per text size, not per row. A one-line header or name needs no layout manager pass: take the attributed string's own `size()`, capped to the width.

- One measurement per message per (width, text size). Measuring every text twice, once for its natural width and once wrapped, is a common and noticeable cost on load older and text size. Measure wrapped at the maximum width; the result's width is the natural width when it is narrower.
- Key the saved value by message id and width; keeping the last two widths makes rotating back, or resizing a window, free. Drop it when that message's content changes (an edit, a new reaction that changes the row's height). A revision number on the message makes that check cheap.
- Text size change: rebuild the stored strings, empty the store, then measure only what the list asks for. Measuring each text once is what makes a text size change fast.
- On insert or prepend, measure only the new rows. Re-measuring the whole loaded window to add one page makes load older the slowest operation on the screen.

## Off the main thread

- Building the first page's attributed strings and heights off the main thread, before the first view controller loads, is safe when the width is checked against the real one afterwards.
- If you measure on more than one thread, give each worker its own measurer. There is no clear gain from parallel row layout; leave it out unless your own measurement shows one.

## Custom drawing

Drawing bodies from a pre-measured layout and swapping in a real `UITextView` on touch adds much code and does not make a text size change faster. Do not do it.

## When older rows arrive

The old first row is now threaded under a new neighbour and can change height. Drop its saved height along with measuring the new page; adding only the new rows' heights to the offset drifts for missing exactly this.

## A message that grows

A reply streamed in pieces changes height many times. Re-measure it at most once per display frame, not per piece, and shift only the rows below it. Re-measuring the whole long message per piece grows with the square of its length; for very long replies keep the finished paragraphs' heights and measure only the last one.

## Image rows

The height comes from the attachment's stored pixel size scaled to the bubble's width (capped), known before the image loads. Never resize a row when its image arrives.
