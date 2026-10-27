# Measuring text that stays on TextKit 2

Use only when a message text view must stay a default (TextKit 2) `UITextView`. Prefer the TextKit 1 view and `NSLayoutManager` measurer in `SKILL.md`: a TextKit 2 view in reused cells reruns its viewport layout, selection rects and content size on every frame or text assignment, and skipping unchanged assignments does not recover it.

- Measure with `NSAttributedString.boundingRect(with:options: [.usesLineFragmentOrigin, .usesFontLeading], context: nil)` on the exact string the view shows, at the same width, with the same zero inset and padding, rounded up to the pixel.
- What breaks a pairing is measuring with one text engine or settings and drawing with another.
- Do not draw bodies from a pre-measured layout and swap in a real text view on touch: much code, and no faster text size change.
