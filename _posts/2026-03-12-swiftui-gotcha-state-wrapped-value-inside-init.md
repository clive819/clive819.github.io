---
title: 'SwiftUI Gotcha - @State wrappedValue Inside init()'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

You've probably heard the rule: SwiftUI only uses `@State`'s initial value once. After that, it ignores whatever you
pass in subsequent `init()` calls. But what exactly happens to `_state.wrappedValue` *inside* `init()` when the view is
re-initialized? Does it return the live SwiftUI-managed state, or the freshly-constructed default?

This distinction matters — especially when you're storing `@Observable` classes in `@State` and doing work in `init()`.

## The Setup

Consider a view that eagerly fetches data in `init()` to avoid a loading flash:

```swift
struct ContentView: View {
    @State private var loader = DataLoader()

    init() {
        let loader = _loader.wrappedValue
        Task { @MainActor in
            await loader.fetch()
        }
    }

    var body: some View {
        if loader.hasFetched {
            Text("Loaded \(loader.items.count) items")
        } else {
            ProgressView()
        }
    }
}
```

This works on first render — `DataLoader` is created, the `Task` fetches data, `hasFetched` flips to `true`, and the
view updates. But what happens when the parent re-renders and `ContentView.init()` runs again?

## What the Docs Say

From Apple's [`State` documentation](https://developer.apple.com/documentation/swiftui/state):

> A `State` property always instantiates its default value when SwiftUI instantiates the view.

And from [`State.init(wrappedValue:)`](https://developer.apple.com/documentation/swiftui/state/init(wrappedvalue:)/):

> SwiftUI initializes the state's storage only once for each container instance that you declare.

So a new default value is *constructed* every time, but SwiftUI *ignores* it after the first creation. The question
remains: when you read `_state.wrappedValue` inside `init()`, which one do you get?

## The Test

```swift
@Observable
final class Counter {
    var count = 0
    var id: String

    init() {
        self.id = String(UUID().uuidString.prefix(8))
        print("🔵 Counter.init — id: \(id), count: \(count)")
    }

    deinit {
        print("🔴 Counter.deinit — id: \(id), count: \(count)")
    }
}

struct ChildView: View {
    @State private var counter = Counter()
    let label: String

    init(label: String) {
        self.label = label
        let val = _counter.wrappedValue
        print("⚪️ ChildView.init — label: \(label), wrappedValue.id: \(val.id), wrappedValue.count: \(val.count)")
    }

    var body: some View {
        VStack {
            Text("\(label) — count: \(counter.count)")
            Text("id: \(counter.id)")
            Button("Increment") { counter.count += 1 }
        }
    }
}

struct ParentView: View {
    @State private var parentCount = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("Parent count: \(parentCount)")
            Button("Re-render parent") { parentCount += 1 }
            ChildView(label: "child-\(parentCount)")
        }
        .padding()
    }
}
```

Steps:

1. Launch the app
2. Tap "Increment" a few times
3. Tap "Re-render parent"
4. Check the console

## The Results

```
🔵 Counter.init — id: 2588864D, count: 0
⚪️ ChildView.init — label: child-0, wrappedValue.id: 2588864D, wrappedValue.count: 0
🔵 Counter.init — id: 2FD75261, count: 0
⚪️ ChildView.init — label: child-0, wrappedValue.id: 2FD75261, wrappedValue.count: 0
🔵 Counter.init — id: 0499F319, count: 0
⚪️ ChildView.init — label: child-1, wrappedValue.id: 0499F319, wrappedValue.count: 0
🔴 Counter.deinit — id: 2FD75261, count: 0
🔵 Counter.init — id: BC6AAD29, count: 0
⚪️ ChildView.init — label: child-2, wrappedValue.id: BC6AAD29, wrappedValue.count: 0
🔴 Counter.deinit — id: 0499F319, count: 0
```

Three observations:

1. **A new `Counter` is allocated on every `init()` call** — each has a unique id (`2588864D`, `2FD75261`, `0499F319`,
   `BC6AAD29`).
2. **`_counter.wrappedValue` returns the throwaway, not the live state** — the ids in `⚪️` always match the `🔵` line
   above it (the freshly-constructed default), never the original `2588864D`. And `count` is always `0`, even after
   incrementing.
3. **SwiftUI keeps the first instance and discards the rest** — `2588864D` is never deinit'd (SwiftUI kept it), while
   every subsequent `Counter` gets deallocated.

## Why This Matters

If you capture `_state.wrappedValue` in a `Task` from `init()`, you're operating on a throwaway object that SwiftUI will
discard. The work you do — network calls, database queries, subscriptions — runs against an object nobody is listening
to.

For value types stored in `@State`, this is mostly harmless — the throwaway is a cheap struct copy. But for
`@Observable` classes, each throwaway is a real heap allocation that may set up subscriptions, start timers, or trigger
I/O in its own `init`.

This also means guards like this won't work:

```swift
init() {
    if !_loader.wrappedValue.hasFetched {
        // This always runs — wrappedValue.hasFetched is always
        // false because it's reading from the new default, not
        // the live state where the fetch already completed.
    }
}
```

## The Fix

Don't do side-effectful work in `init()` based on `_state.wrappedValue`. Use `.task` or `.onAppear` instead — these are
tied to the view's lifecycle:

```swift
struct ContentView: View {
    @State private var loader = DataLoader()

    var body: some View {
        Group {
            if loader.hasFetched {
                Text("Loaded \(loader.items.count) items")
            } else {
                ProgressView()
            }
        }
        .task {
            await loader.fetch()
        }
    }
}
```

If you need the fetch to complete *before* the view appears (to avoid a loading flash during a swipe animation, for
example), consider having the parent pre-fetch the data and pass it in, or ensure the `@Observable` class's `init` is
lightweight and the fetch method is idempotent.
