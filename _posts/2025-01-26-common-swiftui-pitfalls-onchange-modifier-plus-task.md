---
title: 'Common SwiftUI Pitfalls - onChange modifier + Task'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI, Concurrency]
---

When developing in SwiftUI, it is common to encounter scenarios where asynchronous tasks need to be executed upon specific state changes. It may be intuitive to implement the following:

```swift
struct MyView: View {

    @State private var myState: MyState = ...

    var body: some View {
        MyAwesomeView()
            .onChange(of: myState) {
                Task {
                    await doWork()
                }
            }
    }
}
```

However, this approach generates a new unstructured task each time `myState` changes, without a mechanism to cancel it. While you could manage task cancellation by storing the task in a variable and cancelling it in the subsequent `onChange` closure, a more streamlined solution exists:

```swift
struct MyView: View {

    @State private var myState: MyState = ...

    var body: some View {
        MyAwesomeView()
            .task(id: myState) {
                await doWork()
            }
    }
}
```

This method effectively cancels and recreates the task upon `myState` changes, and ensuring task cancellation when the view disappears.

*Please note that the task will also trigger when the view appears.*

