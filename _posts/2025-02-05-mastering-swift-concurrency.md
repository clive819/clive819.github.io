---
title: 'Mastering Swift Concurrency'
author: 'Clive Liu'
layout: post
tags: [Swift, Concurrency]
---

## Intro

Swift Concurrency is a powerful framework introduced in Swift 5.5 that simplifies writing asynchronous and concurrent code, making it safer, more readable, and easier to maintain. Built around modern concepts like `async`/`await` and structured concurrency, Swift Concurrency allows developers to express asynchronous operations in a natural, linear style while ensuring thread safety and eliminating common pitfalls like callback hell or race conditions. By leveraging key features such as `Task`, `Actor`, and the cooperative scheduling model, Swift Concurrency provides developers with a robust and scalable approach to building responsive and efficient applications across Apple platforms.

## Sendable

The `Sendable` protocol is a cornerstone of Swift's concurrency model, ensuring thread safety when working with data across concurrent tasks. It is a marker protocol that indicates a type can be safely transferred, or "sent," between concurrent contexts, such as different tasks or threads. This helps mitigate data races and ensures that shared data is handled correctly in concurrent programs. `Sendable` itself has no requirements (e.g., no methods or properties to implement). Many types in Swift conform to Sendable automatically, including value types like `Int` and `String`, as well as immutable reference types. 

However, classes do not conform to `Sendable` by default because they often introduce shared mutable state. If a class is thread-safe (e.g., by having no mutable state or by synchronizing access internally), you can manually conform it to `Sendable`. For cases where you cannot statically prove thread safety to the compiler but are confident in your implementation, you can mark the type as `@unchecked Sendable`. This bypasses compiler checks, but you take full responsibility for ensuring thread safety in these cases.

## Task

A `Task` represents a unit of asynchronous work. Tasks can be canceled, and cancellation automatically propagates to child tasks. However, task cancellation in Swift is cooperative, meaning that the running code must check for cancellation and decide whether to stop execution.

### Structured concurrency

Tasks can be created as either structured or unstructured. Structured concurrency ensures that tasks are tied to a specific scope, such as a function or task group, and are automatically canceled when the scope exits. This makes it easier to manage and reason about concurrent code.

#### Example: structured task with async let

`async let` allows you to create child tasks that execute concurrently. These tasks are bound to the scope of the parent and are automatically cancelled before the scope exits.

```swift
func fetchData() async throws -> (Data, Data) {
    async let firstData = fetchFromFirstSource()
    async let secondData = fetchFromSecondSource()
    
    return try await (firstData, secondData)
}
```

#### Example: TaskGroup

Use task groups to manage a dynamic number of tasks, ensuring their execution is structured.

```swift
func processItems() async {
    await withDiscardingTaskGroup { group in
        for item in items {
            group.addTask {
                await process(item)
            }
        }
    }
}
```

### Unstructured concurrency

Unstructured tasks are created outside the scope of structured concurrency. While they offer more flexibility, they require more manual management of cancellation and error propagation.

```swift
Task {
    await performAsyncWork()
}
```

Unstructured tasks are not tied to a parent task, so errors and cancellations must be handled explicitly. Avoid using unstructured tasks unless absolutely necessary.

### Best practices

- Prefer structured concurrency over unstructured concurrency.
- Use the `.task(id:)` modifier in SwiftUI for lifecycle-aware task creation.
- Use `async let` for fixed numbers of concurrent operations and task group for dynamic workloads.

## Actor

An `actor` is a concurrency-safe reference type specifically designed to protect mutable state by ensuring that only one task can access an actor's mutable state at a time. Code that interacts with an actor's properties or methods must use `await`, ensuring that access is asynchronous and respects the actor's internal synchronization.

When designing your own actor, keep its functions as simple as possible and try to minimize the time spent inside the actor. Perform non-actor-specific work outside of the actor to avoid unnecessary serialization.

```swift
actor VideoEncoder {

    private var logs: [String] = []
    
    private func log(message: String) {
        logs.append(message)
    }
    
    nonisolated func encode(fileURL: URL) async {
        await log(message: "Start encoding video at \(fileURL)...")
        // Perform long-running operation
        await log(message: "Finished encoding video at \(fileURL)")
    }
}
```

### Actor reentrancy

Actor reentrancy is an important concept in Swift's concurrency model. It refers to the fact that actors are reentrant by default, meaning that they can process other tasks while waiting for an asynchronous operation to complete. This behavior allows actors to remain responsive and avoid deadlocks, but it also introduces potential challenges, such as interleaved execution and unexpected behavior when accessing an actor's isolated state. Developers need to be mindful of these challenges when designing actor-based systems.

```swift
actor BankAccount {
    private var balance: Int = 0

    func deposit(amount: Int) async {
        balance += amount
        print("Deposited \(amount), balance is now \(balance)")
        await Task.sleep(1_000_000_000) // Simulate a delay
        print("Finished deposit, balance is \(balance)")
    }

    func withdraw(amount: Int) async -> Bool {
        if balance >= amount {
            balance -= amount
            print("Withdrew \(amount), balance is now \(balance)")
            return true
        } else {
            print("Insufficient funds.")
            return false
        }
    }
}

let account = BankAccount()

Task {
    await account.deposit(amount: 100)
}

Task {
    let success = await account.withdraw(amount: 50)
    print("Withdrawal success: \(success)")
}
```

- The `deposit(amount:)` method increments the balance to 100 and then suspends at the `await Task.sleep` line.
- While the first task is suspended, the second task (`withdraw(amount:)`) executes and withdraws 50, reducing the balance to 50.
- When the first task resumes, it prints the final balance as 50, even though it expected the balance to remain at 100.

### Best practices

- Avoid mixing long-running operations with actor-isolated state.
- Move non-actor-specific work outside the actor by marking functions as `nonisolated` where appropriate.
- Keep actors simple and focused on protecting shared mutable state.

## Recommended reads

- [Explore Structured Concurrency in Swift - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10134/)
- [Visualize and Optimize Swift Concurrency - WWDC22](https://developer.apple.com/videos/play/wwdc2022/110350/)
