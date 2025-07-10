---
title: 'Access MainActor Synchronously Safely'
author: 'Clive Liu'
layout: post
tags: [Swift, Concurrency]
---

> **Use with caution:** Avoid accessing `MainActor` synchronously unless absolutely necessary.
{: .prompt-warning }

In rare cases—particularly when integrating with legacy code—you may need to synchronously access `MainActor`-isolated code. While this approach should be avoided when possible, Swift does provide a safe mechanism for doing so when absolutely necessary.

Here's how you can accomplish that:

```swift
extension MainActor {

    /// Executes a closure synchronously on the `MainActor`.
    ///
    /// This method allows synchronous access to `MainActor`-isolated code.
    ///
    /// - Parameter operation: A closure that runs on the `MainActor`.
    /// - Returns: The value returned by `operation`.
    /// - Throws: Rethrows any error thrown by `operation`.
    ///
    /// - Warning: Synchronous access to actors should be avoided when possible.
    ///   This is intended for exceptional cases, such as bridging with legacy APIs.
    public static func sync<R: Sendable>(operation: @MainActor () throws -> R) rethrows -> R {
        if Thread.isMainThread {
            // This is valid as per SE-0424:
            // https://github.com/swiftlang/swift-evolution/blob/main/proposals/0424-custom-isolation-checking-for-serialexecutor.md
            //
            // Being able to assert isolation for non-task code this way is important enough
            // that the Swift runtime actually already has a special case for it:
            // even if the current thread is not running a task, isolation checking will succeed
            // if the target actor is the MainActor and the current thread is the main thread.
            try MainActor.assumeIsolated(operation)
        } else {
            // As per https://developer.apple.com/documentation/swift/mainactor
            // MainActor is a singleton actor whose executor is equivalent to the main dispatch queue.
            // The compiler would also yell at you if you use non-main queue.
            // Therefore, this code is valid.
            //
            // - Note: You cannot do `DispatchQueue.main.sync` if you're already on main thread,
            // because that will cause a deadlock. Hence, the `isMainThread` check.
            try DispatchQueue.main.sync(execute: operation)
        }
    }
}
```

