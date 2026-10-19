# Swift concurrency with UIKit

UIKit is a main-thread framework. `UIView`, `UIViewController`, `UIApplication` and most UIKit protocols are annotated `@MainActor`, so in Swift 6 the compiler enforces what used to be a runtime rule. Work with that: keep UI on the main actor, move work off it, and make the boundary explicit.

## The shape that works

```swift
final class FeedViewController: UIViewController {
    private let service: FeedService            // an actor or a Sendable type
    private var loadTask: Task<Void, Never>?

    override func viewIsAppearing(_ animated: Bool) {
        super.viewIsAppearing(animated)
        guard loadTask == nil else { return }
        loadTask = Task { [weak self] in
            await self?.load()
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
        loadTask = nil
    }

    private func load() async {
        do {
            let page = try await service.firstPage()   // runs off the main actor
            try Task.checkCancellation()
            apply(page)                                 // back on the main actor
        } catch is CancellationError {
            // the screen went away; nothing to do
        } catch {
            showError(error)
        }
    }
}
```

- A `Task { }` created inside a main-actor type inherits the main actor. The `await` on a call into another actor or a nonisolated async function suspends the main actor while the work runs elsewhere, then resumes on main. There is no need for `DispatchQueue.main.async` or `MainActor.run` here.
- Store the task and cancel it when the result is no longer wanted. A `Task` is not tied to the view controller's lifetime; it keeps running and keeps anything it captures strongly alive until it finishes.
- Capture `[weak self]` in tasks that may outlive the screen (network, long streams). For a task that ends quickly and must finish (saving a draft), a strong capture is fine and clearer.
- Check cancellation after each suspension point that precedes UI work (`Task.isCancelled` or `try Task.checkCancellation()`).

## Moving work off the main actor

`async` alone does not mean "background". A plain `async` method on a main-actor class runs on the main actor. To move CPU work off main:

- Put it in an actor (`actor ImageCache`, `actor Database`) or a type that is not main-actor isolated.
- Or mark a function `nonisolated` or `@concurrent` (Swift 6.2 and later, where nonisolated async functions run on the caller's actor by default unless marked `@concurrent`). Check the project's Swift version and its default isolation setting (the "Default Actor Isolation" build setting can make everything `@MainActor` by default) before relying on either rule.
- Use `Task.detached` rarely, only when you explicitly need to drop the current actor and priority. Prefer an actor or `@concurrent` function.

Pass `Sendable` values across the boundary: structs of value types, IDs, `Data`, immutable final classes. Do not send `UIImage` subclasses you mutate, view references or non-Sendable model classes into a background actor. `UIImage` itself is `Sendable` and safe to pass.

## Streams and long-lived updates

Consume an `AsyncSequence` (a socket, a database observation, `NotificationCenter.default.notifications(named:)`) in a stored task and apply changes in batches:

```swift
streamTask = Task { [weak self] in
    guard let batches = self?.service.updateBatches() else { return }   // AsyncStream<[Change]>
    for await batch in batches {
        guard let self else { return }
        self.applyChanges(batch)
    }
}
```

One snapshot apply per batch keeps the main actor free for scrolling. Buffer in the producer (an actor that collects changes and yields an array every 50 to 100 ms or every N items), or use a time-based chunking helper such as the ones in swift-async-algorithms. Capturing `self` weakly and re-checking it per element lets the screen go away mid-stream.

## Delegates, callbacks and older APIs

- Delegate protocols from UIKit are main-actor isolated; your conformances run on main.
- Callbacks from non-UIKit frameworks may arrive on any queue. If the callback API is not annotated, hop explicitly with `Task { @MainActor in ... }` or mark the handler `@MainActor`. Wrap one-shot callback APIs with `withCheckedContinuation` once, in the service layer, rather than in view controllers.
- `@preconcurrency import` silences warnings for modules that are not annotated yet; prefer it to `@unchecked Sendable` on your own types.
- `nonisolated` on a main-actor class's method lets it be called from anywhere, but it then cannot touch views or main-actor state.

## Observation

On iOS 26 and later UIKit can track `@Observable` model properties read in `layoutSubviews()`, `updateProperties()`, `updateConfiguration(using:)` and similar methods, and re-run them when those properties change (on iOS 18 this needs the `UIObservationTrackingEnabled` Info.plist key). For iOS 18 targets, keep an explicit update path: the model publishes changes (an `AsyncStream`, a delegate, or a closure), and the controller reconfigures items or updates views.

## Things that cause hangs or crashes

- `DispatchQueue.main.sync` from the main thread deadlocks. From a background thread it blocks that thread until main is free; avoid it.
- Semaphores or `DispatchGroup.wait()` to make async code synchronous on main freeze the UI and can deadlock with the main actor.
- Touching a view from a background queue: the Main Thread Checker catches it in Debug; Swift 6 strict concurrency catches most of it at compile time. Turn on complete concurrency checking in new code.
- Large `@MainActor` model classes that do parsing or sorting inside their methods: the work runs on main. Move it to an actor and hand back results.
- Firing many tasks per cell (one per configuration) without cancelling: they pile up when scrolling fast. Store one task per cell and cancel on reuse.
