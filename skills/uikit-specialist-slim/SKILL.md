---
name: uikit-specialist-slim
description: Short list of what agents get wrong or forget when writing UIKit for iOS 18 and later. Use when building or changing UIKit screens, especially chat or feed lists (UICollectionView with diffable data sources), scroll position, self-sizing cells, selectable text, trait changes, edit menus, and main-actor rules.
---

# UIKit, only what models miss

You already know UIKit. This skill holds only the things that go wrong or have changed. Target iOS 18 unless the project says otherwise; guard newer APIs with `if #available`. Match the project's architecture; do not migrate working code unless it is the bug.

## 1. Changed or deprecated APIs

| Do not write | Write |
|---|---|
| `traitCollectionDidChange(_:)` (deprecated iOS 17) | `registerForTraitChanges([UITraitPreferredContentSizeCategory.self]) { (self: Self, previous: UITraitCollection) in ... }`. The closure gets `self`, so no `[weak self]`. |
| Caching a trait in a property | On iOS 18, traits read inside `layoutSubviews()`, `updateConstraints()`, `draw(_:)`, `viewWillLayoutSubviews()`, `updateConfiguration(using:)`, `configurationUpdateHandler` and a compositional section provider are tracked: UIKit re-runs the method when the trait changes. Read them there. |
| `setOverrideTraitCollection(_:forChild:)` | `traitOverrides` |
| `UIMenuController` | `UIEditMenuInteraction` (custom views) or `UITextViewDelegate` `textView(_:editMenuForTextIn:suggestedActions:)` |
| Only the single-range edit menu method | iOS 26 adds `textView(_:editMenuForTextInRanges:suggestedActions:)` for multi-range selections. Implement both when building with the iOS 26 SDK. |
| `UITableViewCell.textLabel`, `detailTextLabel`, `imageView` (deprecated in the iOS 27 SDK) | `defaultContentConfiguration()`, then assign to `contentConfiguration` |
| `UIScreen.main`, `UIApplication.shared.windows`, `keyWindow` | `view.window?.windowScene`, `traitCollection.displayScale`, container size |
| Keyboard notifications plus frame math | `view.keyboardLayoutGuide` |
| `contentEdgeInsets`, `titleEdgeInsets`, `imageEdgeInsets` | `UIButton.Configuration` |
| `UIGraphicsBeginImageContextWithOptions` | `UIGraphicsImageRenderer` |
| `UIActivityIndicatorView` as an empty or loading screen | `contentUnavailableConfiguration` (`.loading()`, `.empty()`, `.search()`); set it in `updateContentUnavailableConfiguration(using:)` |
| Link taps through the old URL delegate methods | `textView(_:primaryActionFor:defaultAction:)` and `textView(_:menuConfigurationFor:defaultMenu:)` (iOS 17) |
| `viewDidLoad` for first scroll or bounds | `viewIsAppearing(_:)`; bounds are not final in `viewDidLoad` |
| Observation in UIKit | iOS 26: `@Observable` properties read in `layoutSubviews()`, `updateProperties()`, `updateConfiguration(using:)` are tracked (on iOS 18 only with the `UIObservationTrackingEnabled` Info.plist key). On iOS 18 keep an explicit update path. |
| `UIUpdateLink` | iOS 18 replacement for `CADisplayLink` for per-frame work tied to the display |
| Zoom transition by hand | iOS 18: `preferredTransition = .zoom(sourceViewProvider:)`; return the current cell, since cells are reused |
| Opaque backgrounds behind bars | iOS 26 Liquid Glass restyles standard bars and controls; leave them transparent and use system colors and SF Symbols |
| `uniformAcrossSiblings(estimate:)` | iOS 17: group height shared by the tallest item in a row |

## 2. Correctness rules models get wrong

### Text that can be selected

`UILabel` cannot be selected. For selectable message text use a non-editable `UITextView`; selection fails silently if any of these is missing:

```swift
let textView = UITextView()
textView.isEditable = false
textView.isSelectable = true          // the default, but set it
textView.isScrollEnabled = false      // sizes like a label; needed in self-sizing cells
textView.textContainerInset = .zero
textView.textContainer.lineFragmentPadding = 0
textView.backgroundColor = .clear
textView.adjustsFontForContentSizeCategory = true
textView.font = .preferredFont(forTextStyle: .body)
```

- Check that a gesture recogniser on the cell or an overlay view is not eating the touches. A `UIContextMenuInteraction` on the whole cell also uses long press, so it competes with selecting a word. Decide which wins: put the menu on the cell and add a "Select Text" action, or attach the menu outside the text area. Verify by long-pressing text on a device: the simulator hides this.
- Do not set `isUserInteractionEnabled = false` on the text view or on `contentView` ancestors.
- If the whole message is only ever copied, a context menu action writing to `UIPasteboard.general` is enough and cheaper.
- A text view in a cell needs `delegate` set for the edit menu methods above.

### Scroll position when older items are prepended

Never "add the content size delta to the offset": with estimated cell sizes the content size keeps changing. Anchor on a visible item instead, in one main-actor turn with no animation:

```swift
@MainActor
func prepend(_ olderIDs: [ItemID]) async {
    let visibleTop = collectionView.contentOffset.y + collectionView.adjustedContentInset.top
    let anchor = collectionView.indexPathsForVisibleItems.sorted().lazy
        .compactMap { ip -> (ItemID, CGFloat)? in
            guard let a = self.collectionView.layoutAttributesForItem(at: ip),
                  a.frame.minY >= visibleTop,
                  let id = self.dataSource.itemIdentifier(for: ip) else { return nil }
            return (id, a.frame.minY - visibleTop)
        }.first

    var snapshot = dataSource.snapshot()
    if let first = snapshot.itemIdentifiers.first {
        snapshot.insertItems(olderIDs, beforeItem: first)
    } else {
        snapshot.appendItems(olderIDs)
    }
    await dataSource.apply(snapshot, animatingDifferences: false)

    guard let (id, distance) = anchor, let ip = dataSource.indexPath(for: id) else { return }
    collectionView.layoutIfNeeded()
    guard let a = collectionView.layoutAttributesForItem(at: ip) else { return }
    collectionView.setContentOffset(
        CGPoint(x: 0, y: a.frame.minY - distance - collectionView.adjustedContentInset.top),
        animated: false)
}
```

- `setContentOffset` stops a drag or deceleration in progress: wait until the scroll ends before prepending.
- Start loading older content before the user reaches the top.
- The same anchor save and restore is how you keep place across rotation (`viewWillTransition(to:with:)`), a Dynamic Type change, and relaunch. Save an item identifier plus its distance from the top, never a raw `contentOffset`.

### Staying pinned to the bottom (chat, logs)

Keep natural order; never flip the list with a transform (it breaks accessibility, menus and scroll indicators). Read "pinned" before the update, because the content size has already changed after it:

```swift
var isPinnedToBottom: Bool {
    let bottom = collectionView.contentOffset.y + collectionView.bounds.height
        - collectionView.adjustedContentInset.bottom
    return bottom >= collectionView.contentSize.height - 24
}
```

- Append, `await apply`, then `scrollToItem(at:at:.bottom, animated:)` only if it was pinned. Not pinned: show a "new messages" button.
- A fast stream (many messages per second) must be batched: one snapshot apply per frame or per short interval, not one per message. Each apply diffs the whole snapshot.
- Also re-pin after the keyboard appears, after rotation, after a text size change and after a cell grows (a reaction added to the last row).
- First load of a long conversation: apply without animation, `layoutIfNeeded()`, scroll to the last item without animation, in `viewIsAppearing(_:)`.
- A far-away `scrollToItem` with estimated sizes lands off target. For an exact jump: scroll without animation, `layoutIfNeeded()`, scroll to the same item again.
- Do not call `scrollToItem` in the same turn as an animated `apply`; await the apply first.

### Self-sizing cells

- Layout dimensions `.estimated`; constraints pinned to all four edges of `contentView`; no fixed height on text.
- When a cell changes its own height (expanded text, a reaction row appears), call `invalidateIntrinsicContentSize()` in the cell. This only works with `collectionView.selfSizingInvalidation` at `.enabled` (default since iOS 16), or `.enabledIncludingConstraints` when constraint changes should also resize it. Otherwise reconfigure the item. Re-pin to the bottom afterwards if the list was pinned.
- Do not measure heights by hand with `systemLayoutSizeFitting`.

### Diffable data sources

- Identifiers are small, stable, `Hashable` and `Sendable` (an ID, not the model). Look the model up when configuring the cell. Duplicate identifiers crash.
- Content changed, identity same: `snapshot.reconfigureItems(ids)`. It reuses the visible cell and keeps its state. `reloadItems` replaces the cell (selection, selected text and in-flight animations are lost) and is for when the cell type changes. Never delete and reinsert to refresh.
- Never mix `performBatchUpdates` with a diffable data source, and never call `reloadData()` for a small change.
- Create `CellRegistration` objects once, not inside the cell provider.
- Resolve identifiers now, not index paths captured earlier in closures, menus or async callbacks.
- Cell provider work is "look up and assign". Decode and downscale images off the main thread (`byPreparingThumbnail(ofSize:)`), cache, and check the cell still shows the same item before assigning. Reset cell state in `prepareForReuse()` or the registration handler.

### Dynamic Type and colors

- `UIFont.preferredFont(forTextStyle:)` or `UIFontMetrics.scaledFont(for:)`, plus `adjustsFontForContentSizeCategory = true`. Attributed strings do not always update live: rebuild them in a trait registration for `UITraitPreferredContentSizeCategory`.
- At accessibility categories (`preferredContentSizeCategory.isAccessibilityCategory`) switch horizontal rows to vertical stacks. Test at the largest size.
- `CGColor` does not follow dark mode: set `layer.borderColor` and `shadowColor` with `color.resolvedColor(with: traitCollection).cgColor` inside a trait-tracked method.

### Main actor and tasks

- UIKit types are `@MainActor`. A `Task {}` created in a view controller inherits the main actor, and a plain `async` method on a main-actor class also runs on main. `async` does not mean background. Move CPU work into an `actor`, a nonisolated type, or a `@concurrent` function (Swift 6.2; check the project's Default Actor Isolation setting first).
- Store the `Task`, cancel it in `viewDidDisappear` or `deinit`, check `Task.isCancelled` after each `await` that precedes UI work. Use `[weak self]` for tasks that outlive the screen (network, streams).
- Pass `Sendable` values across actors (IDs, structs, `Data`, `UIImage`); never views or mutable model classes.
- Consume `AsyncSequence` streams in a stored task and apply UI changes in batches.
- Never block main: `Data(contentsOf:)`, `DispatchQueue.main.sync`, semaphores, `Thread.sleep`. Timers on the default run loop mode stop while scrolling: use `.common` or `Task.sleep(for:)`.

### Smaller traps

- Do not change constraints inside `layoutSubviews()` (can loop). Do not call `layoutIfNeeded()` in loops.
- Shadow plus corner radius plus `masksToBounds` on scrolling cells needs a `shadowPath`.
- Nested collection views for horizontal rows: use orthogonal scrolling sections.
- Do not use `UIHostingConfiguration` or any SwiftUI to fix a UIKit layout problem.

## 3. Checklist before you say done

1. Builds with no new deprecation warnings (check for the table in section 1).
2. Selectable text: long press selects a word and the edit menu opens.
3. Prepend: the visible item does not move (within a point or two).
4. Pinned list stays pinned after append, keyboard, rotation, text size change, a cell growing.
5. Fast updates are batched; snapshot applied from one place; reconfigure not reload.
6. Nothing slow in cell configuration, `layoutSubviews` or `scrollViewDidScroll`.
7. Tasks stored and cancelled; no work off the main actor touching views.
8. Largest accessibility text size and dark mode look right; image-only buttons have `accessibilityLabel`.
9. `git grep -l 'import SwiftUI'` finds nothing, unless the project mixes them on purpose.
10. Say how you checked: which device or simulator, which scenarios.
