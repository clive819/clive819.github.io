---
title: 'Mastering SwiftUI State'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

SwiftUI is built on three core principles: **Identity**, **Lifetime**, and **Dependencies**. These principles work together to help SwiftUI manage views efficiently:  

- **Identity**: How SwiftUI recognizes elements as the same or distinct across multiple updates of your app.  
- **Lifetime**: How SwiftUI tracks the existence of views and their associated data over time.  
- **Dependencies**: How SwiftUI knows when and why your UI needs to update.  

## Dependencies

All view properties in SwiftUI are dependencies of a view. A **dependency** is simply an input to the view, and when a dependency changes, the view must produce a new body.  For example:  

```swift 
struct MyView: View {

    @Binding var isOn: Bool // This is a dependency
    var name: String // This is also a dependency
    
    var body: some View {
        // View content here
    }
}
```  

### Key points about dependencies:  

- A dependency is any property that influences a view’s content.  
- When a dependency changes, SwiftUI automatically invalidates the view and recalculates its body.   

### Dependency Graph  

SwiftUI maintains a **dependency graph** to track relationships between views and their dependencies. Multiple views can depend on the same dependency source. When a dependency changes, SwiftUI efficiently invalidates and updates all views impacted by that change.

## Identity

**Identity** determines whether views represent the same conceptual UI element in different states or are entirely distinct. This is critical for understanding how SwiftUI transitions between states:  

- **Same identity**: SwiftUI applies new state changes to the same view, enabling fluid transitions (e.g., moving a view from one location to another).  
- **Different identity**: SwiftUI treats the views as distinct, transitioning them independently (e.g., fading one out and fading another in).   Properly managing view identity helps SwiftUI decide how to animate transitions and preserve state.  

### Types of Identity in SwiftUI:  

- **Explicit Identity**: Defined using custom or data-driven identifiers.
- **Structural Identity**: Inferred from the type and position of views in the hierarchy.  

#### Explicit Identity  

You can explicitly define identity using tools like `id` in `ForEach` or the `id(_:)` modifier:  

```swift 
ForEach(data, id: \.someProperty) { ... }  

MyView().id(someHashable) 
```  

Explicit identity is only necessary when SwiftUI needs to refer to a specific view elsewhere in your code (e.g., with `ScrollViewReader`). 

#### Structural Identity  

SwiftUI automatically assigns identity to views based on their position in the hierarchy and their type. For example:  

```swift 
var body: some View {
    if condition {
        viewA
    } else {
        viewB
    }
}
```  

- **`viewA`** is the view displayed when `condition` is true.  
- **`viewB`** is the view displayed when `condition` is false.   

These views have distinct identities because SwiftUI uses the view hierarchy structure to differentiate them. Each branch of the `if-else` results in different concrete types (e.g., `_ConditionalContent<ViewAType, ViewBType>`), which SwiftUI uses to determine identity.  If you want views to share the same identity, you can achieve this by applying the condition indirectly, for example, via a modifier:  

```swift  
var body: some View {
    MyView()
        .background(condition ? Color.green : Color.red)
}
```  

### Best Practices  

- Preserve view identity when possible to enable fluid transitions and retain state.  
- Avoid using `AnyView` unless absolutely necessary, as it makes it harder for SwiftUI to optimize and track view identity. Use `@ViewBuilder` to compose views dynamically instead.

## Lifetime

The **lifetime** of a view is tied to its identity. When a view’s identity changes, its lifetime ends, and SwiftUI creates a new instance of the view. During a view’s lifetime, it can change state, but its identity remains the same.  For example:  

```swift 
var body: some View {
    MyView(value: 1)
}
```  

If we later change the value:  

```swift 
var body: some View {
    MyView(value: 2)
}
```  

From SwiftUI’s perspective, these represent the **same view** because the identity (`MyView`) hasn’t changed. SwiftUI keeps a copy of the old view value to compare with the new one before updating the UI. After that, the old value is destroyed.

### View Value vs. View Identity  

- **View values** are ephemeral and should not be relied upon for persistence.  
- **View identity** is what you control to determine a view’s lifetime.   

The lifetime of a view’s state (e.g., `@State`, `@StateObject`) is tied to the lifetime of the view. When a view is created for the first time, SwiftUI allocates storage for state objects, which persists as long as the view is alive. 

## State

State in SwiftUI is managed using property wrappers like `@State` and `@StateObject`. You can declare a state like this:  

```swift 
@State private var count: Int = 0 
```  

When the value of a state property changes, SwiftUI updates all parts of the UI that depend on it.  

### Initializing State from External Data  

If the initial value of your state depends on an external input, you can initialize it like this:  

```swift 
struct MyView: View {

    @State private var count: Int

    init(count: Int) {
        self._count = .init(wrappedValue: count)
    }

    var body: some View {
        Text(count, format: .number)
    }
}
```  

### Key Consideration  

SwiftUI initializes a state object only the **first time** it creates the view. If the input to `MyView` changes, SwiftUI reinitializes the view, but the state’s initial value won’t change because it was only set during the first initialization.

### Changing State When Input Changes  

If you want the state to reset when the input changes, you can change the view’s identity:  

```swift 
MyView(count: value)
    .id(value)
```  

This forces SwiftUI to treat the view as a new instance, resetting its state. However, this comes with trade-offs:  

1. **Performance cost**: Reinitializing state can be expensive.  
2. **Side effects**: Changing view identity resets all state, including `@State`, `@GestureState`, `@FocusState`, etc. 
3. Additionally, animations within the view may not work as expected if the identity changes.  

## Side Note 

The `.onAppear` or `.task` modifiers are called whenever a view is presented. If you change a view’s identity, SwiftUI creates a new instance of the view, which also triggers these modifiers again. This behavior is intentional and not a bug—it reflects how SwiftUI handles view identity.
