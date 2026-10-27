# More surfaces of a chat list

Read the part your screen has. Everything in `SKILL.md` still applies.

## Keyboard and input bar

- Pin the input bar to `view.keyboardLayoutGuide`. The list's bottom inset is the bar's height plus the keyboard overlap; when it changes while the reader is pinned, re-pin.
- `keyboardDismissMode = .interactive`; during an interactive dismiss update the inset only, never re-pin per frame.
- The input bar's text view stays on TextKit 2; never read its `.layoutManager`.

## Text size and rotation

- Register for `UITraitPreferredContentSizeCategory`; cell labels use `adjustsFontForContentSizeCategory = false`, since fonts come from the measured strings. On a change capture the anchor row, rebuild the strings, empty the heights, invalidate, `layoutIfNeeded` once, set the anchored offset.
- Rotation: capture the anchor in `viewWillTransition(to:with:)` and restore it alongside the transition.

## VoiceOver

- One element per row (sender, text, time); reply and react as `accessibilityCustomActions`.
- Announce arrivals (`.announcement`) only while pinned, grouped; never move focus on a live insert. After a jump, post `.layoutChanged` with the target row.

## Right-to-left

- The layout's `flipsHorizontallyInOppositeLayoutDirection` returns true.
- `.natural` text alignment, leading and trailing constraints; register for `UITraitLegibilityWeight` and `UITraitDisplayScale` with the text size trait.

## Image rows

- Decode thumbnails off the main thread at display size (`preparingThumbnail(of:)`) into a capped cache; never resize a row when its image arrives.
- Collection view prefetching makes little difference to a list pinned at the bottom; turning it off also disables `prefetchDataSource`, so keep it if you prefetch images.

## A reply that grows while it streams

- It changes only its own height: reconfigure it, shift the rows below, keep the pin.
- Re-measure it at most once per display frame, not per piece; for a very long reply keep the finished paragraphs' heights and measure only the last one.

## Measuring on more than one thread

- Give each worker its own measurer. Parallel row layout shows no clear gain; leave it out unless your own measurement shows one.
- Keeping the last two widths' heights makes rotating back free.
