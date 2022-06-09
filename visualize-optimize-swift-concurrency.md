# Visualize and optimize Swift concurrency

Presenters:
- Harjas Monga, Performance Tools Team
- Mike Ash, Swift Runtime Team

Make concurrency code go faster and visualize it, including new concurrency Instrument

## Swift concurrency recap

- Async/await
- Tasks
- Structured concurrency
- Actors

New in Xcode 14 is an Instrument that helps you capure and visualize all this activity within your app. Locate problems and improve performance.

## Concurrency optimization

Common problems when writing concurrent code that can introduce poor performance or bugs:
- Main actor blocking
- Actor contention
- Thread pool exhaustion
- Continuation misuse

### Main Actor Blocking

Long-running task blocks the main thread (UI thread). Your app appears to lock up and becomes unresponsive.

Use the new Swift Concurrency template in Instruments.

Start by checking out the stats in Swift Tasks Instrument. This shows you how many tasks are executing simultaneously. We also have Alive tasks (how many present at a time) and total tasks. Take a close look at Alive and Total task statistics.

The Task Forest in the bottom half of the window shows parent-child relationship betwen

You can right-click on a task to pin a Track that contains all the info about a selected Task to the timeline. You can find tasks of interest that may be running for a very long time or are stuck waiting for access to an actor.

Different views of a pinned task to get access to different information about the task.

To profile the project and check out the Instruments:

Pull up a project in Xcode and press Command + I. This compiles the application and opens Instruments.
Pick the Swift Concurrency template in Instruments, and start recording.
Use the app.
Now that you have a trace, start investigating.

Use option + drag to zoom in on an area of interest.

Right click on a symbol to open in the source viewer. You can view source to find out where tasks of interest are being executed.

In the event of a UI hang, move long-running tasks off the MainActor. Example shows refactoring to add a dedicated actor for long-running tasks. 

Actors serialize access to shared state. You can use concurrent language features to better utilize parallel computation. Move tasks that don't need access to shared state off the actor. Possibly by breaking them up so only the parts that need access to the shared state are accessing the actor. Maybe pull functions out into detached tasks.

Example shows refactoring some functionality to a detached task.

## Thread pool exhaustion

Can hurt performance or deadlock an application. Code within a task can perform a blocking call without suspending. Tasks then occupy threads and reduce the amount of parallel computation that can be done.

- Avoid blocking calls. File and network IO must be performed using async APIs.
- Avoid waiting on condition variables and semaphores.
- Use async APIs for blocking operations. Fine-grained, briefly-held locks are acceptible if necessary. Avoid locks that have a lot of contention.
- Make blocking calls outside Swift Concurrency - i.e. running on a Dispatch Pool

## Continuation misuse

When using continuations, use them correctly. Continuations are the bridge between Swift concurrency and other forms of async code. 

Contination callbacks must be called exactly once; no more, no less. This is a common requirement in callback-based APIs. Swift concurrency makes this a requirement. If a callback is called more than once, the app will crash. If a callback is not called, the task will leak.

## More to explore

- Source and disassembly integration
- Time profiler integration
- Graphic visualizations of structured concurrency
- Task creation calltrees

## Related Sessions

- [Eliminate data races using Swift Concurrency](eliminate-data-races-using-swift-concurrency.md)
