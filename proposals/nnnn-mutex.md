# Synchronous Mutual Exclusion Lock 🔒

* Proposal: [SE-NNNN](NNNN-mutex.md)
* Author: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

This proposal introduces a mutual exclusion lock, or a mutex, to the standard library. `Mutex` will be a new synchronization primitive in the synchronization module.

## Motivation

Since the dawn of Swift, one of the most difficult issues to resolve has been synchronizing access for core standard library collections such as `Array`, `Dictionary` and `Set`. These types are not concurrent and do not provide any sort of thread safety, and are prone to data races. For example, it is a common pitfall to implement a class that contains a property of type `Dictionary` without proper synchronization ensuring that access to this property is well defined across multiple threads.

A simple solution to this problem is to protect this mutable state with a lock. Commonly, the protection would be implemented as a mutex, and ensures that only one execution context at a time has read/write access to the value the lock is protecting. Locks make it easy to protect any sort of value used as shared state. There are hundreds of similar lock implementations in Swift code bases. The following are a few implementations in projects related to Swift:

* [`_ManagedCriticalState` in the Observation module](https://github.com/apple/swift/blob/main/stdlib/public/Observation/Sources/Observation/Locking.swift)
* [`LockedState` in swift-foundation](https://github.com/apple/swift-foundation/blob/main/Sources/_FoundationInternals/LockedState.swift)
* [`NIOLock` in Swift NIO](https://github.com/apple/swift-nio/blob/main/Sources/NIOConcurrencyHelpers/NIOLock.swift)
* [`OSAllocatedUnfairLock`](https://developer.apple.com/documentation/os/osallocatedunfairlock)

Of course the main issue is that there isn't a single standardized implementation for this kind of type resulting in everyone essentially needing to roll their own. Rolling your own lock implementation isn't an easy endeavor. Many implementations either 1. use `UnsafeMutablePointer.allocate` or 2. go through `ManagedBuffer` both to get access to a stable address but to also ensure there won't be any intermediate copies or exclusivity checks. Both of those solutions are super inefficient because of the allocation which is completely unnecessary, but unfortunately it was the only solution Swift had to offer in this space. This is the core reason why the standard library has never proposed a proper mutex type because we always knew it would not be the solution we wanted.

Until now the only workable solution to implement locking in Swift has involved explicit heap allocations, either manual or managed. We can do better.

## Proposed solution

We \_finally\_ propose a new type in the Standard Library Synchronization module: `Mutex`. This type will be a wrapper over a platform-specific mutex primitive, along with a user-defined state to protect. Below is an example use of `Mutex` protecting some internal data in a class usable simultaneously by many threads:

```swift
class FancyManagerOfSorts {
  let cache: Mutex<[String: Resource]>
  
  func save(_ resource: Resouce, as key: String) {
    cache.withLock {
      $0[key] = resource
    }
  }
}
```

Use cases for such a synchronized type are common. Another common need is a global cache, such as a dictionary:

```swift
let globalCache = Mutex<[MyKey: MyValue]>([:])
```

## Detailed design

### Underlying System Mutex Implementation

The `Mutex` type proposed is a wrapper around a platform's implementation.

* macOS, iOS, watchOS, tvOS, visionOS:
  * `os_unfair_lock`
* Linux:
  * `pthread_mutex_t`
* Windows:
  * `SRWLOCK`

These mutex implementations all have different capabilities and guarantee different levels of fairness. Our proposed `Mutex` type does not guarantee fairness, and therefore it's okay to have different behavior from platform to platform. We only guarantee that only one execution context at a time will have access to the critical section, via mutual exclusion.

### API Design

Below is the complete API design for the new `Mutex` type:

```swift
/// A synchronization primitive that protects shared mutable state via
/// mutual exclusion.
///
/// The `Mutex` type offers non-recursive exclusive access to the state
/// it is protecting by blocking threads attempting to acquire the lock.
/// Only one execution context at a time has access to the value stored
/// within the `Mutex` allowing for exclusive access.
///
/// An example use of `Mutex` in a class used simultaneously by many
/// threads protecting a `Dictionary` value:
///
///     class Manager {
///       let cache = Mutex<[Key: Resource]>([:])
///
///       func saveResouce(_ resource: Resouce, as key: Key) {
///         cache.withLock {
///           $0[key] = resource
///         }
///       }
///     }
///
public struct Mutex<Value: ~Copyable>: ~Copyable {
  /// Initializes a value of this mutex with the given initial state.
  ///
  /// - Parameter initialValue: The initial value to give to the mutex.
  public init(_: consuming Value)
  
  /// Attempts to acquire the lock and then calls the given closure if
  /// successful.
  ///
  /// If the calling thread was successful in acquiring the lock, the
  /// closure will be executed and then immediately after it will
  /// release ownership of the lock. If we were unable to acquire the
  /// lock, this will return `nil`.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     guard mutex.tryLock() else {
  ///       return nil
  ///     }
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Warning: Recursive calls to `tryWithLock` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified.
  ///
  /// - Parameter body: A closure with a parameter of `Value`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed if the calling thread acquires
  ///   the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter
  ///   or nil if the lock couldn't be acquired.
  public borrowing func tryWithLock<U: ~Copyable & Sendable, E>(
    _ body: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U?
  
  /// Attempts to acquire the lock and then calls the given closure if
  /// successful.
  ///
  /// If the calling thread was successful in acquiring the lock, the
  /// closure will be executed and then immediately after it will
  /// release ownership of the lock. If we were unable to acquire the
  /// lock, this will return `nil`.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     guard mutex.tryLock() else {
  ///       return nil
  ///     }
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Note: This version of `withLock` is unchecked because it does
  ///   not enforce any sendability guarantees.
  ///
  /// - Warning: Recursive calls to `tryWithLockUnchecked` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified.
  ///
  /// - Parameter body: A closure with a parameter of `Value`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed if the calling thread acquires
  ///   the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter
  ///   or nil if the lock couldn't be acquired.
  public borrowing func tryWithLockUnchecked<U: ~Copyable, E>(
    _ body: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U?
  
  /// Calls the given closure after acquring the lock and then releases
  /// ownership.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     mutex.lock()
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Warning: Recursive calls to `withLock` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified.
  ///
  /// - Parameter body: A closure with a parameter of `Value`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed once the calling thread has
  ///   acquired the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public borrowing func withLock<U: ~Copyable & Sendable, E>(
    _ body: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U
  
  /// Calls the given closure after acquring the lock and then releases
  /// ownership.
  ///
  /// This method is equivalent to the following sequence of code:
  ///
  ///     mutex.lock()
  ///     defer {
  ///       mutex.unlock()
  ///     }
  ///     return try body(&value)
  ///
  /// - Warning: Recursive calls to `withLockUnchecked` within the
  ///   closure parameter has behavior that is platform dependent.
  ///   Some platforms may choose to panic the process, deadlock,
  ///   or leave this behavior unspecified.
  ///
  /// - Note: This version of `withLock` is unchecked because it does
  ///   not enforce any sendability guarantees.
  ///
  /// - Parameter body: A closure with a parameter of `Value`
  ///   that has exclusive access to the value being stored within
  ///   this mutex. This closure is considered the critical section
  ///   as it will only be executed once the calling thread has
  ///   acquired the lock.
  ///
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public borrowing func withLockUnchecked<U: ~Copyable, E>(
    _ body: (inout Value) throws(E) -> U
  ) throws(E) -> U
}

extension Mutex where Value == Void {
  /// Acquires the lock.
  ///
  /// If the calling thread was unsuccessful in acquiring the lock,
  /// it will block execution until it can be acquired.
  ///
  /// - Warning: Recursive calls to 'lock()' by the acquired thread
  ///   has behavior that is platform dependent. Some platforms may
  ///   choose to panic the process, deadlock, or leave this behavior
  ///   unspecified.
  @available(*, noasync)
  public borrowing func lock()
  
  /// Attempts to acquire the lock.
  ///
  /// If the calling thread acquired the lock, this will return true.
  /// Otherwise, the caller was unsuccessful in acquiring the lock and
  /// will return false.
  ///
  /// - Warning: You must not attempt to call this method in a loop to gain
  /// 	ownership of the lock. It is better to directly call 'lock()'
  ///   instead.
  ///
  /// - Returns: A boolean indicating whether the calling thread
  ///   successfully acquired the lock or not.
  @availabile(*, noasync)
  public borrowing func tryLock() -> Bool
  
  /// Releases the lock.
  ///
  /// The calling thread must have acquired the lock in order to call
  /// this method.
  ///
  /// - Warning: Attempts to call this on unacquired threads will
  ///   have platform dependent behavior, such as paniking or unspecified
  ///   behavior. Attempts to call this when the lock is not acquired at all
  ///   will exhibit similar platform dependent behavior.
  @available(*, noasync)
  public borrowing func unlock()
}

extension Mutex: Sendable where Value: Sendable {}
```

## Interaction with Existing Language Features

`Mutex` will decorated with the `@_staticExclusiveOnly` attribute, meaning you will not be able to declare a variable of type `Mutex` as `var`. These are the same restrictions imposed on the recently accepted `Atomic` and `AtomicLazyReference` types. Please refer to the [Atomics proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0410-atomics.md) for a more in-depth discussion on what is allowed and not allowed. These restrictions are enabled for `Mutex` for all of the same reasons why it was resticted for `Atomic`. We do not want to introduce dynamic exclusivity checking when accessing a value of `Mutex` as a class stored property for instance.

### Interactions with Swift Concurrency

It is important that low level system developers have access to the primitive `lock()`, `tryLock()`, and `unlock()` operations. However, in the face of `async`/`await`, these primitives are very dangerous. The example below highlights incorrect usage of these operations in an `async` function:

```swift
func test() async {
  mutex.lock()             // Called on Thread A
  await downloadImage(...) // <--- Potential suspension point
  mutex.unlock()           // May be called on Thread A, B, C, etc.
}
```

The potential suspension point may cause the proceeding code to be called on a different thread than the one that initiated the `await` call. Because of this, these methods are all marked as `@available(*, noasync)` and are only allowed to be called when `Value == Void` to further nudge folks to use the safer `withLock` APIs. Calling `withLock` in an asynchronous function is \_okay\_ because the same thread that calls `lock()` will be the same one that calls `unlock()` because there will not be any suspension points between the calls.

Similar to `Atomic`, `Mutex` will have a conditional conformance to `Sendable` when the underlying value itself is also `Sendable`. Consider the following example declaring a global mutex in some top level script:

```swift
let lockedPointer = Mutex<UnsafeMutablePointer<Int>>(...)

func something() {
  // warning: non-sendable type 'Mutex<UnsafeMutablePointer<Int>>' in
  //          asynchronous access to main actor-isolated let 'lockedPointer'
  //          cannot cross actor boundary
  // note: consider making generic struct 'Mutex' conform to the 'Sendable'
  //       protocol
  let pointer = lockedPointer.withLock {
    $0
  }
  
  ... bad stuff with pointer
}
```

Our variable is treated as immutable by the compiler, but the underlying type is not sendable thus this variable is implicitly main actor-isolated. We're attempting to reference a main actor-isolated piece of data in a synchronous global function that is not isolated to any actor (hence the warning). However, this warning is actually appreciated. While the lock does protect the state it's holding onto, it does not protect class references or any underlying memory being pointed to (by pointers). In the example, the global function `something` now has access to read/write values from this pointer, but there's nothing synchronizing access to the memory referenced by the pointer. Multiple threads could potentially race with each other trying to read/write data into this memory. The same applies to non-sendable class references, once threads have a hold of the reference, they can potentially race with each other trying to mutate the underlying instance.

We can fix this warning though, if we isolate usages of `something()` to the main actor as well:

````swift
@MainActor
func something() {
  // No more warnings!
  lockedPointer.withLock {
    ...
  }
}
````

Usages of this mutex no longer emit these sendability warnings, but if we alter the example just slightly we see the next issue we need to solve:

```swift
class NonSendableReference {
  var prop: UnsafeMutablePointer<Int>
}

// Some non-sendable class reference somewhere, perhaps a global.
let nonSendableRef = NonSendableReference(...)

@MainActor
func something() {
  lockedPointer.withLock {
    // No warnings!
    nonSendableRef.prop = $0
  }
}
```

If the `withLock` method was not labeled with `@Sendable` then this code would emit no warnings. All good right? The compiler hasn't complained to us! Unfortunately, this highlights one of the bigger issues I mentioned earlier in that `Mutex` does not protect class references or any underlying memory referenced by pointers. This is still a hole for shared mutable state. We've now given a non-sendable value complete access to the memory pointed to by our value. This non-sendable class does not guarantee synchronization for the data being modified at by the pointer. We need to mark the closure in the `withLock` as `@Sendable` :

```swift
@MainActor
func something() {
  lockedPointer.withLock {
    // warning: non-sendable type 'NonSendableReference' in asynchronous access
    //          to main actor-isolated let 'nonSendableRef' cannot cross actor
    //          boundary
    nonSendableRef.prop = $0
  }
  
  // or if you tried to be sneaky:
  let nonSendableRefCopy = nonSendableRef
  
  lockedPointer.withLock {
    // warning: capture of 'nonSendableRefCopy' with non-sendable type
    //          'NonSendableReference' in a '@Sendable' closure
    nonSendableRefCopy.prop = $0
  }
}
```

By marking the closure as such, we've effectively declared that the mutex is in itself its own isolation domain. We must not let non-sendable values it holds onto be unsafely sent across isolation domains to prevent these holes of shared mutable state.

## Source compatibility

Source compatibility is preserved with the proposed API design as it is all additive as well as being hidden behind an explicit `import Synchronization`. Users who have not already imported the Synchronization module will not see this type, so there's no possibility of potential name conflicts with existing `Mutex` named types for instance. Of course, the standard library already has the rule that any type names that collide will disfavor the standard library's variant in favor of the user's defined type anyway.

## ABI compatibility

The API proposed here is fully addative and does not change or alter any of the existing ABI.

`Mutex` as proposed will be a new `@frozen` struct which means we cannot change its layout in the future on ABI stable platforms, namely the Darwin family. Because we cannot change the layout, we will most likely not be able to change to a hypothetical new and improved system mutex implementation on those platforms. If said new system mutex were to share the layout of the currently proposed underlying implementation, then we _may_ be able to migrate over to that implementation. Keep in mind that Linux and Windows are non-ABI stable platforms, so we can freely change the underlying implementation if the platform ever supports something better.

## Future directions

There are quite a few potential future directions this new type can take as well as new future similar types.

### Transferring Parameter

With [Region Based Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0414-region-based-isolation.md), a future direction in that proposal is the introduction of a `transferring` modifier to function parameters. This would allow the `Mutex` type to decorate its closure parameter in the `withLock` API as `transferring` and potentially remove the `@Sendable` restriction on the closure altogether. By marking the closure parameter as such, we guarantee that state held within the lock cannot be assigned to some non-sendable captured reference which is the primary motivator for why the closure is marked `@Sendable` now to begin with. A future closure based API may look something like the following:

```swift
public borrowing func withLock<U: ~Copyable & Sendable, E>(
  _: (transferring inout Value) throws(E) -> U
) throws(E) -> U
```

This would remove a lot of the restrictions that a `@Sendable` closure enforces because this closure isn't escaping nor is it being passed to other isolation domains, it's being ran on the same calling execution context. However, again we guarantee that the passed in parameter, who may not be sendable, can't be written to a non-sendable reference within the closure effectively crossing isolation domains.

### Mutex Guard API

A token based approach for locking and unlocking may also be highly desirable for mutex API. This is similar to C++'s `std::lock_guard` or Rust's `MutexGuard`:

```swift
extension Mutex {
  public struct Guard: ~Copyable, ~Escapable {
    // Hand waving some syntax to borrow Mutex, or perhaps
    // we just store a pointer to it.
    let mutex: borrow Mutex<Value>
    
    public var value: Value {...}
    
    deinit {
      mutex.unlock()
    }
  }
}

extension Mutex {
  public borrowing func lock() -> Guard {...}
}

func add(_ i: Int, to mutex: Mutex<Int>) {
  // This acquires the lock by calling the platform's
  // underlying 'lock()' primitive.
  let mGuard = mutex.lock()
  
  mGuard.value += 1
  
  // At the end of the scope, mGuard is immediately deinitialized
  // and releases the mutex by calling the platform's
  // 'unlock()' primitive.
}
```

The above example shows an API similar to Rust's `MutexGuard` which allows access to the protected state in the mutex. C++'s guard on the other hand just performs `lock()` and `unlock()` for the user (because `std::mutex` doesn't protect any state). Of course the immediate issue with this approach right now is that we don't have access to non-escapable types. When one were to call `lock()`, there's nothing preventing the user from taking the guard value and escaping it from the scope that the caller is in. Rust resolves this issue with lifetimes, but C++ doesn't solve this at all and just declares:

> The behavior is undefined if m is destroyed before the `lock_guard` object is.

Which is not something we want to introduce in Swift if it's something we can eventually prevent.

Naming such a function `lock` feels very natural, but it would run into issues when `Value == Void` because there are now two methods named `lock()`:

```swift
func something(with mutex: Mutex<()>) {
  // warning: 'b' inferred to have type '()'
  let b = mutex.lock()
  
  // Instead, you'd need to specify you want the guard:
  
  let b: Mutex<()>.Guard = mutex.lock()
  
  ...
}
```

But again, this issue only occurs with `Void` and we forsee this particular specialization being somewhat rare because attaching state to a mutex is a much more common paradigm that fits into Swift. However, if we had this feature today, the primitive `lock()`/`unlock()` operations would be better suited in the form of the guard API. I don't believe we'd have those methods if we had guards.

### Reader-Writer Locks, Recursive Locks, etc.

Another interesting future direction is the introduction of new kinds of locks to be added to the standard library, such as a reader-writer lock. One of the core issues with the proposal mutual exclusion lock is that anyone who takes the lock, either a reader and/or writer, must be the only person with exclusive access to the protected state. This is somewhat unfortunate for models where there are infinitely more readers than there will be writers to the state. A reader-writer lock resolves this issue by allowing multiple readers to take the lock and enforces that writers who need to mutate the state have exclusive access to the value. Another potential lock is a recursive lock who allows the lock to be acquired multiple times by the acquired thread. In the same vein, the acquired thread needs to be the one to release the lock and needs to release X amount of times equal to the number of times it acquired it.

## Alternatives considered

### Rename to `Lock` or similar

A very common name for this type in various codebases is simply `Lock`. This is a decent name because many people know immediately what the purpose of the type is, but the issue is that it doesn't describe _how_ it's implemented. I believe this is a very important aspect of not only this specific type, but of synchronization primitives in general. Understanding that this particular lock is implemented via mutual exclusion conveys to developers who have used something similar in other languages that they cannot have multiple readers and cannot call `lock()` again on the acquired thread for instance. Many languages similar to Swift have opted into a similar design, naming this type mutex instead of a vague lock. In C++ we have `std::mutex` and in Rust we have `Mutex`.

### Include `Mutex` in the default Swift namespace (either in `Swift` or in `_Concurrency`)

This is another intriguing idea because on one hand misusing this type is significantly harder than misusing something like `Atomic`. Generally speaking, we do want folks to reach for this when they just need a simple traditional lock. However, by including it in the default namespace we also unintentionally discouraging folks from reaching for the language features and APIs they we've already built like `async/await`, `actors`, and so much more in this space. Gating the presence of this type behind `import Synchronization` is also an important marker for anyone reading code that the file deals with managing their own synchronization through the use of synchronization primitives such as `Atomic` and `Mutex`.

### Introduce a separate `UnsafeMutex` or similar

The purpose of this alternative is to absolutely disallow uses of the primitive `lock()` and `unlock()` on the safe `Mutex` type. Moving these to an unsafe facility makes it impossible for users of the safe `Mutex` to accidentally use it both in async code, but also within the `withLock` facility itself as well. 

Consider the following:

```swift
let mutex = Mutex<()>(())

func something() async {
  mutex.lock() // error: this function is marked 'noasync'
  await doThing()
  mutex.unlock() // error: this function is marked 'noasync'
}
```

The above is already resolved with the current design by marking the primitive functions `@available(*, noasync)`, but it would still allow for the following:

```swift
mutex.withLock {
  mutex.lock() // ERROR: NOT OKAY YOU CANNOT REACQUIRE LOCK
}
```

This would still be valid with the proposed design, however only when `Value == Void`. We feel the narrowness of this problem only occurring in that special case is not enough to warrent introducing two separate types just to resolve this niche issue. Documentation for `lock()` should state this issue and explain that this order of code is not allowed.

In a similar vein, we could also separate out the unchecked variants of the methods in `Mutex` to an `UncheckedMutex` of sorts to reduce API surface to improve documentation, auto complete, and push more towards folks opting into the Swift Concurrency world by enforcing sendability constraints.

````swift
public struct UncheckedMutex<Value: ~Copyable>: ~Copyable {
  ...
  
  // Should this have unchecked in the name? It wouldn't
  // be discernable from the safe one at the call site,
  // but then we're repeating information that's already
  // in the type name 🤔
  public borrowing func withLock<U: ~Copyable, E>(
    _: (inout Value) throws(E) -> U
  ) throws(E) -> U
  
  ...
}
````

An approach like this means uses of `Mutex` only has access to two functions making it easier for developers to choose a sendable checked API over debating if they need an unchecked one.
