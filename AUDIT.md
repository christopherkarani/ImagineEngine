# ImagineEngine — Production Readiness Audit

**Date:** 2026-03-07
**Auditor:** Principal Engineer (adversarial review)
**Codebase:** ImagineEngine v0.7.0
**Revision:** `claude/production-readiness-audit-GBG4b`

---

## 1. Executive Summary

### Overall Production Readiness Score: **4 / 10**

The engine is a well-structured, pedagogically clear game engine with a clean public API and a coherent event-driven design. However, beneath the surface are several confirmed correctness bugs, a memory leak that prevents host objects from ever being deallocated, a data race on texture loading, and pervasive use of deprecated Swift language features. The codebase appears to have been written for Swift 4 and has not been meaningfully updated since.

It is **not safe to ship production games on top of this library without resolving the blockers below**.

---

### Top 5 Critical Risks

| # | Risk | Severity |
|---|------|----------|
| 1 | `CADisplayLink` retain cycle on iOS/tvOS — `Game` objects never deallocated | **Blocker** |
| 2 | `Scene.deactivate()` skips blocks — layer orphaning and broken lifecycle | **Blocker** |
| 3 | `TextureManager.load()` recursive scale fallback drops the actor-level name prefix | **Blocker** |
| 4 | `TextureManager.preloadTexture` creates a data race on the texture cache | **Blocker** |
| 5 | `Timeline.activate()` does not clear `unsortedUpdatables` — events fire twice on re-use | **Blocker** |

---

### Release Blockers

All five risks above are release blockers. Items #1–5 in Section 2 are independently sufficient to cause production incidents.

---

## 2. Correctness Issues

### 2.1 [Blocker] `Scene.deactivate()` Silently Skips All Blocks

**File:** `Sources/Core/API/Scene.swift:209–223`

```swift
internal func deactivate() {
    layer.removeFromSuperlayer()
    camera.deactivate()
    timeline.deactivate()
    pluginManager.deactivate()

    for actor in grid.actors { actor.deactivate() }
    for label in grid.labels { label.deactivate() }
    // ← blocks are NEVER deactivated
}
```

`Block.deactivate()` stops the block's `ActionManager` and removes its layer. Neither of these happens when a scene is swapped out via `game.scene = newScene`. Consequences:

- Block `ActionManager`s retain a stale `Scene` reference (it is never `nil`-ed by `deactivate()`) and continue responding to `requestUpdate` calls against a dead timeline.
- `block.scene` is never `nil`-ed by the lifecycle path, leaving actors with dangling scene references.
- Compare with `Scene.reset()` which correctly iterates all three collections: actors, blocks, and labels.

**Root cause:** `deactivate()` has always omitted blocks; the bug was not caught because tests exercise `reset()` (which is correct) and there are no tests for scene swapping with live blocks.

---

### 2.2 [Blocker] `TextureManager.load()` Recursive Fallback Drops the Actor-Level Name Prefix

**File:** `Sources/Core/Internal/TextureManager.swift:96`

```swift
internal func load(_ texture: Texture, namePrefix additionalNamePrefix: String?, scale: Int?) -> LoadedTexture? {
    let scale = scale ?? defaultScale
    var name = texture.name

    if let prefix = additionalNamePrefix { name = "\(prefix)\(name)" }
    if let prefix = namePrefix          { name = "\(prefix)\(name)" }

    let cacheKey = "\(name).\(format.rawValue)"
    // ...

    // Falls back to a lower scale on miss:
    return load(texture, namePrefix: namePrefix, scale: scale - 1)
    //                              ^^^^^^^^^^^
    //                              self.namePrefix, NOT additionalNamePrefix
}
```

When a higher-resolution asset is missing and the engine falls back to a lower scale, it reconstructs the name from `self.namePrefix` (the manager-global prefix), **discarding** `additionalNamePrefix` (the actor-level prefix).

**Concrete failure scenario:**

An `Actor` has `textureNamePrefix = "Wizard-"` and the `TextureManager` has no global `namePrefix`. The engine tries to load `Wizard-Fireball@2x.png` (not found), then falls back with:

```swift
load(texture, namePrefix: nil /*self.namePrefix*/, scale: 1)
```

The fallback looks for `Fireball.png` — the prefix is lost. The wrong texture (or no texture) is loaded silently.

When a global `namePrefix` exists on the manager and an actor-level prefix is also set, the fallback double-applies the global prefix, constructing `"Global-Global-textureName"` — a file that can never exist.

---

### 2.3 [Blocker] Timeline Does Not Clear `unsortedUpdatables` After Activation

**File:** `Sources/Core/API/Timeline.swift:113–119`

```swift
func activate(in game: Game) {
    lastUpdateTime = game.currentTime

    for (updatable, delay) in unsortedUpdatables {
        insert(updatable, at: game.currentTime + delay)
    }
    // unsortedUpdatables is never cleared
}
```

If a `Timeline` is deactivated and later re-activated (e.g., a scene is temporarily removed then re-added), every event that was pending at first activation is re-inserted into the BST. Those events fire twice.

There is no test for this path because `Scene.reset()` creates a brand-new `Timeline`, avoiding the bug. But the API contract for `Activatable` is that `activate` / `deactivate` can be called multiple times, and `Timeline` violates it.

---

### 2.4 [Blocker] `TextureManager.preloadTexture` Data Race on the Cache

**File:** `Sources/Core/Internal/TextureManager.swift:39–46`

```swift
public func preloadTexture(named name: String,
                           scale: Int? = nil,
                           format: TextureFormat? = nil,
                           onQueue queue: DispatchQueue = .main) {
    queue.async {
        _ = self.load(Texture(name: name, format: format), namePrefix: nil, scale: scale)
    }
}
```

The parameter `onQueue` has a default of `.main` but the API explicitly invites callers to pass any queue. `self.load()` reads and writes `self.cache` (a `[String: LoadedTexture]` dictionary). There is no locking or actor isolation. Calling `preloadTexture(onQueue: .global())` while the game loop accesses the same cache from the main thread is an undetected data race.

Even on `.main` this is fragile: there is no documented contract that `.load()` is main-thread-only, so future callers and contributors cannot rely on this.

---

### 2.5 [Blocker — Memory] `CADisplayLink` Retain Cycle on iOS/tvOS

**File:** `Sources/Core/Internal/DisplayLink-iOS+tvOS.swift`

```swift
internal final class DisplayLink: DisplayLinkProtocol {
    private lazy var link = makeLink()

    deinit {
        link.remove(from: .main, forMode: .commonModes)
    }

    private func makeLink() -> CADisplayLink {
        return CADisplayLink(target: self, selector: #selector(screenDidRender))
    }
}
```

`CADisplayLink` strongly retains its `target`. This creates a cycle:

```
DisplayLink → (lazy var link) CADisplayLink → (target) DisplayLink
```

`deinit` can never be reached: the CADisplayLink holds the strong reference that prevents deallocation. The only way to break the cycle is to call `link.remove(from:forMode:)` explicitly before releasing the `Game` — but there is no public API to do this, and no `Game.teardown()` method.

**Consequence:** Every `Game` instance created in an iOS/tvOS app is permanently leaked. In a typical game where scenes change frequently, each scene change triggers `Game.sceneDidChange()` but the `Game` itself lives forever.

**Common fix:** Use a weak-target proxy object as `CADisplayLink`'s target, as recommended by Apple.

---

### 2.6 [Major] macOS `CVDisplayLink` Potential Use-After-Free

**File:** `Sources/Core/Internal/DisplayLink-macOS.swift:30–33`

```swift
let opaquePointerToSelf = UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque())
CVDisplayLinkSetOutputCallback(link, _imageEngineDisplayLinkCallback, opaquePointerToSelf)
```

`passUnretained` does not increment the retain count. The callback is dispatched from a background thread:

```swift
@objc func screenDidRender() {
    DispatchQueue.main.async(execute: callback)
}
```

`CVDisplayLinkStop` in `deinit` is documented to be synchronous and wait for the current callback to finish, which prevents the most obvious race. However, between the `DispatchQueue.main.async` call in a background-thread callback and the actual execution on the main thread, the `DisplayLink` object could be deallocated (if `deinit` has already been called and the `async` block retains only `self.callback`, a closure that captures `self` weakly). The dispatched block captures `callback` (a `var`, by value), so `self` is not retained by the block. This is safe in this specific case, but the `unsafeBitCast` is a footgun that is one refactor away from a crash.

---

### 2.7 [Major] `isCollisionDetectionActive` Is Set But Never Cleared

**File:** `Sources/Core/API/ActorEventCollection.swift:29–33`

```swift
public func collided(with actor: Actor) -> Event<Actor, Actor> {
    object?.isCollisionDetectionActive = true
    actor.isCollisionDetectionActive = true
    return makeEvent(withSubject: actor)
}
```

`isCollisionDetectionActive` is set to `true` the moment a caller accesses the `collided(with:)` event — even if no observer is ever added. There is no path that resets it to `false`. This means full `O(n²)` pair-wise actor collision detection is permanently enabled for any actor whose event collection is queried, even in read-only contexts.

A defensive pattern like `let e = actor.events.collided(with: enemy)` (to store for later observation) permanently alters the performance profile of both actors. This flag should be tied to whether active observers exist, not to event access.

---

### 2.8 [Major] `SpriteSheet.frame(at:)` Division By Zero

**File:** `Sources/Core/API/Animation.swift:149–150`

```swift
case .spriteSheet(let sheet):
    let framesPerRow = sheet.frameCount / sheet.rowCount  // integer division
    let row = index / framesPerRow                        // crashes if framesPerRow == 0
```

`SpriteSheet.init` accepts `rowCount: Int = 1`. A caller can pass `rowCount: 0` or `frameCount: 0` with no validation. Both cases cause division by zero in `frame(at:)`, which crashes the game loop with an unrecoverable `EXC_ARITHMETIC`. There is no precondition guard or documented contract that `rowCount > 0` and `frameCount >= rowCount`.

---

### 2.9 [Major] `RepeatAction` Does Not Propagate Cancellation to Its Inner Action

**File:** `Sources/Core/API/RepeatAction.swift:20–24`

```swift
internal override func update(for object: Object, currentTime: TimeInterval) -> UpdateOutcome {
    let wrapper = actionWrapper.get(orSet: ActionWrapper(action: action, object: object))
    _ = wrapper.update(currentTime: currentTime)
    return .continueAfter(0)
}
```

When the outer `RepeatAction`'s token is cancelled, the outer `ActionWrapper` stops updating `RepeatAction`. However, the inner `action.token` is a separate `ActionToken` that is never cancelled. If the inner `Action` overrides `cancel(for:)` to perform meaningful cleanup (e.g., restore state), that method is never called. The default `Action.cancel(for:)` is a no-op, so built-in actions are unaffected, but this is a silent trap for all custom `Action` subclasses.

---

### 2.10 [Minor] `Game.sceneDidChange()` Reconnects `displayLink.callback` on Every Scene Change

**File:** `Sources/Core/API/Game.swift:74–89`

```swift
private func sceneDidChange(from previousScene: Scene?) {
    previousScene?.deactivate()
    if isActive { scene.activate(in: self) }

    displayLink.callback = { [weak self] in
        guard let `self` = self else { return }
        self.currentTime = self.dateProvider().timeIntervalSinceReferenceDate
        self.scene.timeline.update(currentTime: self.currentTime)
    }
}
```

The `displayLink.callback` closure captures `self` weakly and accesses `self.scene.timeline` on each frame. The game loop only updates `scene.timeline` — it does not update scene plugins, actors' action managers, or any other subsystems directly. All updates flow through the timeline via `requestUpdate`. This design is correct but non-obvious. Adding a system that requires per-frame updates outside the timeline will silently not receive them.

---

### 2.11 [Minor] Silent Failure When Adding an Actor Already in Another Scene

**File:** `Sources/Core/API/Scene.swift:108–116`

```swift
public func add(_ actor: Actor) {
    actor.scene = self    // overwrites the previous scene reference
    grid.add(actor, in: self)
    actor.addLayer(to: layer)
    game.map(actor.activate)
    events.actorAdded.trigger(with: actor)
}
```

`Grid.add(_:in:)` is a no-op if the actor is already in `grid.actors`. But if the actor was in a *different* scene, `actor.scene` is overwritten before any cleanup happens in the original scene. The original scene's grid, event observers, and layer hierarchy still reference the actor. This leads to phantom collision events and double-rendering.

---

## 3. Architecture & Design Gaps

### 3.1 [Major] No Swift Package Manager Support

The only distribution mechanism is CocoaPods (`.podspec`). As of Swift 5.3, SPM is the canonical first-party package manager for all Apple platforms and is required for integration into Xcode projects without CocoaPods. There is no `Package.swift`. This is a material barrier to adoption and is expected by modern developers.

### 3.2 [Major] Game Loop Only Updates the Timeline

The display link callback updates `scene.timeline`. Everything else — plugin updates, action execution — must flow through the timeline via `requestUpdate()`. This is an implicit contract that is not documented and creates fragile code paths. A plugin that needs per-frame updates must schedule itself via the timeline rather than having an explicit `update(deltaTime:)` method. There is no `Plugin.update(deltaTime:)` override point.

### 3.3 [Minor] Pluggable Protocol Has No Method-Based Extension Points

The `Plugin` protocol only provides `activate(for:in:)` and `deactivate()`. Any per-frame logic requires the plugin to schedule work on the scene's timeline inside `activate`. This is non-obvious and forces timeline familiarity on all plugin authors. A standard `update(currentTime:)` callback would be simpler and more explicit.

### 3.4 [Minor] `EventCollection.makeEvent` Uses `#function` as Discriminator

**File:** `Sources/Core/API/EventCollection.swift:52–54`

```swift
public func makeEvent<Subject>(named name: StaticString = #function) -> Event<Object, Subject> {
    return makeEvent(named: name, withSubjectIdentifier: "Void")
}
```

Two computed properties with the same name (e.g., in different protocol extensions) will silently share the same underlying event. This is a latent naming collision hazard with no compile-time protection.

### 3.5 [Minor] `Block.size` Is Immutable After Initialization

`Block.size` is declared `public let size: Size`. There is no ability to resize a block post-creation. This is an intentional constraint (blocks are static), but it is not documented, and there is no API to replace a block with a different-sized version. Developers are left with the workaround of removing and re-adding a block, which loses action state.

### 3.6 [Minor] `Camera.update()` Has Early Return Bug Under Constraint

**File:** `Sources/Core/API/Camera.swift:102–126`

```swift
if constrainedToScene {
    if sceneSize.width >= size.width {
        guard newRect.minX >= 0 else {
            position.x = size.width / 2
            return       // ← early return, layer.frame not updated
        }
        guard newRect.maxX <= sceneSize.width else {
            position.x = sceneSize.width - size.width / 2
            return       // ← early return, layer.frame not updated
        }
    }
    if sceneSize.height >= size.height {
        // same pattern
    }
}

layer.frame.origin = Point(...)  // only reached if no constraint triggered
```

When a constraint fires and `position` is corrected, the function returns before updating `layer.frame.origin`. The correction of `position` triggers a `didSet` which calls `update()` again recursively — but only because `positionDidChange` triggers `update()` through the property observer. This second call must then succeed (the constraint is now satisfied), updating the layer. While this appears to work in tests, it relies on the property observer calling `update()` again, making the behavior non-obvious and the recursion depth unbounded in edge cases (e.g., floating-point oscillation at the constraint boundary).

---

## 4. Concurrency & Safety

### 4.1 [Blocker] `TextureManager.cache` — Unguarded Cross-Thread Access

Covered in §2.4. The cache dictionary has no synchronization primitive. Callers can legally provide a non-main `DispatchQueue` to `preloadTexture`, producing a data race with main-thread rendering.

### 4.2 [Major] CADisplayLink Retain Cycle — iOS/tvOS

Covered in §2.5. Not just a memory issue; it also means `deinit` cleanup never runs, so `CADisplayLink` is never invalidated properly. On deallocated view hierarchies, the callback will attempt to access a deallocated scene.

### 4.3 [Major] `CVDisplayLink` Callback on Background Thread — macOS

**File:** `Sources/Core/Internal/DisplayLink-macOS.swift:36–38`

```swift
@objc func screenDidRender() {
    DispatchQueue.main.async(execute: callback)
}
```

`CVDisplayLink` fires its callback on an internal background thread. `screenDidRender` then dispatches `callback` asynchronously to the main queue. This introduces up to one frame of latency (the callback runs in the *next* main-thread run loop iteration, not during the display link's timing window). More critically, if the game is producing many frames that back up (e.g., main thread is stalled), multiple `callback` invocations queue up and then fire in rapid succession — causing erratic game speed.

### 4.4 [Minor] `UpdatableCollection` Mutation During Iteration

**File:** `Sources/Core/API/Timeline.swift:85–97`

```swift
for updatable in immediateUpdatables {
    let outcome = updatable.update(currentTime: currentTime)
    switch outcome {
    case .continueAfter(let delay):
        if delay > 0 {
            immediateUpdatables.remove(updatable)  // mutated during iteration
            schedule(updatable, delay: delay)
        }
    case .finished:
        immediateUpdatables.remove(updatable)      // mutated during iteration
    }
}
```

`UpdatableCollection` is a value-type struct but is stored on a class (`Timeline`), so mutation affects the live instance. The iterator holds a reference to the first `Node` (a class). Removal updates doubly-linked list pointers but does not nil the removed node's `next` pointer, so the iterator can still traverse correctly. This happens to be safe due to implementation details, but it is fragile: any refactor that clears `next` on removal would silently break iteration.

### 4.5 [Minor] `Event.trigger()` Mutates Dictionary During Iteration

**File:** `Sources/Core/API/Event.swift:62–75`

```swift
for (key, closure) in observationClosures {
    switch key {
    case .token(let token):
        guard !token.isCancelled else {
            observationClosures[key] = nil  // mutation during for-in
            continue
        }
    }
    closure(object, subject)
}
```

In Swift, `for (k, v) in dict` iterates over a snapshot (Dictionary's `Indices` is a value type). The mutation `observationClosures[key] = nil` writes to the live property on the class, but the current iteration continues over the snapshot. Cancelled closures are correctly skipped. However, this behavior depends on an undocumented detail of Swift's Dictionary iteration and could change. The canonical pattern is to collect keys for removal and batch-remove after iteration.

---

## 5. Performance Bottlenecks

### 5.1 [Major] `Grid.Index` Hash Function Produces Catastrophic Collisions

**File:** `Sources/Core/Internal/Grid.swift:363`

```swift
var hashValue: Int { return x ^ y }
```

The grid index divides pixel coordinates by 50 (`Grid.Tile.size`). For a typical 1000×600 scene, tile indices range roughly from (0,0) to (20,12). `x ^ y` for small integer pairs has severe collision behavior:

- `(0, 0)`, `(1, 1)`, `(5, 5)`, `(12, 12)` all hash to `0`.
- `(3, 5)` and `(5, 3)` both hash to `6`.
- Symmetric pairs `(x, 0)` and `(0, x)` always collide.

For a 20×12 grid, approximately 50% of all tile lookups land in the same hash bucket, degrading the `tiles` dictionary from O(1) to O(n) per lookup. Collision detection — already called every time any actor moves — runs O(n) dictionary operations. This is a silent quadratic bottleneck in dense scenes.

**Fix:** Use a Cantor pairing or bit-shift combination: `x &* 31 &+ y` or `(x << 16) | (y & 0xFFFF)`.

### 5.2 [Major] Actor Velocity Changes Allocate New Actions on Every Update

**File:** `Sources/Core/API/Actor.swift:244–258`

```swift
private func velocityDidChange(from oldValue: Vector) {
    velocityActionToken?.cancel()
    guard velocity != .zero else { return }

    velocityActionToken = perform(RepeatAction(
        action: MoveAction(vector: velocity, duration: 1)
    ))
}
```

Every call to `actor.velocity = newVelocity` allocates a new `RepeatAction`, a new `MoveAction`, an `ActionToken`, an `ActionWrapper`, an `UpdatableWrapper`, and a `CancellationToken`. For a joystick-driven game with 60fps updates and per-frame velocity adjustments, this is 60 allocations per actor per second, all of which go through the timeline and are eventually deallocated. ARC churn on these objects is measurable.

**Better pattern:** A persistent velocity-update mechanism that mutates position directly on each frame without re-allocating action objects.

### 5.3 [Major] Timeline BST Degenerates Into a Linked List

**File:** `Sources/Core/API/Timeline.swift:151–167`

The `Timeline` uses a hand-rolled unbalanced binary search tree. In the common game pattern of repeatedly scheduling future events (`timeline.after(interval:)` inside repeating actions), nodes are inserted in increasing time order, producing a right-skewed tree. Insertion is O(n) in this case. The discard logic (§3.1 of the code) replaces a discarded node with its `greaterChild`, which maintains the right-skew property. Over time the tree is effectively a linked list.

**Fix:** Use a min-heap (priority queue) for O(log n) insert and O(log n) extract-min, or `SortedArray` for small queues.

### 5.4 [Minor] `Group` Uses String Heap Allocations for Hashing

**File:** `Sources/Core/API/Group.swift:44–47`

```swift
public var hashValue: Int {
    return identifier.hashValue
}
```

`identifier` constructs a `String` (`"name-\(name)"` or `"number-\(number)"`). This string is heap-allocated and hashed on every `Group` comparison. `Group` comparisons occur during every collision detection pass. Switch directly on the enum cases for hashing to avoid the allocation.

### 5.5 [Minor] `TextureManager.load()` String Interpolation on Every Cache Miss Path

Cache keys are constructed via `"\(name).\(format.rawValue)"`. In the hot-path (every frame for animated actors), the cache is hit and the key is computed but discarded. Swift strings are copy-on-write but the interpolation still evaluates. A pre-computed or pointer-based cache key would be faster, though this is low priority compared to other issues.

---

## 6. Security Risks

ImagineEngine has no network stack, no user-generated content path, and no persistence layer. The attack surface is limited to the local bundle.

### 6.1 [Minor] Unvalidated Bundle URLs in `BundleTextureImageLoader`

**File:** `Sources/Core/Internal/BundleTextureImageLoader.swift:24–28`

```swift
guard let url = bundle.url(forResource: imageName, withExtension: format.rawValue) else {
    return nil
}
guard let data = try? Data(contentsOf: url) else { return nil }
```

`imageName` is derived from `texture.name` with optional name prefixes prepended. There is no validation that `imageName` doesn't contain path traversal sequences (e.g., `../../private/`). While `Bundle.url(forResource:withExtension:)` constrains lookup to the bundle's resource directory and is not exploitable for path traversal via this API, the raw `Data(contentsOf: url)` load with a `try?` discards all I/O errors silently. If the URL resolves to a large or malformed file, the engine silently returns `nil` with no diagnostics even in release builds.

### 6.2 [Minor] `TextureManager.errorMode` Errors Suppressed in Release

Error reporting for missing textures is wrapped in `#if DEBUG`. In release builds, missing textures produce `nil` silently, invisible to the developer. This is not a security issue, but it violates the principle of fail-loudly: a misconfigured asset bundle in production ships without any detectable signal.

---

## 7. Testing Review

### 7.1 Missing: Scene Swap With Live Blocks

No test validates that switching `game.scene` while the old scene has live blocks correctly deactivates those blocks. This is the test gap that masks §2.1.

### 7.2 Missing: Timeline Re-Activation After Deactivation

No test exercises `timeline.activate()` being called twice (deactivate + re-activate). This masks the `unsortedUpdatables` accumulation bug in §2.3.

### 7.3 Missing: `TextureManager` Scale Fallback With Actor Prefix

No test exercises `load()` where the initial scale miss triggers the recursive fallback AND an actor-level `namePrefix` is set. This masks §2.2.

### 7.4 Missing: SpriteSheet With `rowCount = 0`

No test validates input sanitization on `SpriteSheet`. The out-of-bounds division in §2.8 is completely untested.

### 7.5 Missing: Concurrent Texture Preload

No test calls `preloadTexture(onQueue: .global())` and verifies cache integrity. The data race in §2.4 is untested.

### 7.6 Missing: Stress and Performance Tests

There are no benchmark tests. The grid hash collision issue (§5.1) and timeline degeneration (§5.3) are invisible without load testing. A scene with 100+ actors in motion would expose both.

### 7.7 Missing: Action Cancellation Under RepeatAction

`ActionTests.testCancellingChainedActions` only tests chain cancellation. No test verifies that cancelling a `RepeatAction` propagates `cancel(for:)` to its inner action — which it does not (§2.9).

### 7.8 Missing: Actor Added to Two Scenes

No test exercises adding the same `Actor` to two different scenes simultaneously (§2.11).

### 7.9 Quality: TODO in Production Test Code

**File:** `Tests/ImagineEngineTests/ActorTests.swift:597`

```swift
// TODO: Needed until we find a better way of figuring out the image scale factor on macOS
var scaled: Size { ... }
```

A `TODO` in test utility code indicates an unresolved platform discrepancy. The macOS `Screen.mainScreenScale` behavior is not properly understood and the workaround may mask incorrect behavior on non-standard display configurations.

### 7.10 Observation: No Failure-Path Testing

No test verifies behavior when `TextureManager` cannot load any texture (e.g., broken bundle). The engine's contract for missing textures (render nothing? crash?) is undefined and untested.

---

## 8. Refactoring Opportunities

### 8.1 [Major] Adopt `Hashable` Via `hash(into:)` (Swift 4.2+)

Every custom `Hashable` conformance in the codebase uses the deprecated `hashValue` property:

| File | Type |
|------|------|
| `Grid.swift` | `Grid.Index` |
| `Event.swift` | `Event.ObservationKey` |
| `Group.swift` | `Group` |
| `Constraint.swift` | `Constraint` |
| `PluginManager.swift` | `TypeIdentifier` |

Swift 4.2 introduced `hash(into:)` and deprecated `hashValue`. Modern Swift synthesizes `Hashable` from `Equatable` for structs/enums with `Hashable` members. Most of these can be reduced to a `@derived` conformance.

### 8.2 [Major] Replace `class` Keyword in Protocol Declarations

```swift
public protocol Movable: class { ... }
```

`class` is deprecated in protocol declarations as of Swift 5.7. Use `AnyObject`:

```swift
public protocol Movable: AnyObject { ... }
```

All five trait protocols (`Movable`, `Rotatable`, `Scalable`, `Fadeable`, `Pluggable`) have this issue.

### 8.3 [Minor] `ActionToken.cancel()` Recursive Precedent Chain Is Unbounded

**File:** `Sources/Core/API/ActionToken.swift:79–82`

```swift
public override func cancel() {
    super.cancel()
    precedingToken?.cancel()
}
```

Cancelling the last token in a long chain calls `cancel()` recursively up the entire chain. A chain of 10,000 actions (theoretically possible in a timeline-heavy game) results in a 10,000-deep call stack. Use an iterative loop.

### 8.4 [Minor] `UpdatableCollection` Should Be a Proper Doubly-Linked List Abstraction

The current struct/class hybrid with a manual linked list and an `allNodes` dictionary for O(1) lookup is correct but has no tests of its own and the `weak previous` pointer creates a subtle implicit assumption (that `allNodes` always holds the strong references). Extract this into a tested, documented data structure.

### 8.5 [Minor] `Scene.add(actor:)` / `Scene.add(block:)` / `Scene.add(label:)` Are Duplicated

The three `add` methods on `Scene` are structurally identical modulo type. The common logic (set scene reference, add to grid, add layer, activate, trigger event) could be unified via a protocol. The current duplication means that when a new step is added to `add()` (as happened with the missing block deactivation in `deactivate()`), it must be applied in three places.

### 8.6 [Minor] `TextureManager.load()` Recursive Scale Fallback Should Be Iterative

The recursive scale fallback descends from `defaultScale` to 1, one step at a time. With `defaultScale = 3` (retina), this is at most 3 levels of recursion — acceptable. But the recursion passes wrong arguments (§2.2) and should be rewritten as an iterative loop in any case.

---

## 9. Summary Table

| # | Location | Issue | Severity |
|---|----------|-------|----------|
| 2.1 | `Scene.swift:209` | `deactivate()` skips blocks | **Blocker** |
| 2.2 | `TextureManager.swift:96` | Recursive fallback drops actor name prefix | **Blocker** |
| 2.3 | `Timeline.swift:113` | `unsortedUpdatables` not cleared on activate | **Blocker** |
| 2.4 | `TextureManager.swift:42` | Data race on cache from non-main queue | **Blocker** |
| 2.5 | `DisplayLink-iOS+tvOS.swift` | CADisplayLink retain cycle leaks `Game` | **Blocker** |
| 2.6 | `DisplayLink-macOS.swift:30` | `unsafeBitCast` on background thread | **Major** |
| 2.7 | `ActorEventCollection.swift:29` | `isCollisionDetectionActive` never reset | **Major** |
| 2.8 | `Animation.swift:149` | SpriteSheet division by zero | **Major** |
| 2.9 | `RepeatAction.swift:22` | Cancel not propagated to inner action | **Major** |
| 2.10 | `Game.swift:74` | Game loop opaqueness — plugins can't hook frame | **Minor** |
| 2.11 | `Scene.swift:108` | Actor in two scenes — silent state corruption | **Minor** |
| 3.1 | — | No SPM support | **Major** |
| 3.6 | `Camera.swift:102` | Early return leaves layer frame stale | **Minor** |
| 4.4 | `Timeline.swift:85` | Mutation of `immediateUpdatables` during iteration | **Minor** |
| 4.5 | `Event.swift:62` | Dict mutation during iteration (fragile) | **Minor** |
| 5.1 | `Grid.swift:363` | `x ^ y` hash causes O(n) dict access | **Major** |
| 5.2 | `Actor.swift:256` | New actions allocated per velocity change | **Major** |
| 5.3 | `Timeline.swift` | BST degenerates to linked list | **Major** |
| 5.4 | `Group.swift:44` | String alloc per hash comparison | **Minor** |
| 7.1–7.10 | `Tests/` | Multiple test coverage gaps | **Major** |
| 8.1 | Multiple | Deprecated `hashValue` usage | **Minor** |
| 8.2 | Multiple | Deprecated `class` keyword in protocols | **Minor** |
| 8.3 | `ActionToken.swift:79` | Recursive cancel chain | **Minor** |

---

## 10. Recommended Fix Priority

1. **Immediately:** Fix the CADisplayLink retain cycle (§2.5) — use a weak-target proxy.
2. **Immediately:** Fix `Scene.deactivate()` to include blocks (§2.1) — one-line fix, high risk.
3. **Immediately:** Fix `TextureManager.load()` recursive fallback prefix (§2.2) — pass `additionalNamePrefix`, not `namePrefix`.
4. **Immediately:** Clear `unsortedUpdatables` after `Timeline.activate()` (§2.3).
5. **Before next release:** Fix the `Grid.Index` hash function (§5.1) — measurable performance regression in non-trivial scenes.
6. **Before next release:** Add thread safety to `TextureManager.cache` or document and enforce main-thread-only constraint (§2.4).
7. **Before next release:** Add `SpriteSheet` input validation with `precondition(rowCount > 0 && frameCount >= rowCount)` (§2.8).
8. **Short term:** Add Swift Package Manager support.
9. **Short term:** Update all deprecated `hashValue` → `hash(into:)` and `class` → `AnyObject` protocol constraints.
10. **Short term:** Write missing test cases for the five blocker bug paths (§7.1–7.5).
