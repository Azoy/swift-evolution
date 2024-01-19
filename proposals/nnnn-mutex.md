# Synchronous Mutual Exclusion Lock 🔒

* Proposal: [SE-NNNN](NNNN-mutex.md)
* Author: [Alejandro Alonso](https://github.com/Azoy)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

This proposal introduces a mutual exclusion lock, or a mutex, in the standard library as a new synchronization primitive in the synchronization module.

## Motivation

Since the dawn of Swift, one of the most difficult issues to resolve was synchronizing access for core standard library collections like `Array`, `Dictionary` and `Set`. These types are not concurrent and do not provide any sort of concurrency guarantees for things like thread safety. However, it is still very common to accidentally write a class with a dictionary stored property and perform no synchronization to ensure access to this property is well defined across multiple threads.

One of the simplest solutions to this problem is to protect this mutable state with a lock. Most commonly, said lock would be implemented via a mutex and ensures that a single execution context has read/write access to the value the lock is protecting. In fact, locks make it extremely easy to protect any sort of state value. There are hundreds, maybe thousands, of the same lock implementation (slight variations of course) in almost every code base I look through. These are just a few implementations in related Swift projects:

* [Observation module in Standard Library](https://github.com/apple/swift/blob/main/stdlib/public/Observation/Sources/Observation/Locking.swift)
* [Open source Foundation repository](https://github.com/apple/swift-foundation/blob/main/Sources/_FoundationInternals/LockedState.swift)
* [Swift NIO's NIOLock](https://github.com/apple/swift-nio/blob/main/Sources/NIOConcurrencyHelpers/NIOLock.swift)

Of course the main issue is that there isn't a single standardized implementation for this kind of type resulting in everyone essentially needing to roll their own. On Apple platforms with the Apple SDK there is [`OSAllocatedUnfairLock`](https://developer.apple.com/documentation/os/osallocatedunfairlock) which aims to bring some of this standardization to Apple platforms, but of course these open source libraries need to interoperate on platforms where this type doesn't exist.

Rolling your own lock implementation isn't an easy endeavor either if you didn't already know about Swift's memory model for structs and classes. Consider the following naive implementation of an `os_unfair_lock` implementation in Swift:

```swift
struct OSUnfairLock {
  var storage: os_unfair_lock_s
  
  mutating func lock() {
    os_unfair_lock_lock(&storage)
  }
}
```

This is \_extremely\_ incorrect. Swift does not guarantee a stable address at all for the storage property and the compiler is free to emit as many copies of this thing as it wants on any and all uses. `&storage` also invokes the "inout" operator which will emit dynamic exclusivity checks surrounding the access. Ok, just change `struct` to `class` and we have stable addresses to properties right? Well, yes, but actually no. Swift guarantees that ivars have stable addresses, but accessing their pointer with `&` shares all of the same problems we just discussed with structs. The compiler is free to emit a copy of the property and give you a pointer to that copy, emit exclusivity checks, etc. It's still not the right solution.

What you'll see a lot of implementations do is either 1. use `UnsafeMutablePointer.allocate` or 2. go through `ManagedBuffer` to get access to a stable address but also ensure there won't be any intermediate copies or exclusivity checks. Both of those solutions are super inefficient because of the allocation which is completely unnecessary, but unfortunately it was the only solution Swift had to offer in this space. This is the core reason why the standard library has never proposed a proper mutex type because we always knew it would not be the solution we wanted.

## Proposed solution

We \_finally\_ propose a new type in the Standard Library Synchronization module called `Mutex`. This type will be a wrapper over a platform implemented mutex primitive carrying along some user defined state to protect. Below is an example use of `Mutex` protecting some internal data in a class who is to be used across many threads:

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

This is a very common use case for such a type to provide quick and relatively easy synchronization for this internal shared mutable state. Another very common example is a global cache, usually either an array or dictionary:

```swift
let globalCache = Mutex<[MyKey: MyValue]>([:])
```

## Detailed design

### Underlying System Mutex Implementation

The `Mutex` type proposed is not a new mutex implementation whatsoever, we are calling into a platform's implementation. 

* macOS, iOS, watchOS, tvOS:
  * `os_unfair_lock`
* Linux:
  * `pthread_mutex_t`
* Windows:
  * `SRWLOCK`

These mutex implementations all have different capabilities and guarantee different levels of fairness, but for our `Mutex` type we do not guarantee fairness whatsoever meaning it's okay to have different behavior from platform to platform. We only guarantee that a single execution context will have access to the critical section via mutual exclusion.

### API Design

Below is the complete API design for the new `Mutex` type:

```swift
// TODO: Documentation
public struct Mutex<Value: ~Copyable>: ~Copyable {
  // TODO: Documentation
  public init(_: consuming Value)
  
  // TODO: Documentation
  public borrowing func tryWithLock<U: ~Copyable & Sendable, E>(
  	_: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U?
  
  // TODO: Documentation
  public borrowing func tryWithLockUnchecked<U: ~Copyable, E>(
  	_: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U?
  
  // TODO: Documentation
  public borrowing func withLock<U: ~Copyable & Sendable, E>(
    _: @Sendable (inout Value) throws(E) -> U
  ) throws(E) -> U
  
  // TODO: Documentation
  public borrowing func withLockUnchecked<U: ~Copyable, E>(
  	_: (inout Value) throws(E) -> U
  ) throws(E) -> U
}

extension Mutex where Value == Void {
  // TODO: Documentation
  @available(*, noasync)
  public borrowing func lock()
  
  // TODO: Documentation
  @available(*, noasync)
  public borrowing func unlock()
}

extension Mutex: Sendable where Value: Sendable {}
```

## Interaction with Existing Language Features

Following the [SE-0410 Atomics proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0410-atomics.md), `Mutex` will share the same limitations `Atomic` and `AtomicLazyReference` has; you will not be able to declare a value of type `Mutex` with a `var`. There are other limitations such as not being able to pass one via `inout`, but please refer to the [Atomics proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0410-atomics.md) for a more in-depth discussion on what is allowed and not allowed. These restrictions are enabled for `Mutex` for pretty much all of the same reasons why it was resticted for `Atomic`. We do not want to introduce dynamic exclusivity checking when even accessing a value of `Mutex` as a class stored property for instance.

### Interactions with Swift Concurrency

Similar to `Atomic`, `Mutex` will have a conditional conformance to `Sendable` when the underlying value itself is also `Sendable`. Consider the following example declaring a global mutex in some top level script:

```swift
let lockedPointer = Mutex<UnsafeMutablePointer<Int>>(...)

func something() {
  // warning: non-sendable type 'Mutex<UnsafeMutablePointer<Int>>' in
  // 					asynchronous access to main actor-isolated let 'lockedPointer'
  //					cannot cross actor boundary
  // note: consider making generic struct 'Mutex' conform to the 'Sendable'
  //			 protocol
  let pointer = lockedPointer.withLock {
    $0
  }
  
  ... bad stuff with pointer
}
```

Our variable is treated as "immutable", but the underlying type is not sendable, thus this variable is implicitly main actor-isolated. Here, we're attempting to reference a main actor-isolated piece of data on a synchronous global function that is not isolated to any actor, thus the warning (which will become an error in Swift 6). However, this warning is actually appreciated here because while the lock does protect the state it's holding onto, it does not protect class references or any underlying pointers that the value may contain leading to unprotected shared mutable state. In the example, the global function `something` now has access to load/store values from this pointer, but there's nothing synchronizing access to the memory referenced by the pointer thus multiple threads could potentially race with each other trying to read/write data into this memory. The same applies to non-sendable class references, anybody can alter the class instance with a reference to it and the only thing the lock is protecting is the reference value itself which is meaningless once multiple threads can get a copy of the reference.

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

Usages of this mutex no longer emit these sendability warnings, but if we alter the example just very slightly we see the next issue we need to solve:

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

If the `withLock` method was not labeled with `@Sendable` then this code would emit no warnings. All good right? The compiler hasn't complained to us! Unfortunately, this highlights one of the bigger issues I mentioned earlier in that `Mutex` does not protect class references or any underlying memory referenced by pointers. This is still a hole for shared mutable state. We've now given a non-sendable value complete access to the memory pointed to by our value and that class does not guarantee synchronization for the data being modified at the pointer. We need to mark the closure in the `withLock` as `@Sendable` :

```swift
@MainActor
func something() {
  lockedPointer.withLock {
    // warning: non-sendable type 'NonSendableReference' in asynchronous access
    // 					to main actor-isolated let 'nonSendableRef' cannot cross actor
    //					boundary
    nonSendableRef.prop = $0
  }
  
  // or if you tried to be sneaky:
  let nonSendableRefCopy = nonSendableRef
  
  lockedPointer.withLock {
    // warning: capture of 'nonSendableRefCopy' with non-sendable type
    // 					'NonSendableReference' in a '@Sendable' closure
    nonSendableRefCopy.prop = $0
  }
}
```

By marking the closure as such, we've effectively declared that the mutex is in itself its own isolation domain. We must not let non-sendable values it holds onto be unsafely sent across isolation domains to prevent these holes of shared mutable state.

## Source compatibility

Source compatibility is preserved with the proposed API design as it is all additive as well as being hidden behind an explicit `import Synchronization`. Users who have not already imported the Synchronization module will not see this type, so there's no possibility of potential name conflicts with existing `Mutex` named types for instance. Of course, the standard library already has the rule that any type names that collide will disfavor the standard library's variant in favor of the user's defined type anyway.

## ABI compatibility

The API proposed here is fully addative and does not change or alter any of the existing ABI.

`Mutex` as proposed will be a new `@frozen` struct which means we cannot change its layout in the future on ABI stable platforms, namely the Darwin family. Because we cannot change the layout, we will most likely not be able to change to a hypothetical new and improved system mutex implementation on those platforms. If said new system mutex were to share the layout of the currently proposed underlying implementation, then we _may_ be able to migrate over to that implementation. Keep in mind that Linux and Windows are non-ABI stable platforms, so we can freely the underlying implementation if the platform ever supports something better.

## Future directions

There are quite a few potential future directions this new type can take as well as new future similar types.

### Transferring Parameter

With [Region Based Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0414-region-based-isolation.md), a future direction in that proposal is the introduction of a `transferring` modifier to function parameters. This would allow the `Mutex` type to decorate its closure parameter in the `withLock` API as `transferring` and potentially remove the `@Sendable` restriction on the closure altogether. By marking the closure parameter as such, we guarantee that state held within the lock cannot be assigned to some non-sendable captured reference which is the primary motivator for why the closure is marked `@Sendable` now to begin with. A future closure based API may look something like the following:

```swift
public borrowing func withLock<U: ~Copyable & Sendable>(
	_: (transferring inout Value) throws -> U
) rethrows -> U
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

The above example shows an API similar to Rust's `MutexGuard` which allows access to the protected state in the mutex. C++'s guard on the other hand just performs `lock()` and `unlock()` for the user (because `std::mutex` doesn't protect any state). Of course the immediate issue with this approach right now is that we don't have access to non-escapable types so when one were to call `lock()`, there's nothing preventing the user from taking the guard value and escaping it from the scope that the caller is in. Rust resolves this issue with lifetimes, but C++ doesn't solve this at all and just declares:

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

But again, this issue only occurs with `Void` and we forsee this particular specialization being somewhat rare because attaching state to a mutex is a much more common paradigm that fits into Swift.

### Reader-Writer Locks, Recursive Locks, etc.

Another interesting future direction is the introduction of new kinds of locks to be added to the standard library, such as a reader-writer lock. One of the core issues with the proposal mutual exclusion lock is that anyone who takes the lock, either a reader and/or writer, must be the only person with exclusive access to the protected state. This is somewhat unfortunate for models where there are infinitely more readers than there will be writers to the state. A reader-writer lock resolves this issue by allowing multiple readers to take the lock and enforces that writers who need to mutate the state have exclusive access to the value. Another potential lock is a recursive lock who allows the lock to be acquired multiple times by the acquired thread. In the same vein, the acquired thread needs to be the one to release the lock and needs to release X amount of times equal to the number of times it acquired it.

## Alternatives considered

### Rename to `Lock` or similar

A very common name for this type in various codebases is simply `Lock`. This is a decent name because many people know immediately what the purpose of the type is, but the issue is that it doesn't describe _how_ it's implemented. I believe this is a very important aspect of not only this specific type, but of synchronization primitives in general. Understanding that this particular lock is implemented via mutual exclusion conveys to developers who have used something similar in other languages that they cannot have multiple readers and cannot call `lock()` again on the acquired thread for instance. Many languages similar to Swift have opted into a similar design, naming this type mutex instead of a vague lock. In C++ we have `std::mutex`, and in Rust we have `Mutex`.

### Include `Mutex` in the default Swift namespace (either in `Swift` or in `_Concurrency`)

This is another intriguing idea because on one hand misusing this type is significantly harder than misusing something like `Atomic`. Generally speaking, we do want folks to reach for this when they just need a simple traditional lock. However, by including it in the default namespace we also unintentionally discourage folks from reaching for the language features and APIs they we've already built like `async/await`, `actors`, and so much more in this space. In the presence of our concurrency model, these traditional locks generally shouldn't be used except niche scenarios.

### Introduce a separate `UnsafeMutex` or similar

The purpose of this alternative is to absolutely disallow uses of the primitive `lock()` and `unlock()` on the safe `Mutex` type. Moving these to an unsafe facility makes it impossible for users of the safe `Mutex` to accidentally use it both in async code, but also within the `withLock` facility itself as well. 

Consider the following:

```swift
let mutex = Mutex<()>()

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
