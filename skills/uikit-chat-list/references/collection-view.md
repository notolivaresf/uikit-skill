# The collection view path

Use only when the screen needs a `UICollectionView` (other layouts beside the messages, or a project that already has one). For streaming text a table view costs the same with about half the list code. [E1] Tags resolve in `../SOURCES.md`.

## Shape

Every fast collection-view build had the same shape: one section, a `UICollectionViewLayout` subclass that owns exact row frames worked out from saved heights, and explicit item changes. [F1] [F1.1]

- Not a compositional layout with one section per message: S2 and SC2 shipped that and were the slowest; one section with a custom layout took stream CPU from 9.8 to 7.5 s and 15.6 to 8.8 s. [F1.1] [F1.11]
- Not `UICollectionViewFlowLayout` with self-sizing: no build used it, so there is no evidence for it. **PENDING E2** for self-sizing. [F1]

## The layout class

- Keep an array of row frames sorted by y (or a height index such as a Fenwick tree or treap when prepends are frequent: S3, SC3). [F1.1 S3, SC3]
- `prepare()` does the work once per invalidation and caches the attributes. [WWDC18-225]
- `layoutAttributesForElements(in:)` binary-searches the sorted frames for the rect instead of filtering every row; WWDC18 225 shows the linear filter slowing scrolling as the item count grows. [WWDC18-225] [F1.3 AppKit ObjC Opus]
- `layoutAttributesForItem(at:)` reads the cached frame; `collectionViewContentSize` is the last frame's max y.
- Appends are incremental: compute frames only for the new rows (O3 `RoomLayout.swift:31-43`). Recomputing every frame in `prepare()` on each invalidation was O2's stream and load-older cost. [F1.1 O2, O3] [F1.4 O2]
- `shouldInvalidateLayout(forBoundsChange:)` returns true only when the width changes, not on every scroll. [SDK-UICollectionViewLayout] [WWDC18-225]
- `collectionView.selfSizingInvalidation = .disabled`: heights come from your measurer, so a cell's content change must not resize it behind the layout's back. [F1.1 S3] [WWDC22-10068] [SDK-UICollectionView]

## Updates

- Inserts with `insertItems(at:)` inside `performBatchUpdates`, data source changed in the same closure; changed messages with `reconfigureItems(at:)`, which keeps the existing cell. [WWDC18-225] [SDK-UICollectionView] [WWDC21-10252]
- With a diffable data source: one section, edit the current snapshot, never rebuild every section per flush (S2 compared every loaded row's key each time). `applySnapshotUsingReloadData` is for resetting to a different data set, such as a jump. [F1.1 S2] [F1.3 S3] [SDK-UIDiffableDataSource]
- Pinning: read pinned before, insert without animation, then `setContentOffset` to the bottom; in the Round 5 spike this held all 600 messages pinned. [E1]
- Batch-update bookkeeping was estimated at about 7 ms per live update in one build's notes. Not measured separately; not a reason to leave the collection view. [F1.1 OC3] [F §4]

## The offset after updates

`targetContentOffset(forProposedContentOffset:)` is how a layout supplies the offset to use during an update or transition (the SDK header: "a layout can return the content offset to be applied during transition or update animations"). [SDK-UICollectionViewLayout]

- Load older: record the anchor (top visible id and its distance from the top) before the update, and return the offset that puts it back. O3 did this and no cells were made for the wrong place first. **PENDING E3** may name a different anchor. [F1.4 O3]
- Removals, Reset and shrinking text: return the final, in-range offset here, or the collection view animates its own offset back into range and undoes the pin or anchor. It cost 14 to 20 minutes in two builds. For large removals use `reloadData` instead of an animated delete. [F1.4] [F2.2]
- Text size: S3 and SC3 returned the anchored offset from the same method after refreshing heights, the fastest text-size rows measured. [F1.5 S3, SC3]
- Anchor from fresh attributes, not stale cached ones: S3 lost 14 minutes to an anchor read from attributes cached before the update. [F2.2 S3]

## Jump

`applySnapshotUsingReloadData` (or `reloadData`) with a page around the target, then the offset from the layout's own frame for that item, rounded to pixels. `scrollToItem(at:at:animated:)` alone landed one row off when the frames above were not final. [F1.3 S3, O3] [F2.2]
