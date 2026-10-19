# Performance

Smooth UIKit comes down to one rule: each frame the main thread has a few milliseconds (about 8 ms at 120 Hz, 16 ms at 60 Hz) to run your code, lay out and commit. Anything that does not have to happen on the main thread, or does not have to happen now, should not.

## Where time goes in a scrolling list

1. Cell configuration (your registration handler). Must be a lookup plus property sets.
2. Auto Layout for self-sizing cells. Grows with constraint count and with ambiguity.
3. Text layout. Long attributed strings are measured every time a cell is sized.
4. Image decoding. A compressed JPEG or PNG is decoded the first time it is drawn, on the main thread, unless you decode it earlier.
5. Rendering in the render server: offscreen passes from masks, shadows without paths, group opacity and blur.

## Images

Decode and downscale before the image reaches an image view.

```swift
actor ThumbnailLoader {
    private var cache: [URL: UIImage] = [:]
    private var inFlight: [URL: Task<UIImage?, Never>] = [:]

    func image(for url: URL, size: CGSize, scale: CGFloat) async -> UIImage? {
        if let cached = cache[url] { return cached }
        if let task = inFlight[url] { return await task.value }
        let task = Task<UIImage?, Never> {
            guard let (data, _) = try? await URLSession.shared.data(from: url),
                  let image = UIImage(data: data) else { return nil }
            let pixelSize = CGSize(width: size.width * scale, height: size.height * scale)
            return await image.byPreparingThumbnail(ofSize: pixelSize)
        }
        inFlight[url] = task
        let result = await task.value
        inFlight[url] = nil
        cache[url] = result
        return result
    }
}
```

In the cell:

```swift
final class PhotoCell: UICollectionViewCell {
    let imageView = UIImageView()
    private var loadTask: Task<Void, Never>?
    private(set) var representedID: ItemID?

    func show(_ item: Photo, loader: ThumbnailLoader) {
        representedID = item.id
        imageView.image = nil
        loadTask?.cancel()
        let size = bounds.size
        let scale = traitCollection.displayScale
        loadTask = Task { [weak self] in
            let image = await loader.image(for: item.url, size: size, scale: scale)
            guard let self, !Task.isCancelled, self.representedID == item.id else { return }
            self.imageView.image = image
        }
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        loadTask?.cancel()
        imageView.image = nil
    }
}
```

- `byPreparingThumbnail(ofSize:)` decodes and scales in one step; `byPreparingForDisplay()` decodes at full size. Both are async and run off the main thread. Completion-handler forms exist (`prepareThumbnail(of:completionHandler:)`, `prepareForDisplay(completionHandler:)`) and synchronous forms (`preparingThumbnail(of:)`, `preparingForDisplay()`) for use on a background queue.
- Thumbnail size is in pixels, so multiply points by `traitCollection.displayScale`. Never use `UIScreen.main.scale`.
- For large files on disk, `CGImageSourceCreateThumbnailAtIndex` with `kCGImageSourceThumbnailMaxPixelSize` and `kCGImageSourceCreateThumbnailFromImageAlways` avoids loading the whole image into memory first.
- Cache the decoded, downscaled image, not the data. Use `NSCache` (it evicts under memory pressure) or an actor-wrapped dictionary with a size limit; a plain unbounded dictionary as in the sketch above is fine only for a bounded set.
- Check the cell still represents the same item before assigning, because cells are reused while the load is in flight.
- SF Symbols and asset catalog images are already optimised; do not prepare them.

## Prefetching

Set `collectionView.prefetchDataSource` and implement `UICollectionViewDataSourcePrefetching`:

```swift
func collectionView(_ collectionView: UICollectionView, prefetchItemsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
        guard let id = dataSource.itemIdentifier(for: indexPath), let photo = store[id] else { continue }
        prefetchTasks[id] = Task { _ = await loader.image(for: photo.url, size: thumbnailSize, scale: scale) }
    }
}

func collectionView(_ collectionView: UICollectionView, cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
    for indexPath in indexPaths {
        guard let id = dataSource.itemIdentifier(for: indexPath) else { continue }
        prefetchTasks.removeValue(forKey: id)?.cancel()
    }
}
```

- Prefetching is a hint. The cell must still work when the image is not ready.
- Prefetch also works for data pages: when index paths near the end are prefetched, request the next page.
- Cell prefetching (`isPrefetchingEnabled`, on by default) prepares cells ahead of time. Leave it on; it is why cell configuration must not depend on being visible.

## Text

- Reuse formatters. `DateFormatter`, `NumberFormatter`, `ListFormatter` and `RelativeDateTimeFormatter` are expensive to create; keep one per format, or use the `formatted()` APIs, which cache internally.
- Build attributed strings once per model change, not per configuration. Store them with the model or in a cache keyed by ID and content size category.
- `UILabel` is cheaper than `UITextView` for display. Use `UITextView` only when you need selection, links with custom handling, or editing.
- Very long text in self-sizing cells: cap with `numberOfLines` and a "more" control instead of measuring thousands of lines.

## Layout cost

- Fewer views and fewer constraints per cell is faster. Prefer one `UIStackView` or one content configuration over many nested stacks. Stack views are convenient but add constraints; in cells that appear hundreds of times, flat constraints can be cheaper.
- Avoid changing constraints in `layoutSubviews`. Change constants, not constraints, and do it in `updateConstraints` or when the data changes.
- Activate constraints in batches with `NSLayoutConstraint.activate(_:)`.
- An ambiguous layout is a slow layout: the engine works harder and the result can change between passes. Fix every "Unable to simultaneously satisfy constraints" log.

## Rendering

- Rounded corners: `layer.cornerRadius` with `layer.cornerCurve = .continuous` is cheap on its own. Adding `masksToBounds = true` over content that changes forces an offscreen pass; prefer rounding an image view only, or use a background configuration with a corner radius.
- Shadows: always set `layer.shadowPath` (and update it in `layoutSubviews`) or the shadow is computed from the layer's alpha every frame.
- `shouldRasterize` helps only for complex content that is static and drawn at the same size; on cells that change, it costs more than it saves. Measure before using it.
- Avoid `alpha < 1` on container views with many subviews (group opacity); fade the specific subviews or use colors with alpha.
- Visual effect views (blur) are expensive; avoid them inside every cell.

## Animation and frame callbacks

- Use `UIView.animate`, `UIViewPropertyAnimator` and spring animations; they run in the render server and survive main-thread stalls better than manual frame updates.
- For per-frame work tied to display refresh, use `UIUpdateLink` (iOS 18+) on a view; it adapts to the display's refresh rate and can be paused. `CADisplayLink` remains for older targets. Neither is a reason to drive layout manually; prefer system animation.

## Measuring

Always measure on a real device in a Release build. The simulator and Debug builds mislead.

Instruments templates to reach for:

- **Animation Hitches**: shows each hitch, its duration and whether the commit or the render phase was late. Start here for "scrolling feels janky".
- **Time Profiler**: what the main thread is doing. Filter to the main thread, invert the call tree, hide system libraries, and look for your code in cell configuration and layout.
- **Allocations** and **Leaks**: growth while scrolling back and forth over the same content means a cache or reuse bug.
- **Swift Concurrency**: shows tasks and actor hops, useful when work you expected off main is on it.
- **os_signpost**: wrap your own phases with `OSSignposter` intervals so they appear in Instruments next to the system tracks.

Automate the check with XCTest performance tests:

```swift
func testScrollHitches() {
    let app = XCUIApplication()
    app.launch()
    let list = app.collectionViews.firstMatch
    measure(metrics: [XCTOSSignpostMetric.scrollingAndDecelerationMetric]) {
        list.swipeUp(velocity: .fast)
    }
}
```

This reports hitch time ratio and frame rate during scrolling, so a regression shows up in CI rather than in a user's hand. Keep the same device and data set between runs, and compare medians.

Also worth checking:

- The Xcode Organizer's hang reports and scroll hitch rate metrics for field data from real users.
- Main Thread Checker (on by default in Debug) for UIKit calls off main.
- `-UIViewLayoutFeedbackLoopDebuggingThreshold` launch argument when layout seems to loop.
