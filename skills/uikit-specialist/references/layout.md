# Layout

Auto Layout, safe areas, the keyboard and size changes. The goal is layout that is correct in every size, orientation, text size and multitasking arrangement without special cases.

## Auto Layout hygiene

- Set `translatesAutoresizingMaskIntoConstraints = false` on every view you constrain in code, and never on a view whose frame UIKit manages for you (a cell's `contentView`, a view controller's root `view`, a collection view's cells).
- Activate constraints together with `NSLayoutConstraint.activate([...])`. Keep references only to constraints you will change, and change their `constant` or `isActive` rather than removing and recreating them.
- Every view needs a full, unambiguous set: horizontal position and width, vertical position and height, from constraints or from intrinsic content size. Check with `view.hasAmbiguousLayout` and `exerciseAmbiguityInLayout()` in the debugger, or the view debugger's purple warnings.
- Read "Unable to simultaneously satisfy constraints" logs and fix the cause. Add `identifier` to constraints so the log names them. A symbolic breakpoint on `UIViewAlertForUnsatisfiableConstraints` stops at the moment it happens.
- Use priorities to say what may give way. A label that may truncate gets lower compression resistance; a label that should hug its text gets higher hugging priority. Use `.defaultHigh` (750) or 999 for "almost required" constraints that would otherwise conflict during transitions, such as a height that must yield while a cell is being resized.
- Use leading and trailing, not left and right, so right-to-left languages mirror correctly. Use `directionalLayoutMargins` and `NSDirectionalEdgeInsets`.
- `UIStackView` is a good default for rows and columns of controls. Hide arranged subviews with `isHidden` to collapse them; do not remove and re-add them to toggle.
- Use `UILayoutGuide` instead of empty spacer views.
- Do not mix frame-based layout and constraints on the same view. If you lay out by hand in `layoutSubviews`, use the view's `bounds` and its margins, call `super`, and keep it idempotent.

## Guides to pin to

| Guide | Use for |
|---|---|
| `safeAreaLayoutGuide` | content that must avoid the notch, Dynamic Island, home indicator and bars |
| `layoutMarginsGuide` | content that should line up with system margins |
| `readableContentGuide` | text-heavy content on wide screens, so lines stay readable |
| `keyboardLayoutGuide` | anything that sits above the keyboard, such as an input bar |

Scroll views and collection views should usually fill the whole view (edges to `view`, not to the safe area) and let `contentInsetAdjustmentBehavior` (default `.automatic`) add the safe area as content inset, so content scrolls under the bars. Read the real inset from `adjustedContentInset`. Add your own extra inset with `additionalSafeAreaInsets` on the view controller or `contentInset` on the scroll view.

## Keyboard

Use the keyboard layout guide (iOS 15+); it replaces observing keyboard notifications and converting frames.

```swift
NSLayoutConstraint.activate([
    inputBar.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    inputBar.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    inputBar.bottomAnchor.constraint(equalTo: view.keyboardLayoutGuide.topAnchor),
    collectionView.topAnchor.constraint(equalTo: view.topAnchor),
    collectionView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    collectionView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    collectionView.bottomAnchor.constraint(equalTo: inputBar.topAnchor),
])
```

- When the keyboard is hidden, the guide's top sits at the bottom safe area, so the input bar stays above the home indicator. Set `keyboardLayoutGuide.usesBottomSafeArea = false` if you want to handle that inset yourself.
- On iPad, set `followsUndockedKeyboard = true` if the bar should follow a floating or split keyboard.
- The guide animates with the keyboard. To keep a chat pinned to the bottom while the keyboard rises, check "pinned" before the change and scroll to the last item inside the same layout pass (for example in `viewDidLayoutSubviews` when the collection view's height changed and the list was pinned).
- `collectionView.keyboardDismissMode = .interactive` lets the user drag the keyboard down with the list.
- For a text view inside a scrolling form, scroll the focused field into view with `scrollRectToVisible(_:animated:)` after the layout updates; do not hard-code keyboard heights.

## Size classes, rotation and multitasking

Treat rotation and window resizing as the same event: the container size changed.

- Read the current width and height classes from `traitCollection.horizontalSizeClass` and `.verticalSizeClass`. An iPhone in landscape and an iPad in a narrow split view are both compact width; design for the class, not the device.
- Respond to size changes in `viewWillTransition(to:with:)`. Do layout-dependent work in the coordinator's `animate(alongsideTransition:completion:)` so it moves with the rotation.
- In compositional layouts, compute columns from `environment.container.effectiveContentSize` inside the section provider; it is called again for the new size.
- On iPad, the app can be any width at any time (Split View, Slide Over, Stage Manager, and windowed apps on recent iPadOS). Never assume full screen, never read `UIScreen.main.bounds` to size content, and test at narrow widths.
- Do not lock orientation to avoid layout work. If a screen truly needs it, override `supportedInterfaceOrientations` on that view controller and request updates with `setNeedsUpdateOfSupportedInterfaceOrientations()`.
- Scene-based apps get the window from the view (`view.window?.windowScene`), not from a global. Multiple windows of the same scene type can exist side by side.

## View controller lifecycle

- `viewDidLoad`: build the hierarchy and constraints once. Bounds are not final here.
- `viewIsAppearing(_:)`: the view is in the hierarchy with correct traits and size. Do final appearance-dependent setup here (initial scroll position, trait-dependent styling) rather than in `viewWillAppear`.
- `viewWillLayoutSubviews` and `viewDidLayoutSubviews`: may run many times. Keep them cheap and idempotent.
- `viewDidDisappear(_:)`: stop timers, cancel tasks that only matter while visible.
- Use child view controllers (`addChild`, `didMove(toParent:)`) to compose screens instead of one giant controller.

## Navigation and presentation

- Use `UINavigationController`, `UITabBarController` and `UISplitViewController`; they handle large titles, bars, safe area and adaptivity. On iOS 18 tabs can be described with `UITab` objects, and the tab bar adapts to a sidebar on iPad.
- For sheets use `UISheetPresentationController` with `.medium()` and `.large()` detents, and `prefersGrabberVisible`.
- `preferredTransition = .zoom(sourceViewProvider:)` (iOS 18+) gives the system zoom transition from a cell to a detail screen; return the current cell for the item from the provider, since cells are reused.
- Configure bar appearance with `UINavigationBarAppearance` on the navigation item (`standardAppearance`, `scrollEdgeAppearance`), not by setting colors on the bar at runtime.
- Recent iOS releases restyle bars and controls automatically (Liquid Glass on iOS 26). Use standard bars, buttons and system colors, and do not paint opaque backgrounds behind bars, so the system style applies.
