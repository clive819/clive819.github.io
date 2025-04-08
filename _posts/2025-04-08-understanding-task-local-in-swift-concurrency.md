---
title: 'Understanding TaskLocal in Swift Concurrency'
author: 'Clive Liu'
layout: post
tags: [Swift, Concurrency]
---

`TaskLocal` is a mechanism within Swift’s concurrency system that allows you to store and access values that are local to a task and its child tasks. It’s similar to thread-local storage from the pre-concurrency days, but tailored for Swift’s structured concurrency model.

Imagine you’re writing code and want to pass contextual information—like a tracing context—throughout the call stack without explicitly passing it into every function. That’s exactly the kind of scenario `TaskLocal` is designed to simplify. Think of it like SwiftUI’s environment values: data flows from parent to child, and any descendant can access it without needing every function in the call chain to propagate it manually.

## Usage

Here’s how to define and use a task-local value:

```swift
enum MetricsReporter {
    // Must be declared as static properties (below Swift 6.0)
    @TaskLocal static var workflowID: UUID?
}

// Global task-local properties are supported starting with Swift 6.0
@TaskLocal var workflowID: UUID?
```

To set a task-local value, you must use the `withValue` function:

```swift
await $workflowID.withValue(.init()) {
    await myAmazingWorkflow()

    Task {
        await myOtherAmazingWorkflow()
    }

    Task.detached {
        // Detached tasks do NOT inherit TaskLocal values.
    }
}
```

Within `myAmazingWorkflow()`, `myOtherAmazingWorkflow()`, and any functions they call, `workflowID` will return the value assigned via `withValue`. However, **detached tasks** do **not** inherit task-local values—they start with a clean slate.

## Example

Here’s a full example to demonstrate `TaskLocal` in action:

```swift
@TaskLocal var workflowID: UUID?

func log(_ message: String) {
    print("\(workflowID) \(message)")
}

func step1() async {
    // do work
    log("step1 completed")
}

func step2() async {
    // do work
    log("step2 completed")
}

func step3() async {
    // do work
    log("step3 completed")
}

func myAmazingWorkflow() async {
    await step1()
    await step2()
    await step3()
}

Task {
    await withDiscardingTaskGroup { group in
        for _ in 0..<3 {
            group.addTask {
                await $workflowID.withValue(.init()) {
                    await myAmazingWorkflow()
                }
            }
        }
    }
}
```

**Sample Output:**

```
Optional(36A1DF29-CF82-4706-A268-B5056A3D83CA) step1 completed
Optional(36A1DF29-CF82-4706-A268-B5056A3D83CA) step2 completed
Optional(177104F2-15A8-437C-8114-69C5D88E7C09) step1 completed
Optional(36A1DF29-CF82-4706-A268-B5056A3D83CA) step3 completed
Optional(177104F2-15A8-437C-8114-69C5D88E7C09) step2 completed
Optional(CA2DA6F3-DAA8-4F2A-AEBE-B5A6357723BF) step1 completed
Optional(177104F2-15A8-437C-8114-69C5D88E7C09) step3 completed
Optional(CA2DA6F3-DAA8-4F2A-AEBE-B5A6357723BF) step2 completed
Optional(CA2DA6F3-DAA8-4F2A-AEBE-B5A6357723BF) step3 completed
```

Each task gets its own `workflowID`, making it easy to correlate logs and trace execution flow—especially useful in debugging and monitoring.

## Conclusion

`TaskLocal` is a powerful tool in Swift’s concurrency toolbox. When used wisely, it can simplify your code by removing the need to pass contextual parameters everywhere.

But like any powerful abstraction, it should be used with care. Make sure you're not hiding too much logic behind implicit context, and use it where it truly improves readability and maintainability.
