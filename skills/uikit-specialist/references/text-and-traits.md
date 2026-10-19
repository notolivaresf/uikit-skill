# Text, Dynamic Type and traits

## Dynamic Type

Every piece of text should grow and shrink with the user's text size setting, including the accessibility sizes.

```swift
let title = UILabel()
title.font = .preferredFont(forTextStyle: .headline)
title.adjustsFontForContentSizeCategory = true
title.numberOfLines = 0
```

- Use text styles (`.largeTitle`, `.title1` to `.title3`, `.headline`, `.body`, `.callout`, `.subheadline`, `.footnote`, `.caption1`, `.caption2`) instead of point sizes.
- `adjustsFontForContentSizeCategory = true` updates the font live when the setting changes. It works for `UILabel`, `UITextField`, `UITextView` and button titles that use text-style fonts.
- Custom fonts: scale them with `UIFontMetrics`.

```swift
let base = UIFont(name: "Avenir-Medium", size: 17) ?? .systemFont(ofSize: 17)
label.font = UIFontMetrics(forTextStyle: .body).scaledFont(for: base)
label.adjustsFontForContentSizeCategory = true
```

- Scale spacing and fixed sizes that sit next to text with `UIFontMetrics(forTextStyle:).scaledValue(for:)` so icons and padding grow with it. SF Symbols in image views scale with `preferredSymbolConfiguration = UIImage.SymbolConfiguration(textStyle: .body)`.
- Weights and traits: `UIFont.preferredFont(forTextStyle:).fontDescriptor.withSymbolicTraits(.traitBold)`, then `UIFont(descriptor:size: 0)` to keep the scaled size.
- Attributed strings: set the font attribute to a text-style font. Live updates of attributed text are not automatic for every attribute; rebuild attributed strings when the content size category changes (register for `UITraitPreferredContentSizeCategory`, below).
- Design for the largest sizes. At accessibility categories (`traitCollection.preferredContentSizeCategory.isAccessibilityCategory`), switch horizontal rows to vertical stacks, allow labels to wrap, and let cells grow. Never clip text with fixed heights.
- `UILabel.showsExpansionTextWhenTruncated` (iOS 14+) shows the full text in the large content viewer for truncated labels. Bars and tab items get the large content viewer automatically; for custom controls adopt `UILargeContentViewerInteraction`.
- Test with the Accessibility Inspector or the Environment Overrides panel in Xcode at "AX5", and in the simulator's Settings.

## Responding to trait changes

On iOS 17 and later, register for the traits you care about. `traitCollectionDidChange(_:)` is deprecated.

```swift
registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) { (self: Self, previous: UITraitCollection) in
    self.rebuildAttributedText()
    self.updateAxisForTextSize()
}
```

- The handler runs only for the listed traits. Group traits by what they affect; `UITraitCollection.systemTraitsAffectingColorAppearance` and `systemTraitsAffectingImageLookup` are ready-made lists.
- The registration lasts as long as the object, and the closure receives `self` as a parameter, so you do not need `[weak self]` in this form.
- On iOS 18, automatic trait tracking removes most registrations: if `layoutSubviews()`, `updateConstraints()`, `draw(_:)`, `viewWillLayoutSubviews()`, `updateConfiguration(using:)`, a `configurationUpdateHandler`, or a compositional layout section provider reads a trait from `traitCollection`, UIKit records the dependency and re-runs that method when the trait changes. Read traits inside those methods instead of caching them.
- Custom traits: define a `UITraitDefinition`, read and write it through `traitOverrides` on a view or view controller. Use this instead of passing theme or density flags down the hierarchy by hand.
- `traitOverrides` replaces `setOverrideTraitCollection(_:forChild:)`. Override `userInterfaceStyle` or `preferredContentSizeCategory` for a subtree in previews or for a specific screen.

## Colors, appearance and contrast

- Use semantic system colors (`.label`, `.secondaryLabel`, `.systemBackground`, `.secondarySystemGroupedBackground`, `.separator`, `.tintColor`) or asset catalog colors with dark and high-contrast variants. They resolve per trait automatically.
- `CGColor` does not update with traits. When you set `layer.borderColor` or `layer.shadowColor`, resolve the color with `color.resolvedColor(with: traitCollection).cgColor` and do it inside a method that is trait-tracked (or in a registration for `systemTraitsAffectingColorAppearance`).
- Dynamic colors in code: `UIColor { traits in traits.userInterfaceStyle == .dark ? darkValue : lightValue }`.
- Respect `UIAccessibility.isReduceMotionEnabled` (skip or simplify movement), `isReduceTransparencyEnabled` and `isDarkerSystemColorsEnabled`; their change notifications are `UIAccessibility.reduceMotionStatusDidChangeNotification` and friends.

## Accessibility basics

- Image-only buttons need `accessibilityLabel`. Prefer `UIButton.Configuration` with an image and a title, and let the title be the label.
- Group a cell's content into one element when it reads as one thing: set `isAccessibilityElement = true` on the cell or content view and compose `accessibilityLabel` and `accessibilityValue`. List content configurations do this for you.
- Expose secondary actions as `accessibilityCustomActions` (reply, delete, react) so VoiceOver users do not need swipe or long press.
- Mark headers with `.header` trait so the rotor can jump between them; selected states with `.selected`.
- Post `UIAccessibility.post(notification: .announcement, argument:)` sparingly for important changes that happen off screen, and `.layoutChanged` when a large part of the screen changes.
- Make tap targets at least 44 by 44 points; extend small controls with insets or `point(inside:with:)`.
- Keep VoiceOver order matching visual order; set `accessibilityElements` on a container when the default order is wrong.

## Localization

- Use `String(localized:)` and string catalogs. Never build sentences by concatenation; use format strings with arguments so translators can reorder.
- Let labels wrap or shrink rather than fixing widths for English text. German and Finnish run long; test with the "double-length pseudolanguage" scheme option.
- Use leading and trailing constraints and `semanticContentAttribute` only for views that should not mirror (media controls).
- Format numbers, dates, measurements and lists with `formatted()` or the matching formatters so they follow the user's locale.
