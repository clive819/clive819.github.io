---
title: 'Understanding Noncopyable Types in Swift'
author: 'Clive Liu'
layout: post
tags: [Swift]
---

At WWDC24, Apple introduced a groundbreaking feature for Swift developers: **noncopyable types**. This enhancement expands Swift's value ownership system, allowing developers to express intent more clearly and write safer, more predictable code. Let’s explore what noncopyable types are, why they’re useful, and how you can leverage them in your projects.

## Copying in Swift: A Quick Overview

In Swift, values are **copyable by default**. This means that when you assign a value to a new variable or pass it to a function, Swift automatically creates a copy of that value. For example:

```swift
struct Player {
    var icon: String
}

func test() {
    let player1 = Player(icon: "🐸")
    var player2 = player1
    player2.icon = "🚚"
    assert(player1.icon == "🐸") // ✅ player1 is unaffected by changes to player2
}
```

Here, `player2` is a copy of `player1`, so modifying `player2` doesn’t affect `player1`.

In contrast, **reference types** behave differently:

```swift
class PlayerClass {
    var icon: String
    init(_ icon: String) { self.icon = icon }
}

func test() {
    let player1 = PlayerClass("🐸")
    let player2 = player1
    player2.icon = "🚚"
    assert(player1.icon == "🐸") // ❌ Assertion fails: both references point to the same object
}
```

Here, `player1` and `player2` point to the same object in memory, so changes to one affect the other.

## What Are Noncopyable Types?

While copyable types work well for most scenarios, there are situations where copying values can lead to bugs or unexpected behavior. Noncopyable types solve certain problems that reference types cannot solve easily or effectively. While both noncopyable types and reference types involve constraints on how values are used, noncopyable types provide value semantics with strict ownership rules, offering benefits that reference types alone cannot achieve. Let’s explore the key advantages of noncopyable types over reference types.

### Improved Program Correctness

With reference types, shared ownership can lead to **aliasing issues** and **unexpected mutations**:

```swift
class Task {
    var isComplete = false
    func run() {
        assert(!isComplete)
        isComplete = true
    }
}

func example() {
    let task1 = Task()
    let task2 = task1 // task2 is a reference to the same object
    task2.run()
    task1.run() // ❌ Assertion fails because task1 and task2 are the same object
}
```

To prevent such issues, developers often resort to runtime checks, locks, or throwing errors—each with its own trade-offs. Noncopyable types eliminate these risks by enforcing ownership rules at compile time:

```swift
struct Task: ~Copyable {
    consuming func run() {
        // Perform task
    }
}

func example() {
    var task = Task()
    task.run() // ✅ Task cannot be reused after this
    task.run() // ❌ Compile-time error: task has already been consumed
}
```

This guarantees that a `Task` is used only once, ensuring correctness without runtime checks or manual assertions.

### Value Semantics with Ownership

Noncopyable types retain **value semantics**, meaning each instance is a distinct, independent value. Reference types, on the other hand, rely on **reference semantics**, where multiple variables can point to the same object in memory.  

Value semantics are particularly useful in concurrent or functional programming, where immutable data structures and unique ownership are key. Noncopyable types allow you to maintain value semantics while explicitly managing ownership and preventing copies.

Example: Modeling a **bank transfer**:  

- With a **reference type**, you risk accidentally sharing the same transfer object across different parts of the program. 
- With a **noncopyable type**, you can ensure that a transfer is **moved** (not shared) and cannot be reused after it’s processed.

```swift
struct BankTransfer: ~Copyable {
    consuming func process() {
        // Perform transfer
    }
}

func handleTransfer(transfer: consuming BankTransfer) {
    transfer.process() // Transfer is consumed here
}
```

Here, `BankTransfer` cannot be reused after being processed, ensuring each transfer is handled exactly once.

### Safer Resource Management

Noncopyable types make **resource management** safer and more predictable. Reference types rely on **reference counting** for memory management, but deinitialization only occurs when all references to an object are destroyed. This can lead to **dangling references** or delays in cleanup if references are unintentionally retained.

Noncopyable types, however, guarantee that a value is **consumed and destroyed** when it goes out of scope or is explicitly discarded. This ensures timely cleanup and prevents multiple references from delaying deinitialization.

Example: Closing a file or canceling a task:

```swift
struct FileHandle: ~Copyable {
    deinit {
        close() // Ensure the file is closed when the handle is destroyed
    }

    consuming func read() -> Data {
        // Read data from the file
    }
}

func processFile(handle: consuming FileHandle) -> Data {
    return handle.read() // Handle is consumed here
}
```

Here, the `FileHandle` is destroyed immediately after use, ensuring no dangling references remain.  

*You can manually call `discard self` within a consuming function. However, note that doing so comes with a caveat: the `deinit` method will **not** be invoked in this case.*

### Borrowing and Temporary Access

Noncopyable types offer **fine-grained control** over how values are passed to functions:

- **Borrowing**: Read-only access (like a `let` binding). 
- **Consuming**: Full ownership transfer, preventing further use by the caller. 
- **Inout**: Temporary write access, requiring reinitialization before returning.

```swift
struct FloppyDisk: ~Copyable {}

func loadDisk(disk: borrowing FloppyDisk) {
    // Read-only access
}

func consumeDisk(disk: consuming FloppyDisk) {
    // Use and destroy the disk
}

func formatDisk(disk: inout FloppyDisk) {
    var tempDisk = disk
    // Modify tempDisk
    disk = tempDisk // Reinitialize before returning
}
```

## Noncopyable Generics

Noncopyable types also extend to **generics**. Previously, all generic types in Swift were implicitly constraint to `Copyable`.

```swift
func execute<T: Runnable>(_ t: T) {
    t.run()
}
```

By default, `T` is constrained to `Copyable`. If you want to allow noncopyable types, you can explicitly relax this constraint using `~Copyable`:

```swift
protocol Runnable: ~Copyable {
    consuming func run()
}

func execute<T: Runnable & ~Copyable>(_ t: consuming T) {
    t.run()
}
```

Simply removing the `Copyable` constraint from `Runnable` is not enough. The generic parameter `T` still has an implicit `Copyable` constraint, meaning `T` is constrained to both `Runnable` and `Copyable`. To fully support noncopyable types, you must explicitly remove the `Copyable` constraint from `T`.  

With this change, the `execute` function can now work with both copyable and noncopyable types.   

*Think of `~Copyable` as meaning: “It might be `Copyable`, but it also might not.”*

### Conditionally Copyable Types

Sometimes, you might want a type to be copyable only if its generic parameter is copyable. For example:

```swift
struct Job<Action: Runnable & ~Copyable>: ~Copyable {
    var action: Action?
}

// Allow Job to be copyable if Action is copyable
extension Job: Copyable where Action: Copyable {}
```

In this case: 

- A `Job` containing a noncopyable type is noncopyable. 
- A `Job` containing a copyable type is copyable.

### Extensions and Noncopyable Types

When extending types with noncopyable generic parameters, extensions default to requiring `Copyable` constraints. For example:

```swift
extension Job {
    consuming func getAction() -> Action? {
        return action
    }
}
```

This extension works only if `Action` is copyable. To support noncopyable types, remove the `Copyable` constraint:

```swift
extension Job where Action: ~Copyable {
    consuming func getAction() -> Action? {
        return action
    }
}
```

## Summary: What Noncopyable Types Solve That Reference Types Cannot

| Feature | Reference Types | Noncopyable Types | 
|--------------------------------|------------------------------|---------------------------| 
| **Prevent accidental reuse** | Requires manual checks | Enforced at compile time | 
| **Value semantics** | Not supported | Fully supported | 
| **Timely resource cleanup** | Depends on reference counting| Guaranteed via ownership | 
| **Explicit ownership** | Not possible | Borrowing, consuming, inout | 
| **Generic constraints** | Limited | Supports noncopyable generics | 

Noncopyable types introduce a new dimension of **ownership and correctness** to Swift. They solve problems that reference types cannot handle effectively, especially in scenarios involving **value semantics**, **resource management**, and **ownership enforcement**. By leveraging noncopyable types, you can write safer, more efficient, and more predictable code.
