---
title: '@Observable vs ObservableObject: Understanding the Differences'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI, Observation]
---

With the introduction of Swift's new concurrency features and updates to the SwiftUI ecosystem, developers are continually provided with tools to write more declarative and efficient code. One such advancement is the `@Observable` macro in Swift, which addresses some limitations of `ObservableObject` while enhancing the overall developer experience.  

In this post, we’ll explore what `@Observable` is, how it compares to `ObservableObject`, and when you should choose one over the other.

## Syntax Comparison 

```swift
final class MyObservableObject: ObservableObject {

    @Published var property1: Int = 0

    var property2: Double = 0
    let property3: Bool = false
}

@Observable final class MyObservable {

    var property1: Int = 0

    @ObservationIgnored var property2: Double = 0
    let property3: Bool = false
}
```

With `ObservableObject`, only properties annotated with `@Published` trigger SwiftUI view updates.  

In contrast, with `@Observable`, all mutable stored properties are automatically annotated with `@ObservationTracked`. You can opt out of tracking specific properties using `@ObservationIgnored`.

## Usage in SwiftUI

```swift
@StateObject private var observableObject: MyObservableObject = .init()
@State private var observable: MyObservable = .init()

@ObservedObject var observedObject: MyObservableObject
let observable: MyObservable
```

### Ownership Rules  

**When the view owns the object:**  
- Use `@StateObject` for `ObservableObject`.  
- Use `@State` for `@Observable`. It is crucial to use `@State`; otherwise, it's just a regular variable defined in the view, the state variables stored in the `@Observable` will be reset when the view is re-initialized. For a deeper understanding, check out [Mastering SwiftUI State](../mastering-swiftui-state/).

**When the object is passed from outside:**  
- Use `@ObservedObject` for `ObservableObject`.  
- For `@Observable`, no special property wrapper is required. However, if you need bindings to its properties, use `@Bindable`.  

### Environment Injection  

You can inject these objects into the SwiftUI environment and retrieve them as follows:  
- **`ObservableObject`:**  
```swift
view.environmentObject(myObservableObject)
@EnvironmentObject private var myObservableObject: MyObservableObject
```

- **`@Observable`:**  
```swift
view.environment(myObservable)
@Environment(MyObservable.self) private var myObservable
```

## Performance Considerations

```swift
final class MyObservableObject: ObservableObject {

    @Published var property1: Int = 0
    @Published var property2: Double = 0
}

@Observable final class MyObservable {

    var property1: Int = 0
    var property2: Double = 0
}
```

`@Observable` is significantly more performant compared to `ObservableObject`. 

- With `ObservableObject`, the view is re-evaluated whenever the value of any `@Published` property changes—even if the view does not use that property.  
- With `@Observable`, views are updated only when the properties they explicitly depend on are modified.  This targeted update behavior can lead to noticeable performance improvements in complex UIs.

## Nested Observability

Nesting one `ObservableObject` within another introduces challenges. Changes in the nested object are not automatically propagated, requiring manual observation and re-publishing:

```swift
import Combine

final class MyObservableObject: ObservableObject {

    @Published var property: Int

    let other: MyOtherObservableObject

    var otherObservableObjectObserver: AnyCancellable?

    init() {
        self.property = 0
        self.other = .init()

        self.otherObservableObjectObserver = self.other.objectWillChange.sink { [weak self] in
            self?.objectWillChange.send()
        }
    }
}

final class MyOtherObservableObject: ObservableObject {

    @Published var property1: Int = 0
    @Published var property2: Double = 0
}
```

With `@Observable`, nesting objects is seamless. You don’t need to manually handle observation; updates propagate automatically:

```swift
@Observable final class MyObservable {

    var property: Int = 0
    let other: MyOtherObservable = .init()
}

@Observable final class MyOtherObservable {

    var property1: Int = 0
    var property2: Double = 0
}
```

## Conclusion  

While `ObservableObject` has served SwiftUI developers well, `@Observable` introduces significant improvements:  

1. **Automatic Property Tracking:** No need for `@Published`—mutable properties are tracked by default.  
2. **Better Performance:** Updates are scoped to only the properties a view depends on.  
3. **Seamless Nesting:** No manual observation setup is required for nested objects.  

If you’re targeting the latest Swift version, now is a great time to explore `@Observable` and take your SwiftUI development to the next level. However, `ObservableObject` remains a viable option, especially for projects that need compatibility with older Swift/OS versions.
