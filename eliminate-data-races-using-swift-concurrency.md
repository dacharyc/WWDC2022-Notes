# Eliminate data races using Swift concurrency

Presenters:
- Doug Gregor, Swift Team

Swift concurrency is a set of language-related features that make it easy to write concurrent programs:
- async/await
- structured concurrency
- actors

This talk is a more holistic view, helping you structure your app for data-race-free concurrency

- Task isolation
- Actor isolation
- Atomicity
- Ordering

## Task isolation

A task performs a specific job from start to finish:
- Sequential
- Async and can be suspended at await operations
- Self-contained

```swift
Task.detached {
  let fish = await catchFish()
  let dinner = await cook(fish)
  await eat(dinner)
}
```

Swift protocols can categorize types to reason about behavior.

`Sendable` protocol describes types that can cross an isolation domain. A type can be made `Sendable` by writing a conformance. Value types, such as structs, can conform to Sendable, but unsynchronized reference types, like classes, cannot. It is not safe to operate on them independently because they are reference types.

Wherever tasks can exchange data, there is a `Sendable` constraint.

Classes are reference types, so they can only be `Sendable` under vary narrow constraints, such as when a final class has only immutable storage:

```swift
final class Chicken: Sendable {
    let name: String
    var currentHunger: HungerLevel // produces an error because it contains mutable state
}
```

You _can_ do this when you do stuff like using a lock. You can have the compiler stop throwing errors in these cases by marking them as unchecked:

```swift
//@unchecked can be used, but be careful!
class ConcurrentCache<Key: Hashable & Sendable, Value: Sendable>: @unchecked Sendable {
  var lock: NSLock
  var storage: [Key: Value]
}
```

Values passed into Tasks must conform to Sendable. `@Sendable` function types must conform to `Sendable`:

```swift
struct Task<Success: Sendable, Failure: Error> {
  static func detached(
    priority: TaskPriority? = nil,
    operation: @Sendable @escaping () async throws -> Success
  ) -> Task<Success, Failure>
}
```

Sendable checking maintains task isolation.

However, we need a way to share data among tasks that doesn't introduce data races. This is where actors come in.

## Actor isolation

Actors isolate mutable state and all the code that touches it

```swift
actor Island {
  var flock: [Chicken]
  var food: [Pineapple]

  func advanceTime()
}
```

In order to execute code controlled by the actor, Tasks must access the actor, which they can do in sequence, not concurrently. Only one task can execute an actor at a time. Other tasks must await their turn. This is a potential suspension point.

```swift
func nextRound(islands: [Island]) async {
    for island in islands {
        await island.advanceTime()
    }
}
```

Non-`Sendable` data canot be shared between a task and an actor.

Actors are reference types, but they isolate all their internal mutable state so referencing an actor from a different isolation domain is safe.

Functions within actors can be explicitly marked non-isolated:

```swift
extension Island {
  nonisolated func meetTheFlock() async {
    let flockNames = await flock.map { $0.name }
    print("Meet our fabulous flock: \(flockNames)")
  }
}
```

We still need to access the actor to execute the actor-specific portions of the code, though, like here to access `flock`.

Non-isolated async code executes on the global cooperative pool.

### @MainActor

Represents the main thread where drawing and user interaction occurs.
- Main actor carries a lot of state related to the program's UI
- Lots of UI framework code and app code needs to run on it
- BUT it can still only run a single job at a time

Isolation to the main actor is expressed with `@MainActor`:

```swift
@MainActor func updateView() { … }

Task { @MainActor in
	// …
  view.selectedChicken = lily
}

nonisolated func computeAndUpdate() async {
  computeNewValues()
  await updateView()
}
```

The Swift compiler will guarantee that this code is only executed on the main thread. This can also be applied to types:

```swift
@MainActor
class ChickenValley: Sendable {
  var flock: [Chicken]
  var food: [Pineapple]

  func advanceTime() {
    for chicken in flock {
      chicken.eat(from: &food)
    }
  }
}
```

The properties are only accessible while on the main actor, and the methods are isolated to the main actor unless they explicitly opt out.

### Architecting your app with actors

In your app, your views and view controllers will be on the main actor.

Other program logic should be separated from that main actor, using other actors to safely model shared state and tasks to describe independent work.

## Atomicity

- Actors run one tastk at a time
- When you stop running on an actor, it can run other tasks

You could end up with a high-level data race when the program is in an unexpected state even though no data is corrupted.

Task can get suspended between awaits and state can change. Write code that relies on state as synchronous code so the state of the actor cannot change between awaits.

## Ordering

Programs often rely on handling events in consistent order:
- User input
- Messages from a server

Effects of each event should appear in the order they happened.

Actors are not strictly first-in, first-out. They execute the highest-priority work first. This is an important difference from serial dispatch queues.

Tools for ordering:
- Tasks. They run code in order, from beginning to end, with the normal control flow you're used to (within a task).
- `AsyncStreams` deliver elements in order

## Related Sessions

- Visualize and Optimize Swift Concurrency
