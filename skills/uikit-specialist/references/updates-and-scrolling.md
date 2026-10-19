# Updates and scrolling

Most visible list bugs are update bugs: the list jumps when older content loads, a new message does not scroll into view, a cell flashes because it was reloaded, or an update during a drag stutters. This file covers how to change a list without the user noticing anything but the change.

## Choose the right update

| Change | Do |
|---|---|
| Item content changed, same identity | `snapshot.reconfigureItems(ids)` then apply |
| Cell type for an item changed | `snapshot.reloadItems(ids)` then apply |
| Items added, removed or moved | build or edit a snapshot, apply with animation if the user caused it |
| Everything replaced (new account, new query) | apply without animation, or `applySnapshotUsingReloadData(_:)` |
| A cell's own height changed (expanded text) | `invalidateIntrinsicContentSize()` inside the cell with `selfSizingInvalidation` on, or reconfigure |
| Only a visual effect (badge pulse) | change the visible cell directly through `collectionView.cellForItem(at:)` and leave the snapshot alone |

Coalesce bursts. If messages arrive many per second, collect changes and apply one snapshot per frame or per short interval rather than one per message. Each apply diffs the whole snapshot, so frequent tiny applies cost more than one bigger one.

Animate changes the user caused (sending, deleting, expanding). Do not animate changes that happen far from the visible area or during a fast scroll; the animation work is wasted and can cause hitches.

## Keeping the position when content is inserted above

When you insert items above what the user is looking at (loading older messages, a new item at the top of a feed), the content grows above the viewport and everything visible moves down. The fix is to measure where an anchor item is before the update and put it back after.

Anchor on a visible item, not on `contentSize`. With estimated sizes, the content size changes as cells are measured, so a "add the size delta to the offset" approach drifts.

```swift
@MainActor
func prependPreservingPosition(_ olderIDs: [ItemID]) async {
    // 1. Pick the anchor: the first fully visible item, and its distance from the top of the visible area.
    let visibleTop = collectionView.contentOffset.y + collectionView.adjustedContentInset.top
    let anchor = collectionView.indexPathsForVisibleItems
        .sorted()
        .lazy
        .compactMap { indexPath -> (ItemID, CGFloat)? in
            guard let attributes = self.collectionView.layoutAttributesForItem(at: indexPath),
                  attributes.frame.minY >= visibleTop,
                  let id = self.dataSource.itemIdentifier(for: indexPath) else { return nil }
            return (id, attributes.frame.minY - visibleTop)
        }
        .first

    // 2. Apply without animation.
    var snapshot = dataSource.snapshot()
    if let first = snapshot.itemIdentifiers(inSection: .messages).first {
        snapshot.insertItems(olderIDs, beforeItem: first)
    } else {
        snapshot.appendItems(olderIDs, toSection: .messages)
    }
    await dataSource.apply(snapshot, animatingDifferences: false)

    // 3. Put the anchor back where it was.
    guard let (id, distance) = anchor,
          let indexPath = dataSource.indexPath(for: id) else { return }
    collectionView.layoutIfNeeded()
    guard let attributes = collectionView.layoutAttributesForItem(at: indexPath) else { return }
    let newOffset = attributes.frame.minY - distance - collectionView.adjustedContentInset.top
    collectionView.setContentOffset(CGPoint(x: 0, y: newOffset), animated: false)
}
```

Notes:

- Do the measure, apply and restore in one main-actor turn with no animation, so no frame is drawn in between.
- If the user is dragging or decelerating (`isDragging`, `isDecelerating`), setting the offset stops the scroll. Either queue the prepend until the scroll ends, or keep the scroll alive by adjusting the offset by the anchor's movement in `scrollViewDidScroll` rather than calling `setContentOffset`.
- Trigger loading of older content before the user reaches the very top (for example when the first visible index is within a screen of the start, or through prefetching), so the insert happens off screen.
- With estimated sizes, the inserted cells above the anchor are measured lazily. That is fine for this approach: only the anchor's final frame matters.

## Sticking to the bottom (chat and logs)

Keep the natural top-to-bottom order. Track whether the user is at the bottom, and only follow new content when they are.

```swift
var isPinnedToBottom: Bool {
    let bottomEdge = collectionView.contentOffset.y + collectionView.bounds.height - collectionView.adjustedContentInset.bottom
    return bottomEdge >= collectionView.contentSize.height - 24
}

@MainActor
func append(_ newIDs: [ItemID]) async {
    let follow = isPinnedToBottom
    var snapshot = dataSource.snapshot()
    snapshot.appendItems(newIDs, toSection: .messages)
    await dataSource.apply(snapshot, animatingDifferences: follow)
    if follow, let last = newIDs.last, let indexPath = dataSource.indexPath(for: last) {
        collectionView.scrollToItem(at: indexPath, at: .bottom, animated: true)
    }
}
```

- Read `isPinnedToBottom` before the update; after it the content size has already changed.
- When the keyboard appears, the visible area shrinks from the bottom. If the user was pinned, keep them pinned: see the keyboard section in `layout.md`.
- For the first load of a long conversation, apply without animation, call `layoutIfNeeded()`, then scroll to the last item without animation, before the view appears (in `viewIsAppearing(_:)`).
- If the user is not pinned, show a "new messages" button instead of moving them.

## Restoring a position across launches or reloads

Save the identifier of the top visible item and its offset from the top, not a raw `contentOffset`. After the next load, find that identifier's index path and use the same restore step as the prepend above. Raw offsets are wrong as soon as anything above changes size, including after a Dynamic Type change.

## Scrolling programmatically

- `scrollToItem(at:at:animated:)` works with estimated sizes but the final position can be off by the difference between estimate and real size when the target is far away. For an exact landing, scroll without animation, `layoutIfNeeded()`, then scroll again to the same item.
- Do not scroll in `viewDidLoad`; the bounds are not final. Use `viewIsAppearing(_:)` (available back to iOS 13) or after the first layout.
- Do not call `scrollToItem` in the same turn as an animated `apply`; await the apply first.

## Scroll delegate hygiene

`scrollViewDidScroll(_:)` runs every frame while scrolling. Keep it to comparisons and cheap property changes. Never apply snapshots, start layout passes, or do allocation-heavy work there. For "load more" triggers, prefer prefetching (see `performance.md`) or a check that runs at most once per threshold crossing.

## Rotations and size changes

When the container width changes, cells resize and every height can change, so the visible item moves. Save the anchor in `viewWillTransition(to:with:)` and restore it in the coordinator's animation or completion block, using the same anchor method as above.
