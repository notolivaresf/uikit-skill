# Interactions

Context menus, text selection, edit menus, swipe actions, drag and drop, and pointer and keyboard support.

## Context menus in collection views

Use the delegate methods that take a set of index paths (iOS 16+); they support multiple selection and are the current API.

```swift
func collectionView(_ collectionView: UICollectionView,
                    contextMenuConfigurationForItemsAt indexPaths: [IndexPath],
                    point: CGPoint) -> UIContextMenuConfiguration? {
    let ids = indexPaths.compactMap { dataSource.itemIdentifier(for: $0) }
    guard !ids.isEmpty else { return nil }
    return UIContextMenuConfiguration(identifier: nil, previewProvider: nil) { [weak self] _ in
        guard let self else { return nil }
        let copy = UIAction(title: String(localized: "Copy"), image: UIImage(systemName: "doc.on.doc")) { _ in
            self.copy(ids)
        }
        let delete = UIAction(title: String(localized: "Delete"), image: UIImage(systemName: "trash"),
                              attributes: .destructive) { _ in
            self.delete(ids)
        }
        return UIMenu(children: [copy, delete])
    }
}
```

- Capture identifiers, not index paths, in the actions; the list can change while the menu is open.
- Build the menu lazily in the action provider. Use `UIDeferredMenuElement` for items that need async work (for example "Open in" targets).
- Customise the lifted preview with `collectionView(_:contextMenuConfiguration:highlightPreviewForItemAt:)` and `...dismissalPreviewForItemAt:` returning a `UITargetedPreview` with `UIPreviewParameters` (a rounded `visiblePath` for bubble-shaped cells).
- For views that are not cells, add a `UIContextMenuInteraction` to the view and implement its delegate. Buttons can show a menu directly with `button.menu` and `showsMenuAsPrimaryAction`.
- The same menu also appears on secondary click on iPad with a pointer and on Mac Catalyst, so it doubles as the right-click menu.

## Text selection

`UILabel` does not support selection. When users must select part of a text (messages, notes, code, anything they may want to quote), display it in a non-editable `UITextView`.

```swift
let textView = UITextView()
textView.isEditable = false
textView.isSelectable = true
textView.isScrollEnabled = false          // lets it size to its content like a label
textView.backgroundColor = .clear
textView.textContainerInset = .zero
textView.textContainer.lineFragmentPadding = 0
textView.adjustsFontForContentSizeCategory = true
textView.font = .preferredFont(forTextStyle: .body)
textView.dataDetectorTypes = [.link]
```

- With `isScrollEnabled = false` the text view has an intrinsic content size and works in self-sizing cells.
- Remove the default insets (`textContainerInset`, `lineFragmentPadding`) so it lines up with neighbouring labels.
- Selection in a cell competes with the cell's own gestures. Long press selects a word in the text view; a context menu on the cell also uses long press. Decide which wins: typical apps let the context menu own the whole cell and add a "Select Text" or "Copy" action, or keep selection active and attach the context menu to an area outside the text. Test both on device; the simulator hides the conflict.
- Customise the edit menu that appears for a selection with `textView(_:editMenuForTextIn:suggestedActions:)` (iOS 16+) on `UITextViewDelegate`; iOS 26 adds `textView(_:editMenuForTextInRanges:suggestedActions:)` for multi-range selections, so implement that one too when you build with the iOS 26 SDK. Return a `UIMenu` that adds your actions to the suggested ones.
- Links: `UITextViewDelegate`'s `textView(_:primaryActionFor:defaultAction:)` and `textView(_:menuConfigurationFor:defaultMenu:)` (iOS 17+) handle taps and menus on links and other text items. Prefer them to the older URL interaction methods.
- If the text should be selectable only as a whole (copy the message), skip the text view: a context menu action that writes to `UIPasteboard.general` is simpler and cheaper.
- A text view is heavier than a label. In very long lists, measure; if needed, show a label normally and swap to a text view when the user starts a selection gesture.

## Edit menus for custom views

For custom views that need Copy, Paste or custom commands on selection, use `UIEditMenuInteraction` (iOS 16+). `UIMenuController` is deprecated.

```swift
let editMenu = UIEditMenuInteraction(delegate: self)
canvas.addInteraction(editMenu)

func showMenu(at point: CGPoint) {
    let configuration = UIEditMenuConfiguration(identifier: nil, sourcePoint: point)
    editMenu.presentEditMenu(with: configuration)
}

func editMenuInteraction(_ interaction: UIEditMenuInteraction,
                         menuFor configuration: UIEditMenuConfiguration,
                         suggestedActions: [UIMenuElement]) -> UIMenu? {
    UIMenu(children: suggestedActions + [UIAction(title: "Duplicate") { [weak self] _ in self?.duplicate() }])
}
```

Standard commands (copy, paste, select all) come from the responder chain. Override `canPerformAction(_:withSender:)` and implement `copy(_:)` on the responder to take part.

## Swipe actions

With list layouts, provide `leadingSwipeActionsConfigurationProvider` and `trailingSwipeActionsConfigurationProvider` on `UICollectionLayoutListConfiguration`. Return `UISwipeActionsConfiguration` with `UIContextualAction`s; call the completion handler with `true` when the action finished. Resolve the item through the data source at the time of the swipe.

Every swipe action should also exist in the context menu and as an accessibility custom action, because swipes are not discoverable.

## Drag and drop

- Collection views: set `dragDelegate` and `dropDelegate`, return `UIDragItem`s with `NSItemProvider`s in `collectionView(_:itemsForBeginning:at:)`, and use `dragInteractionEnabled = true` on iPhone (it is on by default on iPad).
- Reordering within a diffable data source: set `dataSource.reorderingHandlers.canReorderItem` and `didReorder`, and update your model from the transaction's final snapshot.
- Any view: `UIDragInteraction` and `UIDropInteraction`.
- Load dropped data asynchronously from the item provider; never block the main thread waiting for it.

## Pointer, keyboard and focus

- Buttons and bar items get pointer effects for free. For custom views, add `UIPointerInteraction` and return a `UIPointerStyle` (`.highlight` or `.lift`) from its delegate.
- Add keyboard shortcuts with `UIKeyCommand` in `keyCommands` or, better, in the main menu with `buildMenu(with:)` on the app delegate, so they appear in the iPad shortcut overlay and the Mac menu bar.
- Hover effects: `hoverStyle` on views (iOS 17+) for visionOS and pointer.
- Collection views support focus-based keyboard navigation with `allowsFocus = true`; set `selectionFollowsFocus` where arrow keys should select.

## Haptics

Use `UIFeedbackGenerator` subclasses (`UIImpactFeedbackGenerator`, `UISelectionFeedbackGenerator`, `UINotificationFeedbackGenerator`). On iOS 17.5+ create them with `init(view:)` so feedback can be tied to a location, and call `prepare()` just before an expected event. Many standard controls already play haptics; do not double them.
