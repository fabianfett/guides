# Swift Concurrency adoption guidelines for Swift Server Libraries

This writeup attempts to provide a set of guidelines to follow by authors of server-side swift libraries. Specifically a lot of the discussion here revolves around what to do about existing APIs and libraries making extensive use of Swift NIO’s EventLoopFuture and related types.

Swift Concurrency is a multi-year effort. It is very valuable for the server community to participate in this multi-year adoption of the concurrency features, one by one, and provide feedback while doing so. As such, we should not hold off adopting concurrency features until Swift 6 as we may miss valuable opportunity to improve the concurrency model.

In 2021 we saw structured concurrency and actors. Now is a great time to provide APIs using those primitives. In the future we will see fully checked Swift concurrency. This will come with breaking changes. For this reason adopting the new concurrency features can be split into two phases.


## What you can do right now

### API Design

Firstly, existing libraries should strive to add `async` functions where possible to their user-facing “surface” APIs in addition to existing `*Future` based APIs wherever possible. These additive APIs can be gated on the Swift version and can be added without breaking existing users, for example like this:

```swift
extension Worker {
  func work() -> EventLoopFuture<Value> { ... }
  
  #if swift(>=5.5)
  func work() async throws -> Value { ... }
  #endif
}
```

If a function cannot fail but was using futures before, it should not include the throws keyword in its new incarnation. 

Such adoption can begin immediately, and should not cause any issues to existing users of existing libraries. 

### SwiftNIO helper functions

To allow an easy transition to async code, SwiftNIO offers a number of helper methods on EventLoopFuture and -Promise. Those live in the `_NIOConcurrency` module and will move to NIOCore once Swift concurrency is released.

On every `EventLoopFuture` you can call `.get()` to transition the future into an awaitable invocation. If you want to translate async/await calls to an EventLoopFuture we recommend the following pattern:

```swift 
#if swift(>=5.5)
func yourAsyncFunctionConvertedToAFuture(on eventLoop: EventLoop) 
    -> EventLoopFuture<Result> {
    let promise = context.eventLoop.makePromise(of: Out.self)
    promise.completeWithTask {
        try await yourMethod(yourInputs)
    }
    return promise.futureResult
}
#endif
```

Further helpers exist for `EventLoopGroup`, `Channel`, `ChannelOutboundInvoker` and `ChannelPipeline`.

### Sendable Checking

> [SE-0302][SE-0302] introduced the Sendable protocol, which is used to indicate which types have values that can safely be copied across actors or, more generally, into any context where a copy of the value might be used concurrently with the original. Applied uniformly to all Swift code, Sendable checking eliminates a large class of data races caused by shared mutable state.
>
> -- from [Staging in Sendable checking][sendable-staging], which outlines the Sendable adoption plan for Swift 6.

In the future we will see fully checked Swift concurrency. The language features to support this are the `Sendable` protocol and the `@Sendable` keyword for closures. Since sendable checking will break existing Swift code, a new major Swift version is required.

To ease the transition to fully checked Swift code, it is possible to annotate your APIs with the Sendable protocol today.

You can start adopting Sendable and getting appropriate warnings in Swift 5.5 already by passing the `-warn-concurrency` flag, you can do so in SwiftPM for the entire project like so:

```
swift build -Xswiftc -Xfrontend -Xswiftc -warn-concurrency
```


#### Sendable checking today

Sendable checking is currently disabled in Swift 5.5(.0) because it was causing a number of tricky situations for which we lacked the tools to resolve.

Most of these issues have been resolved on today’s `main` branch of the compiler, and are expected to land in the next Swift 5.5 releases. It may be worthwhile waiting with adoption until the next version(s) after 5.5.0.

For example, one of such capabilities is the ability for tuples of `Sendable` types to conform to `Sendable` as well. We recommend holding off adoption of `Sendable` until this patch lands in Swift 5.5 (which should be relatively soon). With this change, the difference between Swift 5.5 with `-warn-concurrency` enabled and Swift 6 mode should be very small, and manageable on a case by case basis.

#### Backwards compatibility of declarations and “checked” Swift Concurrency

Adopting Swift Concurrency will progressively cause more warnings, and eventually compile time errors in Swift 6 when sendability checks are violated, marking potentially unsafe code.

It may be difficult for a library to maintain a “pre Swift 6” version along with a Swift 6 version that fully embraces the new concurrency checks. For example, it may be necessary to mark generic types as Sendable, like so:

```swift
struct Container<Value: Sendable>: Sendable { ... }
```

Here, the `Value` type must be marked `Sendable` for Swift 6’s concurrency checks to work properly with such container. However, since the Sendable type does not exist in Swift prior to Swift 5.5, it would be difficult to maintain a library that supports both Swift 5.4+ as well as Swift 6.

In such situations, it may be helpful to utilize the following trick to be able to share the same Container declaration between both Swift versions of the library:

```swift
#if compiler(>=5.5)
public typealias MYPREFIX_Sendable = Swift.Sendable
#else 
public typealias MYPREFIX_Sendable = Any
#endif
```

The Any alias is effectively a no-op when applied as generic constraint, and thus this way it is possible to keep the same Container<Value> declaration working across Swift versions.

### Task Local Values and Logging

The newly introduced TaskLocal Values API ([SE-0311][SE-0311]) allows for implicit carrying of metadata along with Task execution. It is a natural fit for for tracing and carrying metadata around with task execution, and e.g. including it in log messages. 

We are working on adjusting Swift Log to become powerful enough to automatically pick up and log specific task local values. This change will be introduced in a source compatible way. 

For now libraries should continue using logger metadata, but we expect that a lot of the manually passing metadata to each log statement can be replaced in the future with setting task local values. 

### Preparing for the concept of Deadlines

Deadlines are another feature that closely relate to Swift Concurrency, and were originally pitched during the early versions of the Structured Concurrency proposal and later on moved out of it. The Swift team remains interested in introducing deadline concepts to the language and some preparation for it already has been performed inside the concurrency runtime. Right now however, there is no support for deadlines in Swift Concurrency and it is fine to continue using mechanisms like NIODeadline or similar mechanisms to cancel tasks after some period of time has passed. 

Once Swift Concurrency gains deadline support, they will manifest in being able to cancel a task (and its child tasks) once such deadline (point in time) has been exceeded. For APIs to be “ready for deadlines” they don’t have to do anything special other than preparing to be able to deal with Tasks and their cancellation.

### Cooperatively handling Task cancellation

Task cancellation exists today in Swift Concurrency and is something that libraries may already handle. In practice it means that any asynchronous function (or function which is expected to be called from within Tasks), may use the Task.isCancelled or try Task.checkCancellation() APIs to check if the task it is executing in was cancelled, and if so, it may cooperatively abort any operation it was currently performing.

Cancellation can be useful in long running operations, or before kicking off some expensive operation. For example, an HTTP client MAY check for cancellation before it sends a request — it perhaps does not make sense to send a request if it is known the task awaiting on it does not care for the result anymore after all!

Cancellation in general can be understood as “the one waiting for the result of this task is not interested in it anymore”, and it usually is best to throw a “cancelled” error when the cancellation is encountered. However, in some situations returning a “partial” result may also be appropriate (e.g. if a task is collecting many results, it may return those it managed to collect until now, rather than returning none or ignoring the cancellation and collecting all remaining results).

## What to expect with Swift 6

### Sendable: Global variables & imported code

Today, Swift 5.5 does not yet handle global variables at all within its concurrency checking model. This will soon change but the exact semantics are not set in stone yet. In general, avoid using global properties and variables wherever possible to avoid running into issues in the future. Consider deprecating global variables if able to.

Some global variables have special properties, such as errno which contains the error code of system calls. It is a thread local variable and therefore safe to read from any thread/Task. We expect to improve the importer to annotate such globals with some kind of “known to be safe” annotation, such that the Swift code using it, even in fully checked concurrency mode won’t complain about it. Having that said, using errno and other “thread local” APIs is very error prone in Swift Concurrency because thread-hops may occur at any suspension point, so the following snippet is very likely incorrect:

```swift
sys_call(...)
await ...
let err = errno // BAD, we are most likely on a different thread here (!)
```

Please take care when interacting with any thread-local API from Swift Concurrency. If your library had used thread local storage before, you will want to move them to use task-local values (https://github.com/apple/swift-evolution/blob/main/proposals/0311-task-locals.md) instead as they work correctly with Swift’s structured concurrency tasks.

Another tricky situation is with imported C code. There may be no good way to annotate the imported types as Sendable (or it would be too troublesome to do so by hand). Swift is likely to gain improved support for imported code and potentially allow ignoring some of the concurrency safety checks on imported code. 

These relaxed semantics for imported code are not implemented yet, but keep it in mind when working with C APIs from Swift and trying to adopt the -warn-concurrency mode today. Please file any issues you hit so we can inform the development of these checking heuristics based on real issues you hit.

### Custom Executors

We expect the ability that Swift Concurrency would allow custom executors. A custom executor would allow the ability to run actors / tasks “on” such executor. It is possible that EventLoops could become such executors, however the custom executor proposal has not been pitched yet.

While we expect potential performance gains from using custom executors “on the same event loop” by avoiding asynchronous hops between calls to different actors, their introduction will not fundamentally change how NIO libraries are structured.

The guidance here will evolve as Swift Evolution proposals for Custom Executors are proposed, but don’t hold off adopting Swift Concurrency until custom executors “land” - it is important to start adoption early. For most code we believe that the gains from adopting Swift Concurrency vastly outweigh the slight performance cost actor-hops might induce.


### Reduce use of Swift NIO Futures as “Concurrency Library“

Swift NIO currently provides a number of currency types for the Swift on Server ecosystem. Most notably `EventLoopFuture`s and `EventLoopPromise`s, that are used widely for asynchronous results. While the SSWG recommended using those in the past for easier interplay of server libraries, we advice to deprecate or remove them once Swift 6 lands. The swift-server ecosystem should go all in on the structured concurrency features the languages provides. For this reason, it is crucial to provide async/await APIs today, to give your library users time to adopt the new APIs.

Some NIO types will remain however in the public interfaces of Swift on server libraries. We expect that networking clients and servers continue to be initialized with EventLoopGroups. The underlying transport mechanism (NIOPosix and NIOTransportServices) should become implementation details however and should not be exposed to library adopters.

### NIO 3

While subject to change, it is likely that Swift NIO will cut a 3.0 release in the months after Swift 6.0, at which point in time Swift will have enabled “full” Sendable checking.

Do not expect NIO to suddenly become “more async”, NIO’s inherent design principles are about performing small tasks on the event loop and using Futures for any async operations. The design of NIO is not expected to change. It is crucial to its high performance networking design. Channel pipelines are not expected to become “async”.

The NIO team will however use the chance to remove deprecated APIs and improve some APIs. The scope of changes should be comparable to the NIO1 → NIO2 version bump. If your SwiftNIO code compiles today without warnings, chances are high that it will continue to work without modifications in NIO3.

After the release of NIO3, NIO2 will see bug fixes only.

### End-user code breakage

It is expected that Swift 6 will break some code. As mentioned Swift NIO 3 is also going to be released sometime around Swift 6 dropping. Keeping this in mind, it might be a good idea to align major version releases around the same time, along with updating version requirements to Swift 6 and NIO 3 in your libraries.

Both Swift and Swift NIO are not planning to do “vast amounts of change”, so adoption should be possible without major pains.

### Guidance for library users

As soon as Swift 6 comes out, we recommend using the latest Swift 6 toolchains, even if using the Swift 5.5.n language mode (which may yield only warnings rather than hard failures on failed Sendability checks).

[sendable-staging]: https://github.com/DougGregor/swift-evolution/blob/sendable-staging/proposals/nnnn-sendable-staging.md
[SE-0302]: https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md
[SE-0311]: https://github.com/apple/swift-evolution/blob/main/proposals/0311-task-locals.md