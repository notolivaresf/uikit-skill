# What to avoid

A quick list for reviews. Each line names the problem and the replacement.

## Old APIs with better replacements

| Avoid | Use instead |
|---|---|
| `register(_:forCellWithReuseIdentifier:)` plus string identifiers and casts | `UICollectionView.CellRegistration` and `dequeueConfiguredReusableCell(using:for:item:)` |
| `UICollectionViewDataSource` with index-path bookkeeping | `UICollectionViewDiffableDataSource` |
| `performBatchUpdates` with manual insert and delete index paths | snapshots |
| `UICollectionViewFlowLayout` subclasses for complex layouts | `UICollectionViewCompositionalLayout` |
| `textLabel`, `detailTextLabel`, `imageView` on `UITableViewCell` (replaced by content configurations since iOS 14, marked deprecated in the iOS 27 SDK) | `defaultContentConfiguration()` |
| `traitCollectionDidChange(_:)` (deprecated in iOS 17) | `registerForTraitChanges(_:handler:)` or automatic trait tracking |
| `UIMenuController` (deprecated) | `UIEditMenuInteraction`, `textView(_:editMenuForTextIn:suggestedActions:)` |
| `UIScreen.main` for size, scale or orientation | the view's `traitCollection`, `window?.windowScene`, container size |
| `UIApplication.shared.windows` / `keyWindow` | `view.window`, the scene's windows |
| `statusBarOrientation`, `interfaceOrientation` | size and size classes, `windowScene.effectiveGeometry` |
| Keyboard notifications plus frame conversion | `keyboardLayoutGuide` |
| `UIActivityIndicatorView` overlays for empty or loading screens | `UIContentUnavailableConfiguration.loading()` |
| `setOverrideTraitCollection(_:forChild:)` | `traitOverrides` |
| `UIGraphicsBeginImageContextWithOptions` | `UIGraphicsImageRenderer` |
| `contentEdgeInsets`, `titleEdgeInsets`, `imageEdgeInsets` on buttons (deprecated) | `UIButton.Configuration` |
| `UIWebView` | `WKWebView` |
| `NotificationCenter` selectors for every model update | an explicit update path (stream, delegate) or Observation tracking where supported |

## Patterns that hurt performance

- `reloadData()` after every change.
- Model structs as diffable identifiers, so every edit becomes delete plus insert.
- Doing anything slow in cell configuration: decoding images or JSON, creating formatters, building long attributed strings, measuring text by hand.
- Decoding full-size images into small image views.
- Shadows without `shadowPath`, `masksToBounds` with corner radius over changing content, many blur views.
- Nesting collection views inside cells for horizontal rows; use orthogonal scrolling sections.
- Heavy work in `scrollViewDidScroll(_:)` or `layoutSubviews()`.
- Applying one snapshot per incoming event in a fast stream; batch them.
- Calling `layoutIfNeeded()` in loops, or on the whole window, to "fix" a layout.
- Using `systemLayoutSizeFitting` on a sizing cell to compute heights manually; let self-sizing do it.

## Patterns that cause bugs

- Using index paths captured earlier (in closures, menus, async callbacks); resolve the identifier now.
- Forgetting to reset cell state in `prepareForReuse()` or in the registration handler, so old images or selection show on reused cells.
- Assigning an async-loaded image without checking that the cell still shows the same item.
- Changing constraints inside `layoutSubviews()`, which can loop.
- Reading `view.bounds` in `viewDidLoad` and caching it.
- Hard-coding status bar, navigation bar or keyboard heights.
- Inverting a scroll view with a transform for chat.
- Fixed heights on text containers, which clip at large text sizes.
- `UILabel` where users expect to select text.
- Retain cycles: closures stored on `self` (registrations, handlers, timers, tasks) capturing `self` strongly.
- `Timer` scheduled on the main run loop default mode stops during scrolling; use `.common` mode, or better, a `Task` with `Task.sleep(for:)` for delays.

## Mixing frameworks

- Do not wrap a UIKit screen in SwiftUI (or the reverse) to solve one layout problem. Mixing has a cost in layout passes and in debugging; do it only when the project has decided on it.
- `UIHostingConfiguration` lets a SwiftUI view render inside a collection view cell. In a UIKit-only codebase, use a `UIContentConfiguration` instead.
