---
title: 'SwiftUI View Preferences'
author: 'Clive Liu'
layout: post
tags: [Swift, SwiftUI]
---

SwiftUI allow us to pass data from parent to child views via `environment`. But what if you need to send data from child views back to their parent? This is where **SwiftUI preference** comes in. `preference` allow child views to communicate their state or data up the view hierarchy to their parent views.

To create a custom preference, simply define a struct that conforms to the `PreferenceKey` protocol. It’s also a good practice to create a convenience extension method for your view. This keeps the preference key type hidden and simplifies usage.

```swift
extension View {

    func dismissable(_ dismissable: Bool) -> some View {
        self.preference(key: MyPreferenceKey.self, value: dismissable)
    }
}

fileprivate struct MyPreferenceKey: PreferenceKey {

    static let defaultValue: Bool = true

    static func reduce(value: inout Bool, nextValue: () -> Bool) {
        value = nextValue()
    }
}
```

## Usage Example

```swift
struct MyView: View {

    @State private var isDismissable: Bool = true

    var body: some View {
        VStack {
            if isDismissable {
                Button("dismiss") {
                    // ...
                }
            }

            Text("Child View 1")
                .dismissable(false)

            Text("Child View 2")
        }
        .onPreferenceChange(MyPreferenceKey.self) {
            isDismissable = $0
        }
    }
}
```

The `onPreferenceChange` modifier allows you to observe and update a parent view based on the preference value set by child views.

## Handling Conflicts

Let’s consider a scenario where multiple child views set conflicting preference values. For example:

```swift
struct MyView: View {

    @State private var isDismissable: Bool = true

    var body: some View {
        VStack {
            if isDismissable {
                Button("dismiss") {
                    // ...
                }
            }

            Text("Child View 1")
                .dismissable(false)

            Text("Child View 2")
                .dismissable(true)
        }
        .onPreferenceChange(MyPreferenceKey.self) {
            isDismissable = $0
        }
    }
}
```

If any child view sets `dismissable` to `false`, the parent should honor that. In this case, the dismiss button will incorrectly appear because the last child view (`Child View 2`) overrides the preference set by `Child View 1`. This happens because the `reduce` function simply replaces the current value with the next one. To handle conflicts properly, we need to update the `reduce` function.

```swift
fileprivate struct MyPreferenceKey: PreferenceKey {

    static let defaultValue: Bool = true

    static func reduce(value: inout Bool, nextValue: () -> Bool) {
        value = value && nextValue()
    }
}
```

With this implementation, the `reduce` function uses a logical `AND` operation to combine values. This means if **any child view sets `dismissable` to `false`**, the final value will also be `false`. This ensures the dismiss button only appears if **all child views agree** that the view is dismissable.
