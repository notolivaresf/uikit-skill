---
name: uikit-specialist
description: Write, review and speed up modern UIKit code for iOS 18 and later. Use when building or changing UIKit screens, especially lists and feeds (UICollectionView with compositional layout and diffable data sources), self-sizing cells, content configurations, image loading, scroll position, keyboard and safe area handling, Dynamic Type, size classes, context menus, Auto Layout, Swift concurrency in view controllers, and when profiling hitches or slow scrolling with Instruments.
---

# UIKit specialist

Use this skill to write UIKit the way it is meant to be written today: collection views driven by diffable data sources, cells built from registrations and configurations, layout that adapts to traits, and work kept off the main thread. Target iOS 18 unless the project says otherwise; call out anything that needs a newer release and guard it with `if #available`.

## How to use this skill

1. Read the project first. Match its deployment target, its architecture and its naming. Do not migrate working code to a new pattern unless asked or unless the old pattern is the bug.
2. Pick the reference file for the task (table below) and read only that one. Each is self-contained.
3. When you write code, follow the rules in "Always" and "Never" below. When you review code, report findings in the order: crashes and data races, main-thread stalls, layout and accessibility bugs, then style.
4. After a change that touches scrolling, layout or images, say how to check it (the Instruments section of `references/performance.md`).

| Task | Read |
|---|---|
| Any list, grid, feed or table | `references/collection-views.md` |
| Inserting, prepending, reloading, keeping the scroll position | `references/updates-and-scrolling.md` |
| Images, prefetching, hitches, Instruments | `references/performance.md` |
| Auto Layout, safe area, keyboard, rotation, size classes | `references/layout.md` |
| Fonts, Dynamic Type, traits, dark mode, accessibility | `references/text-and-traits.md` |
| Context menus, text selection, edit menus, drag and drop | `references/interactions.md` |
| async/await, actors, tasks and view controller lifetime | `references/concurrency.md` |
| Quick list of what to avoid | `references/avoid.md` |

## Always

- Build lists with `UICollectionView`, `UICollectionViewCompositionalLayout` and `UICollectionViewDiffableDataSource`. For a plain vertical list use `UICollectionLayoutListConfiguration`, not `UITableView`, in new code. Keep an existing `UITableView` that works; give it a `UITableViewDiffableDataSource` if it still calls `reloadData()` for every change.
- Make item identifiers small, stable, `Hashable` and `Sendable` (an ID, not the whole model). Look the model up when you configure the cell. When content changes but identity does not, `reconfigureItems(_:)`; never delete and reinsert to refresh.
- Register cells with `UICollectionView.CellRegistration` and supplementary views with `UICollectionView.SupplementaryRegistration`. Create registrations once (as properties or before the data source), never inside the cell provider.
- Describe cell content with `UIListContentConfiguration` or a custom `UIContentConfiguration`, and backgrounds with `UIBackgroundConfiguration`. Update for state (selected, highlighted, editing) in `updateConfiguration(using:)` or `configurationUpdateHandler`.
- Let cells size themselves: `.estimated` dimensions in the layout, constraints that pin content to all four edges of `contentView`, and `selfSizingInvalidation` left on.
- Decode and downscale images off the main thread (`byPreparingThumbnail(ofSize:)` or `byPreparingForDisplay()`), cache the result, and check that the cell still shows the same item before you assign it.
- Use text styles: `UIFont.preferredFont(forTextStyle:)` or `UIFontMetrics` for custom fonts, with `adjustsFontForContentSizeCategory = true`. Scale fixed spacing with `UIFontMetrics.scaledValue(for:)`.
- Respond to trait changes with `registerForTraitChanges(_:handler:)` (iOS 17+); `traitCollectionDidChange(_:)` is deprecated. On iOS 18, traits read inside `layoutSubviews()`, `updateConstraints()`, `draw(_:)` and `updateConfiguration(using:)` are tracked for you.
- Pin to `safeAreaLayoutGuide`, `layoutMarginsGuide` or `readableContentGuide`, and to `keyboardLayoutGuide` for anything that must sit above the keyboard.
- Treat UIKit types as `@MainActor`. Start async work from a `Task` you store and cancel when the view goes away; hop back to the main actor only to touch views.
- Use semantic colors (`.label`, `.secondarySystemBackground`, asset catalog colors) and SF Symbols so dark mode, increased contrast and Liquid Glass work without extra code.

## Never

- Call `reloadData()` to show a small change, or mix `performBatchUpdates` with a diffable data source.
- Do work in `cellForItemAt` or a cell provider that is not "look up and assign": no networking, decoding, date formatting with a new formatter, or layout passes.
- Set `translatesAutoresizingMaskIntoConstraints` on a cell's `contentView`, add subviews to the cell instead of `contentView`, or give self-sizing cells a fixed height constraint that fights the estimate.
- Block the main thread with `Data(contentsOf:)`, synchronous file reads, `DispatchQueue.main.sync`, semaphores or `Thread.sleep`.
- Use `UIScreen.main` for sizes or scale; read the window scene, the view's `traitCollection.displayScale`, or the container size.
- Flip a list upside down with a transform to make a chat; it breaks accessibility, context menus and scroll indicators. Keep the order natural and manage the offset (see `references/updates-and-scrolling.md`).
- Add `cornerRadius` plus `masksToBounds` plus shadow on the same layer in scrolling cells without a `shadowPath`.
- Reach for `UIHostingConfiguration` or a SwiftUI view to solve a UIKit layout problem unless the project already mixes the two on purpose.

## Review checklist

When asked to review UIKit code, walk these in order and quote the line for each finding:

1. Main thread: anything slow on the main actor, especially in cell configuration, `viewDidLoad`, `layoutSubviews` or scroll delegate methods.
2. Data source: identifiers stable and unique (duplicate identifiers crash diffable data sources), snapshots applied from one place, reconfigure instead of reload.
3. Cells: reuse safe (state reset, image tasks cancelled), sizing constraints complete, no work repeated per configuration that could be cached.
4. Layout: no ambiguous or conflicting constraints, no `frame` math that ignores safe area or margins, works in both size classes and at the largest accessibility text size.
5. Lifetime: no retain cycles in closures (`[weak self]` in long-lived closures, registrations that capture the controller), tasks and observers cancelled.
6. Accessibility: labels on image-only buttons, traits set, Dynamic Type honoured, VoiceOver order matches the visual order.

Keep answers practical: show the changed code, say why in one line, and point to the reference file for the longer explanation.
