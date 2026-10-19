# Collection views

The modern UIKit list is three parts that each do one job: a compositional layout says where things go, a diffable data source says what is there, and cell registrations with content configurations say what each item looks like. Keep them separate and each stays simple.

## Layout

### Lists

For anything that looks like a table, start from the list configuration. It gives you separators, swipe actions, headers, outline disclosure and the system look for free.

```swift
var config = UICollectionLayoutListConfiguration(appearance: .insetGrouped)
config.headerMode = .supplementary
config.trailingSwipeActionsConfigurationProvider = { [weak self] indexPath in
    self?.swipeActions(at: indexPath)
}
let layout = UICollectionViewCompositionalLayout.list(using: config)
```

To mix lists with other section types, build the layout per section:

```swift
let layout = UICollectionViewCompositionalLayout { [weak self] sectionIndex, environment in
    guard let self, let section = self.dataSource.sectionIdentifier(for: sectionIndex) else { return nil }
    switch section {
    case .featured:
        return Self.carouselSection()
    case .all:
        var list = UICollectionLayoutListConfiguration(appearance: .plain)
        list.headerMode = .supplementary
        return NSCollectionLayoutSection.list(using: list, layoutEnvironment: environment)
    }
}
```

The section provider is called again when the container size or traits change, so read `environment.container.effectiveContentSize` and `environment.traitCollection` there to adapt the number of columns. On iOS 18 traits read inside the section provider are tracked automatically.

### Grids and custom sections

Describe sizes relative to the container or as estimates; avoid absolute widths.

```swift
static func gridSection(environment: NSCollectionLayoutEnvironment) -> NSCollectionLayoutSection {
    let width = environment.container.effectiveContentSize.width
    let columns = max(2, Int(width / 180))
    let item = NSCollectionLayoutItem(layoutSize: .init(
        widthDimension: .fractionalWidth(1.0 / CGFloat(columns)),
        heightDimension: .estimated(220)))
    let group = NSCollectionLayoutGroup.horizontal(
        layoutSize: .init(widthDimension: .fractionalWidth(1), heightDimension: .estimated(220)),
        repeatingSubitem: item, count: columns)
    group.interItemSpacing = .fixed(8)
    let section = NSCollectionLayoutSection(group: group)
    section.interGroupSpacing = 8
    section.contentInsetsReference = .layoutMargins
    return section
}
```

- `fractionalWidth` and `fractionalHeight` are relative to the containing group, not the screen.
- `.estimated` lets the cell decide its own size; `.absolute` is for things that truly never change, such as a fixed avatar strip.
- When items in a row should share the tallest height, use `.uniformAcrossSiblings(estimate:)` (iOS 17+) for the group's height instead of measuring yourself.
- `orthogonalScrollingBehavior` (`.continuous`, `.groupPaging`, `.paging`) gives horizontal carousels inside a vertical list without nesting collection views. Avoid nesting a collection view inside a cell.
- `contentInsetsReference = .layoutMargins` or `.readableContent` keeps content aligned with the rest of the system on wide screens.
- Boundary supplementary items (headers, footers) can be pinned with `pinToVisibleBounds = true`.
- `visibleItemsInvalidationHandler` is for cheap visual effects on visible items (scale, alpha). It runs during scrolling, so keep it to arithmetic.

## Data source

### Identifiers

Use identifiers that are cheap to hash and stable across updates. An ID type is ideal.

```swift
enum Section: Hashable, Sendable { case featured, all }
struct ItemID: Hashable, Sendable { let rawValue: UUID }
```

If you use whole model structs as identifiers, every change to any field makes the diff see a delete plus insert, which animates badly and drops cell state. If two items in a snapshot share an identifier, the data source raises an exception. De-duplicate before you build the snapshot.

### Building and applying snapshots

```swift
var snapshot = NSDiffableDataSourceSnapshot<Section, ItemID>()
snapshot.appendSections([.featured, .all])
snapshot.appendItems(featuredIDs, toSection: .featured)
snapshot.appendItems(allIDs, toSection: .all)
dataSource.apply(snapshot, animatingDifferences: true)
```

- Apply snapshots from one place and always from the main actor. Since iOS 15, `apply(_:animatingDifferences: false)` still diffs; it no longer falls back to a full reload.
- Use `applySnapshotUsingReloadData(_:)` only when you really want to throw away all cell state, for example after switching accounts.
- To change content of existing items, take the current snapshot, call `reconfigureItems(_:)` with the changed IDs and apply. Reconfigure reuses the existing cell and runs your registration handler again; `reloadItems(_:)` dequeues a new cell and is slower. Use reload only when the cell type changes.
- Section snapshots (`NSDiffableDataSourceSectionSnapshot`) handle expandable outlines and let you update one section without rebuilding the others. Apply them with `apply(_:to:animatingDifferences:)`.
- Large snapshots (tens of thousands of items) are fine to build on a background queue as plain arrays of IDs; append them in one call rather than item by item.
- `apply` has an async variant; `await dataSource.apply(snapshot)` when the next step depends on the update having finished, such as scrolling to a new item.

### Model storage

Keep models in a store keyed by ID (a dictionary, a database, an observable model object). The cell registration handler receives an ID, looks up the model and configures the cell. That way a model change is "update the store, reconfigure the ID", and the snapshot only changes when membership or order changes.

## Cells

### Registration

```swift
let cellRegistration = UICollectionView.CellRegistration<UICollectionViewListCell, ItemID> { [weak self] cell, indexPath, id in
    guard let self, let item = self.store[id] else { return }
    var content = cell.defaultContentConfiguration()
    content.text = item.title
    content.secondaryText = item.subtitle
    content.image = UIImage(systemName: item.symbolName)
    cell.contentConfiguration = content
    cell.accessories = item.isUnread ? [.customView(configuration: Self.unreadDot)] : []
}

dataSource = UICollectionViewDiffableDataSource<Section, ItemID>(collectionView: collectionView) { collectionView, indexPath, id in
    collectionView.dequeueConfiguredReusableCell(using: cellRegistration, for: indexPath, item: id)
}
```

Create the registration before the data source and outside the cell provider. Creating a registration inside the provider registers a new class every call and, on recent iOS versions, raises an exception.

Supplementary views work the same way with `UICollectionView.SupplementaryRegistration` and `dataSource.supplementaryViewProvider`.

### Content configurations

`UIListContentConfiguration` covers text, secondary text, image and their layout in the system style. Change its properties (`textProperties.font`, `imageProperties.tintColor`, `directionalLayoutMargins`) rather than adding your own labels next to it.

For custom cells, write a configuration and a content view:

```swift
struct MessageConfiguration: UIContentConfiguration, Hashable {
    var author = ""
    var body = ""
    var isHighlighted = false

    func makeContentView() -> UIView & UIContentView { MessageContentView(configuration: self) }

    func updated(for state: UIConfigurationState) -> MessageConfiguration {
        var copy = self
        if let cellState = state as? UICellConfigurationState {
            copy.isHighlighted = cellState.isHighlighted || cellState.isSelected
        }
        return copy
    }
}

final class MessageContentView: UIView, UIContentView {
    private let authorLabel = UILabel()
    private let bodyLabel = UILabel()
    private var current: MessageConfiguration

    var configuration: UIContentConfiguration {
        get { current }
        set {
            guard let new = newValue as? MessageConfiguration, new != current else { return }
            current = new
            apply(new)
        }
    }

    init(configuration: MessageConfiguration) {
        current = configuration
        super.init(frame: .zero)
        // build the view hierarchy and constraints once here
        apply(configuration)
    }

    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    private func apply(_ c: MessageConfiguration) {
        authorLabel.text = c.author
        bodyLabel.text = c.body
    }
}
```

- Make the configuration `Hashable` and skip work when it has not changed; the system sets it often.
- Build subviews and constraints once in `init`, then only change values in `apply`.
- Put state-dependent styling (highlight, selection, editing, swipe) in `updated(for:)`, in the cell's `updateConfiguration(using:)`, or in `configurationUpdateHandler`. Do not toggle colors in `isSelected` setters.
- Backgrounds go in `UIBackgroundConfiguration` (`.listCell()`, `.clear()`, custom corner radius and stroke) instead of layers you add yourself.
- A subclass of `UICollectionViewListCell` keeps accessories (disclosure, checkmark, reorder, delete, custom views) and swipe actions.

### Empty and loading states

Use `UIContentUnavailableConfiguration` (iOS 17+) on the view controller: set `contentUnavailableConfiguration` in `updateContentUnavailableConfiguration(using:)` and call `setNeedsUpdateContentUnavailableConfiguration()` when the data changes. `.loading()`, `.search()` and `.empty()` give the system styles.

## Self-sizing

- Use `.estimated` for the dimension that depends on content, usually height. A good estimate is close to the typical size; a wildly wrong estimate makes scroll indicators jump and costs extra layout passes.
- Constrain the content view's subviews to all four edges of `contentView` (or its `layoutMarginsGuide`) with no gaps in the vertical chain, so Auto Layout can compute a height.
- Leave multi-line labels at `numberOfLines = 0` and let them wrap; do not set `preferredMaxLayoutWidth` by hand inside cells.
- Keep `collectionView.selfSizingInvalidation` at `.enabled` (the default since iOS 16), or `.enabledIncludingConstraints` if constraint changes inside the cell should resize it. Then a cell that changes size (expanding text, a loaded image) can call `invalidateIntrinsicContentSize()` and the layout animates the change. Without it, reconfigure the item.
- If an image's size is known from metadata, set an aspect ratio constraint up front so the cell does not change height when the image arrives.
- When a cell height is constant and known, `.absolute` is cheaper than self-sizing. Use it for fixed rows, not to work around a constraint bug.

## Selection, editing and focus

- Selection state lives in the collection view (`indexPathsForSelectedItems`). Map index paths to identifiers with `dataSource.itemIdentifier(for:)` before storing anything.
- Use `collectionView.isEditing` and list cell accessories (`.delete()`, `.reorder()`, `.multiselect()`) for editing. Reordering with diffable data sources uses `dataSource.reorderingHandlers`.
- Support keyboard focus and selection follows focus on iPad and Mac Catalyst with `allowsFocus` and `selectionFollowsFocus`.

## Table views

`UITableView` is not deprecated and remains fine for existing screens. The same rules apply: `UITableViewDiffableDataSource`, `UIListContentConfiguration` via `defaultContentConfiguration()`, `reconfigureRows(at:)` or snapshot reconfigure, and self-sizing rows with `UITableView.automaticDimension` plus `estimatedRowHeight`.
